---
name: Feature Task
about: PRD&SRS 기반의 구체적인 개발 태스크 명세
title: "[API/DATA] T2-003: NC Action & Report Generation Specification"
labels: 'api, smart-audit, spec, priority:high'
assignees: ''
---

## 🎯 Summary
- 기능명: [T2-003] NC Action & Report Generation
- 목적: 현장 품질 심사 중 부적합(NC) 판정 발생 시 AI가 즉각적으로 시정 조치 초안을 자동 생성하고, 원청 통보 및 조치 소요 시간을 최소화(5분 이내)하기 위한 전체 파이프라인 및 데이터 무결성 검증 체계를 구현한다.

## 🔗 References (Spec & Context)
- `DATA-DB_SCHEMA_v1.md`
- `T2-001_VISION_AI_SPEC.md`

## 🧭 Scope
### In Scope
- NC 시정 조치 6단계 프로세스 전이 관리
- `nc_reports`, `nc_action_items`, `nc_audit_trail` 등 관련 데이터베이스 테이블 모델링
- Gemini API (Vercel AI SDK `streamObject`) 연동 실시간 초안 생성 라우트 핸들러
- Upstash QStash 기반 비동기 백그라운드 자동 생성 워커
- 시정 조치 전/후 무결성 비교(SHA-256 해시) 로직
- NC 진행 추적 Kanban 보드 로직 및 Supabase Realtime 알림 연동

## 📄 Detailed Specification

### 1. NC 시정 조치 전체 프로세스 정의 (6단계)
원청의 부적합(NC) 통보 접수부터 최종 시정 완료 검증까지의 6단계 프로세스와 5분 이내 초안 작성 KPI 달성을 위한 파이프라인 명세입니다.

| 단계 | 상태 | 담당자 / 컴포넌트 | 시스템 액션 | 완료 기준 (DoD) |
| :--- | :--- | :--- | :--- | :--- |
| **1** | **접수 (RECEIVED)** | 품질 관리자 | 원청 NC 통보서(이메일/문서) 수신 및 시스템 등록 | `nc_reports` 레코드 생성 |
| **2** | **초안작성 (DRAFTING)** | 백엔드 API + Gemini | 접수 정보 기반 AI 시정 조치 초안 실시간 생성 (Streaming) | 5분 내 첫 토큰 반환 및 JSON 수신 |
| **3** | **검토중 (IN_REVIEW)** | 품질 관리자 | 생성된 초안 검토, 수정 및 내부 결재 | `nc_action_items` 확정 및 해시 락업 |
| **4** | **제출완료 (SUBMITTED)** | 관리자 → 원청 | 확정된 시정 조치 계획서를 원청 시스템에 전송 | 상태 `SUBMITTED` 전환 및 알림 |
| **5** | **검증중 (VERIFYING)** | 현장 작업자 + Vision AI| 실제 조치 내역 촬영 및 AI 검증 결과(T2-001 연동) 제출 | 검증 데이터 및 사진 첨부 완료 |
| **6** | **완료 (CLOSED)** | 원청 담당자 | 조치 결과 승인 및 최종 종결 통보 접수 | `CLOSED` 처리 및 스냅샷 해시 고정 |

### 2. NC 데이터 모델 정의
기존 `DATA-DB_SCHEMA_v1.md`와의 통합을 고려하여 설계된 테이블 구조입니다. 연쇄 삭제 방지를 위해 `ON DELETE RESTRICT`를 권장합니다.

1. **`nc_reports` (NC 보고서 헤더)**
   - `id` (UUID), `report_number` (String, 유니크), `client_code` (String), `notified_at` (Date), `defect_type` (String), `severity` (Enum: CRITICAL, MAJOR, MINOR), `status` (Enum: 위 6단계)
2. **`nc_action_items` (시정 조치 항목)**
   - `id` (UUID), `nc_report_id` (UUID, FK), `immediate_cause` (Text), `corrective_action` (Text), `preventive_action` (Text), `horizontal_deployment` (Text), `target_completion_date` (Date), `confidence_notes` (Text), `data_hash` (String)
3. **`nc_attachments` (첨부파일 참조)**
   - `id` (UUID), `reference_id` (UUID, FK), `reference_type` (Enum: REPORT, ACTION_VERIFICATION), `storage_path` (String, Supabase 버킷 경로), `uploaded_at` (Timestamptz)
