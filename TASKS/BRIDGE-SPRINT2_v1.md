# BRIDGE-SPRINT2_v1.md

## 1. Sprint 1 → Sprint 2 인수인계 기준

### A. Sprint 1 완료 정의(DoD) 체크리스트 (기능 프리즈 조건)
- [ ] **인증 및 권한**: Admin/User 2단계 RBAC 분리가 구현되고, 비인가 라우트 접근 차단율이 100%인가? *(T1-003 범위. MFA 강제 적용은 Sprint 4 T4-003 범위)*
- [ ] **데이터 무결성**: `audit_log` 테이블에 어떠한 경우에도 `UPDATE` 및 `DELETE` 쿼리가 발생하지 않도록 DB 트리거가 적용되었는가?
- [ ] **핵심 AI 1단계**: Gemini 기반 STT 및 Smart Audit 매핑이 작동하고, Vercel Streaming에서 60초 타임아웃 및 429 에러 방어(지수 백오프)가 구현되었는가?
- [ ] **컴플라이언스**: PIPA 동의 이력 로그 DB 스키마(T1-014)가 Insert-only로 적용되었고, `user_consents` 테이블의 무결성이 검증되었는가? *(동의 UI 팝업 강제 노출은 Sprint 4 T4-002 범위)*
- [ ] **데모 시나리오**: `TEST-S1_DEMO_v1.md`에 명시된 5개 E2E 테스트를 Seed Data 기반으로 버그 없이 완주할 수 있는가?

### B. Sprint 2 착수 전 반드시 확정되어야 할 기술 결정 사항 목록
- **Vision AI 데이터 파이프라인 정책**: 이미지 업로드 시 Edge 단말 직접 업로드인지, 외부 클라우드 브릿지 서버를 통하는지 여부.
- **NC 시정 프로세스 승인 권한**: 시정 조치 초안 생성 후, 최종 반영 전 '관리자 승인(Approval Workflow)' 단계를 넣을 것인지 여부.
- **Edge 하드웨어 스펙 최종 확정**: 카메라 해상도 및 영상 프레임 레이트에 따른 모바일 브라우저 한계(메모리 부족 문제) 검증 결과.

---

## 2. NC 시정 조치 파이프라인 선행 설계 (T2-003)

### A. NC 사유 파싱 Gemini 프롬프트 구조 (초안 수준)
```text
System: 당신은 스마트 제조 품질 관리 AI 전문가입니다.
사용자가 제공하는 불량 내역(NC) 로그와 현장 작업자의 추가 설명(텍스트/음성 변환본)을 분석하여 구체적인 시정 조치안(Corrective Action)을 도출하세요.
기존의 ISO 품질 매뉴얼을 참고하여 실행 가능하고 안전한 조치 지침을 작성해야 합니다.

출력은 반드시 다음 JSON 스키마를 따르십시오.
```

### B. 시정 조치 초안 출력 JSON 스키마
```json
{
  "nc_id": "문제가 발생한 audit log ID",
  "root_cause_analysis": "근본 원인 분석 요약 (100자 이내)",
  "recommended_action": "즉각적인 시정 조치 내용",
  "preventive_action": "장기적 예방 조치 내용",
  "required_resources": ["필요 자원 1", "필요 자원 2"],
  "estimated_time_minutes": 30,
  "risk_level": "HIGH | MEDIUM | LOW"
}
```

### C. Sprint 1의 `audit_log` 테이블과의 연계 포인트
- `audit_log` 테이블에서 `status` 필드가 'NC(부적합)'로 판정된 레코드(Insert-only)를 트리거로 인지합니다.
- NC 발생 즉시 백그라운드 워커(또는 Upstash 큐)에서 상기 프롬프트를 호출하여 `nc_actions` (Sprint 2 추가 테이블)에 시정 조치 초안 레코드를 생성합니다. 
- 이후 원본 로그 ID(`audit_log_id`)를 FK로 매핑하여 무결성을 유지합니다.

---

## 3. Vision AI 파이프라인 선행 설계 (T2-001)

### A. Gemini Vision 입력 포맷
- **Base64 vs URL**: 실시간 Edge 검사의 특성을 고려하여, 네트워크 지연을 줄이고 스토리지 비용($0)을 준수하기 위해 사진 촬영 직후 브라우저에서 리사이징을 거친 후 **Base64 인코딩 문자열** 형태로 API(Route Handler)에 직접 전송하는 방식을 기본으로 합니다.

### B. STT 파이프라인과의 분리 기준 (DEC-003 준수)
- Vision 파이프라인과 STT(음성) 파이프라인은 단일 프롬프트에서 동시 처리하지 않고 API 엔드포인트를 완전히 분리(`/api/stt`와 `/api/vision`)합니다. 
- 이유: 멀티모달 동시 요청은 토큰 한도를 빠르게 소진시키며 응답 지연을 유발합니다. 사진 분석과 음성 매핑은 각각의 독립된 Edge 함수에서 병렬로 호출된 후, 프론트엔드 상태(Zustand 등)에서 하나로 머지(Merge)하여 DB에 최종 Insert합니다.

### C. Edge 하드웨어 스펙 미확정 시 임시 대응 방안
- 모바일 웹 브라우저 호환성을 위해 HTML5 `<input type="file" accept="image/*" capture="environment">`를 활용해 네이티브 카메라 뷰를 호출합니다.
- 브라우저 단에서 `canvas` API를 이용해 전송 전 해상도를 가로/세로 최대 1080px 수준으로 강제 다운사이징(Downsizing) 및 WebP 압축을 수행합니다.

---

## 4. F1-Q-001 Pagination 전략 확정

### A. Cursor 방식 vs Offset 방식 비교 및 선택 근거
- **비교**:
  - Offset 방식 (`LIMIT`, `OFFSET`): 구현이 직관적이나, 데이터가 누적될수록 뒷단 페이지 조회 시 DB 성능이 기하급수적으로 하락하며, 실시간 Insert-only 환경에서 데이터 삽입 시 페이지 뷰가 밀리는(중복/누락) 현상이 발생합니다.
  - Cursor 방식: 고유한 식별자(마지막으로 본 레코드의 ID나 타임스탬프)를 기준으로 이후 데이터를 가져오므로 대용량 데이터에서도 일정한 O(1) 성능을 유지하며 데이터 밀림이 없습니다.
- **선택 근거**: `audit_log`는 끊임없이 쌓이는 시계열성 'Insert-only' 테이블이므로, 성능과 일관성을 위해 무조건 **Cursor 기반 Pagination**을 채택합니다.

### B. UI-011 Audit Session List 화면과의 연동 인터페이스
- 화면 스크롤 하단 도달 시(Infinite Scroll) 프론트엔드에서 직전 API 응답의 `nextCursor` 값을 쿼리 파라미터로 넘겨 다음 청크를 요청합니다.
- React Query의 `useInfiniteQuery` 훅을 사용하여 상태를 관리합니다.

### C. 검색 필터 파라미터 스키마
```typescript
import { z } from 'zod';

export const AuditQuerySchema = z.object({
  cursor: z.string().optional(), // 마지막 항목의 ID 또는 타임스탬프
  limit: z.number().min(10).max(100).default(30),
  status: z.enum(['PASS', 'FAIL', 'NC', 'ALL']).default('ALL'),
  startDate: z.string().datetime().optional(),
  endDate: z.string().datetime().optional(),
  assigneeId: z.string().uuid().optional(),
  tenantId: z.string().uuid().optional() // 서버에서 JWT 기준 자동 오버라이딩 필수
});
```