4. **`nc_audit_trail` (모든 변경 이력 - Insert-only)**
   - `id` (UUID), `nc_report_id` (UUID, FK), `action_type` (String), `changed_by` (UUID), `snapshot_data` (JSONB), `created_at` (Timestamptz)

*(※ Insert-only 패턴: 시정 조치 초안 생성 후 "수정"하는 과정은 기존 필드를 무작정 덮어쓰지 않고 `nc_audit_trail`에 새로운 버전을 추가하여 컴플라이언스를 만족시킵니다.)*

### 3. AI 초안 생성 API 및 프롬프트 명세

#### 3-1. Route Handler (실시간 스트리밍)
- **Endpoint**: `POST /api/nc-reports/{report_id}/generate-draft`
- **Request Body (Zod 검증)**:
  ```typescript
  const DraftRequestSchema = z.object({
    nc_description: z.string().min(10),
    defect_type: z.string(),
    affected_process: z.string(),
    severity: z.enum(['CRITICAL', 'MAJOR', 'MINOR'])
  });
  ```
- **Streaming 응답 방식**: Vercel AI SDK의 `streamObject` 함수를 활용하여 5분 KPI 체감 대기시간을 없애고, 사용자가 실시간으로 JSON 덩어리가 채워지는 모습을 볼 수 있도록 청크를 반환합니다.

#### 3-2. Gemini 프롬프트 설계
**System Prompt**:
```text
System: 당신은 ISO 9001 기반 제조 현장의 시정 조치 전문가 AI입니다.
원청의 NC(부적합) 통보 내용과 이전 공정 데이터를 분석하여 즉각적이고 구체적인 조치 계획 초안을 5-Why 기법 논리로 작성하십시오.
반드시 제공된 핵심 섹션(즉각원인, 시정조치, 재발방지, 수평전개)을 명확히 구분해야 하며, 각 내용은 200자 이하로 서술하십시오.
출력은 반드시 지정된 JSON 구조만을 반환해야 합니다.
```

**User Prompt 템플릿**:
```typescript
const userPrompt = `
[NC 통보 내용 / 발생 정보]
- 원본 로그 ID: ${reportId}
- 공정/불량 유형: ${affected_process} / ${defect_type}
- 내용/현장설명: "${nc_description}"
- 심각도: ${severity}

[과거 유사 NC 이력 (RAG 컨텍스트)]
${similarPastReports.map(r => `- ${r.defect_type}: ${r.root_cause} -> ${r.preventive_action}`).join('\n')}

위 상황과 유사 이력을 종합하여 가장 적합한 시정 조치 계획을 도출하십시오.
판단이 불확실하거나 인간의 현장 검증이 필요한 부분은 confidence_notes에 메모를 남기십시오.
`;
```

#### 3-3. 출력 JSON 스키마 (Zod)
```typescript
import { z } from 'zod';

export const NcDraftSchema = z.object({
  immediate_cause: z.string().max(200).describe('5-Why 기법 기반 즉각 원인 분석'),
  corrective_action: z.string().max(200).describe('현장에서 취해야 할 시정 조치 내용'),
  preventive_action: z.string().max(200).describe('재발 방지를 위한 장기적 예방 대책'),
  horizontal_deployment: z.string().max(200).describe('수평 전개 계획 (유사 공정 확대 적용)'),
  target_completion_date: z.string().datetime().describe('예상 조치 완료일 (ISO 8601)'),
  required_resources: z.array(z.string()).describe('수행에 필요한 공구, 자재 등 리소스'),
  confidence_notes: z.string().describe('AI 불확실 항목 메모 및 현장 확인 요청 사항')
});
```

### 4. 5분 KPI 달성 보장 전략 및 아키텍처

1. **클라이언트 체감 대기시간 최소화 (Streaming)**:
   - API 호출 직후 AI 모델의 `streamObject` 이터레이터를 프론트엔드로 바로 연결. 서버의 데이터 수집 대기 없이 사용자는 1~2초 이내에 결과를 보기 시작함.
   - 초안 생성 시간 측정: `(첫 토큰 반환 타임스탬프 - 접수 상태 레코드 생성 타임스탬프) / 1000` (클라이언트 측정)
   - 300초 초과 시 지연 원인 로깅 및 KPI Violation 알림 발송.

2. **완전 비동기 백그라운드 큐 아키텍처 (Upstash QStash)** (선택적 Fallback/자동화 연계용):
   - 현장 작업자에 의한 `audit_log` Insert 트리거 발생 등, 즉시 UI 스트리밍이 필요 없는 백그라운드 자동 생성 케이스의 경우 Vercel의 짧은 응답 시간 제한(60초) 우회를 위해 사용.
   - QStash가 Webhook 엔드포인트 `/api/webhooks/nc-generator` 호출 시 최대 5분(Pro 플랜 설정) 활용하여 백그라운드 DB Insert 처리 및 재시도 정책 자동 적용.

### 5. NC 진행 추적 UI 명세 (T2-004 연계)
- **Kanban 보드 구성**: 화면 가로 영역에 6개 단계 컬럼을 배치 (DnD 기반 상태 이동).
- **카드 컴포넌트**: `NC 번호`, `원청명`, `심각도 배지`, `D-Day 카운터` 표시.
- **기한 임박 경고**: 기한이 3일 이내일 경우 카드 배경색을 경고 톤(`bg-red-50`)으로 변경.
- **상태 변경 알림**: 보드 조작 또는 API 전이 발생 시, `Supabase Realtime` 채널을 통해 접속 중인 담당자에게 즉각 토스트(Toast) 알림 송출.

### 6. 전/후 무결성 비교 기능 (T2-005 연계)
컴플라이언스 준수를 위해, 최종 승인된 조치 데이터가 이후 임의로 변조되지 않았음을 수학적으로 증명합니다.

```typescript
import { createHash } from 'crypto';

// 해시 생성 유틸 함수 (직렬화 시 공백/키 순서 보정)
export function generateDataHash(data: NcActionItem): string {
  const normalizedString = JSON.stringify({
    r_id: data.nc_report_id,
    ic: data.immediate_cause.trim(),
    ca: data.corrective_action.trim(),
    pa: data.preventive_action.trim(),
    hd: data.horizontal_deployment.trim()
  });
  return createHash('sha256').update(normalizedString).digest('hex');
}
```
- **병렬 비교 검증**: 좌측 원본(`nc_audit_trail`의 승인 시점 스냅샷), 우측 현재 DB 데이터. 생성된 해시값을 비교하여 무결성 즉시 입증 가능.

## ✅ Task Breakdown (실행 계획)
- [ ] 데이터베이스 스키마(NC 관련 4개 테이블) 생성 및 마이그레이션 (`ON DELETE RESTRICT` 적용)
- [ ] Zod 스키마(`NcDraftSchema`) 및 Gemini 프롬프트 템플릿 구현 (`lib/ai/nc-prompt.ts`)
- [ ] Vercel AI SDK `streamObject`를 활용한 실시간 초안 생성 API 작성 (`/api/nc-reports/...`)
- [ ] Upstash QStash 기반 비동기 자동 생성 Webhook 작성 (Background Queue용)
- [ ] 데이터 저장 시 SHA-256 해시를 적용한 스냅샷 기록(Audit Trail) 로직 적용
- [ ] Kanban 보드 상태 업데이트 로직 및 Supabase Realtime 알림 컴포넌트 구현

## 🧪 Acceptance Criteria (BDD / GWT)
- **Scenario 1**: 초안 스트리밍 생성
  - Given: 원청으로부터 NC 통보가 시스템에 접수되었다.
  - When: 초안 생성 API가 호출된다.
  - Then: 5분 이내에 첫 토큰이 반환되며, 즉각원인/시정조치/재발방지 대책이 포함된 JSON 구조가 클라이언트에 스트리밍된다.
- **Scenario 2**: 데이터 무결성(해시 락업) 
  - Given: 품질 관리자가 초안을 확정(Approved/In_Review) 처리하였다.
  - When: 상태 업데이트 로직이 실행된다.
  - Then: 현재 데이터 내용 기반의 SHA-256 해시값이 계산되어 `nc_audit_trail`에 영구 스냅샷으로 기록된다.

## 🏁 Definition of Done (DoD)
- [ ] 모든 6단계 프로세스 상태 전이가 정상 동작하는가?
- [ ] 초안 생성 시간이 Vercel 제한시간 내에 처리되거나 비동기로 안전하게 반환되는가?
- [ ] 해시 무결성 스냅샷 로직 및 Insert-only 원칙이 지켜지고 있는가?
- [ ] Zod 스키마에 의한 AI 출력 검증이 100% 통과하는가?
