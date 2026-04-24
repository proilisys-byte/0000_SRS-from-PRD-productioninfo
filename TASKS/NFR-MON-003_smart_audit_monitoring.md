# NFR-MON-003_smart_audit_monitoring.md

## 1. 목적
Vercel + Supabase Serverless 환경에서 $0 추가 비용 원칙을 유지하면서 프로덕션 운영에 필요한 모니터링, 로깅, 알림, 성능 관리 체계를 정의한다. (Sprint 4 T4-001 대응 문서)

## 2. SLI/SLO 정의

시스템의 신뢰성을 정량적으로 측정하기 위한 핵심 지표(SLI)와 목표 수준(SLO)을 다음과 같이 정의한다.

| SLI 항목 | 측정 방법 | SLO 목표 | 위반 시 알림 |
|---------|---------|---------|-----------|
| Audit 리포트 생성 시간 | DB 매핑 완료 타임스탬프 간격 | p95 ≤ 10분 | 즉시 Slack 웹훅 (#alerts-critical) |
| STT 음성 인식 Fallback율 | 수동 정정 횟수(fallback) / 전체 STT 요청 건수 | ≤ 8% | 일간 리포트 (#alerts-daily) |
| API 오류율 (5xx) | Vercel Analytics 기준 HTTP 5xx 응답 비율 | ≤ 0.5% | 즉시 Slack 웹훅 (#alerts-critical) |
| Gemini API 429 발생율 | 커스텀 로그 기반 (RATE_429 에러 집계) | ≤ 2% | 일간 리포트 (#alerts-ai) |
| 서비스 가용성 (Uptime) | Vercel 상태 페이지 및 헬스체크 핑 | ≥ 99.5% | 즉시 이메일 & Slack (#alerts-critical) |
| **[추가] NC 초안 생성 시간** | API 요청 시간 ~ DB 저장 완료 시간 간격 | p95 ≤ 5분 | 주간 리포트 |
| **[추가] Bulk Import 파싱 실패율**| 파싱 에러 발생 파일 수 / 전체 업로드 파일 수 | ≤ 5% | 주간 리포트 |
| **[추가] 로그인 실패(권한 위반)율**| 401/403 에러 응답 수 / 전체 로그인 시도 | ≤ 3% | 즉시 Slack 웹훅 (#alerts-security) |

## 3. 로깅 전략

### 3.1. 구조화 로그 포맷
서버 내 발생하는 모든 로그는 식별 및 검색의 용이성을 위해 아래 JSON 포맷을 따른다.

```json
{
  "timestamp": "2026-04-24T21:30:00.000Z",
  "level": "info",
  "service": "ai-pipeline",
  "trace_id": "tr_9f8a7b6c5d",
  "user_id": "usr_1234abcd",
  "session_id": "sess_9876xyz",
  "event": "mapping_completed",
  "duration_ms": 1450,
  "metadata": {
    "token_used": 1050,
    "model": "gemini-1.5-pro",
    "fallback_triggered": false
  }
}
```

### 3.2. 반드시 로깅해야 할 이벤트 목록

| 카테고리 | 이벤트명 (`event`) | 로그 레벨 | 필수 `metadata` 항목 |
|---|---|---|---|
| **AI 파이프라인** | `stt_call_completed` | info | `duration_ms`, `audio_length_sec` |
| | `vision_call_completed` | info | `duration_ms`, `image_size_kb` |
| | `mapping_completed` | info | `mapped_fields_count`, `missing_fields` |
| | `fallback_triggered` | warn | `original_text`, `corrected_text` |
| **비즈니스** | `session_created` | info | `tenant_id`, `audit_type` |
| | `bulk_import_completed` | info | `total_rows`, `failed_rows`, `file_name` |
| | `nc_ticket_created` | info | `nc_reason`, `severity` |
| | `pdf_generated` | info | `file_size_kb`, `render_time_ms` |
| **보안** | `login_success` / `login_failed`| info / warn | `ip_address`, `method` |
| | `permission_violation` | error | `attempted_action`, `required_role` |
| | `mfa_verified` | info | `mfa_type` |
| **시스템** | `gemini_rate_limit_exceeded`| error | `retry_count`, `rpm_limit` |
| | `vercel_timeout_warning` | warn | `route_path`, `duration_ms` |
| | `db_connection_failed` | error | `error_code`, `timeout_ms` |

### 3.3. Vercel 환경에서의 로그 수집 방법
- **접근법**: `console.log`로 위 JSON 형태를 표준 출력. Vercel의 Log Drains 기능을 활용.
- **도구 비교**:
  - **Vercel 내장 Logs**: 추가 설정이 필요 없으나 보관 기간 제한(Hobby/Pro 등급에 따라 짧음)과 복잡한 집계 쿼리 불가.
  - **Axiom Free Tier**: Vercel 연동 1-Click 지원. 한 달 500GB 스토리지, 무제한 데이터 보존(요금제에 따라 상이), 강력한 APL(Axiom Processing Language) 쿼리 지원으로 대시보드 구성에 매우 유리함.
  - **Logflare**: 스트리밍 기능은 우수하나 설정 및 쿼리 편의성이 Axiom 대비 다소 낮음.
- **선택 도구 및 설정**: **Axiom Free Tier** 채택 ($0 원칙 부합). Vercel Integration에서 Axiom 앱을 추가하여 모든 Vercel Function 로그를 자동으로 Axiom으로 스트리밍(Drain) 하도록 설정.

## 4. 성능 모니터링

### 4.1. Vercel Analytics 활용
- `@vercel/analytics` 패키지를 설치하여 루트 Layout에 `<Analytics />` 컴포넌트 삽입. LCP, FID, CLS 등 Core Web Vitals 자동 수집.
- **커스텀 성능 지표**: `useReportWebVitals` 훅을 활용하여 사용자 커스텀 이벤트(예: 페이지 전환 소요 시간 등)를 Analytics 이벤트로 추가 전송 가능.

### 4.2. AI 파이프라인 성능 추적
- 각 Gemini API 호출의 지연시간을 계측하여, 메타데이터와 함께 DB(또는 Axiom)에 기록.

```typescript
// src/lib/monitoring/ai-tracker.ts

export async function trackedGeminiCall<T>(
  callFn: () => Promise<T>,
  metadata: { type: 'stt' | 'vision' | 'audit' | 'nc', session_id: string }
): Promise<T> {
  const startTime = Date.now();
  let success = true;
  let errorDetail = null;

  try {
    const result = await callFn();
    return result;
  } catch (error) {
    success = false;
    errorDetail = error instanceof Error ? error.message : String(error);
    throw error;
  } finally {
    const duration_ms = Date.now() - startTime;
    // JSON 로그 출력 (Axiom이 수집)
    console.log(JSON.stringify({
      timestamp: new Date().toISOString(),
      level: success ? 'info' : 'error',
      service: 'ai-pipeline',
      event: 'gemini_call_executed',
      duration_ms,
      metadata: { ...metadata, success, errorDetail }
    }));
    
    // 장기 보존이 필요한 핵심 지표는 Supabase 통계 테이블에 비동기 저장 가능
    // supabase.from('ai_metrics').insert({ type: metadata.type, duration_ms, success })
  }
}
```

- **백분위 집계 쿼리 (Axiom APL 예시)**:
  `['vercel-logs'] | where event == "gemini_call_executed" | summarize p50=percentile(duration_ms, 50), p95=percentile(duration_ms, 95) by metadata.type`

### 4.3. Supabase 데이터베이스 모니터링
- **느린 쿼리 식별**: Supabase 대시보드 내 "Query Performance" 메뉴에서 내부적으로 활성화된 `pg_stat_statements` 확장 기능을 통해 평균 실행 시간 순으로 느린 쿼리를 파악.
- **Free Tier 한계 모니터링**: 
  - DB 크기 (500MB 한도), 월간 API 통신 대역폭, Storage 용량(1GB 한도).
- **알림 설정**: Supabase 설정 내 "Project Usage"에서 사용량 알림(Usage Alerts)을 80% 및 90% 도달 시 이메일과 커스텀 웹훅(Slack 연동)으로 발송되도록 구성.

## 5. 알림 시스템 구성

- **도구**: Slack Webhook ($0 비용)
- **채널 구조**:
  - `#alerts-critical`: 시스템 다운, 500 에러, DB 한도 90% 접근 (즉각 대응 필요)
  - `#alerts-daily`: 일간 가입자 수, Bulk Import 성공 건수 등 요약 지표
  - `#alerts-ai`: 429 Rate Limit 초과, 타임아웃 경고, Fallback 증가율 등 AI 전용
  - `#alerts-security`: 권한 위반(403), 비정상 로그인 시도 

```typescript
// src/lib/notifications.ts
import { AppError } from './errors';

// 중복 알림 방지를 위한 메모리 캐시 (Edge 등 환경 한계상 DB/Redis 고려 가능)
const alertCache = new Map<string, number>();

export async function sendSlackAlert(
  channel: '#alerts-critical' | '#alerts-daily' | '#alerts-ai' | '#alerts-security',
  title: string,
  message: string,
  error?: Error | AppError
) {
  // 중복 알림 방지 로직 (10분 내 동일 title 발송 차단)
  const cacheKey = `${channel}:${title}`;
  const now = Date.now();
  if (alertCache.has(cacheKey) && (now - alertCache.get(cacheKey)!) < 10 * 60 * 1000) {
    return; // 스킵
  }
  alertCache.set(cacheKey, now);

  const webhookUrl = process.env[`SLACK_WEBHOOK_${channel.replace('#', '').toUpperCase().replace('-', '_')}`];
  if (!webhookUrl) return;

  // Slack Block Kit 포맷 구성
  const payload = {
    blocks: [
      {
        type: "header",
        text: { type: "plain_text", text: `🚨 ${title}` }
      },
      {
        type: "section",
        text: { type: "mrkdwn", text: message }
      },
      ...(error ? [{
        type: "section",
        text: { type: "mrkdwn", text: `*Error Details:*\n\`\`\`${error.message}\n${(error as AppError).code || ''}\`\`\`` }
      }] : [])
    ]
  };

  await fetch(webhookUrl, {
    method: 'POST',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify(payload)
  }).catch(console.error);
}
```

## 6. 운영 대시보드 구성

Supabase SQL Editor에서 정기적으로 실행하여 지표를 파악할 수 있는 View를 구성한다.

```sql
-- 일별 AI 파이프라인 성능 요약 뷰 (지표 적재용 테이블 'ai_metrics'가 있다고 가정)
CREATE OR REPLACE VIEW daily_ai_performance AS
SELECT 
  DATE(created_at) AS metric_date,
  ai_type,
  COUNT(*) AS total_calls,
  COUNT(*) FILTER (WHERE success = false) AS error_calls,
  ROUND(AVG(duration_ms)) AS avg_duration_ms,
  PERCENTILE_CONT(0.95) WITHIN GROUP (ORDER BY duration_ms) AS p95_duration_ms
FROM ai_metrics
GROUP BY DATE(created_at), ai_type
ORDER BY metric_date DESC;

-- 주간 SLO 달성률 요약 뷰 (로그/메트릭 집계 테이블 기반 가상 예시)
CREATE OR REPLACE VIEW weekly_slo_report AS
SELECT 
  DATE_TRUNC('week', created_at) AS week_start,
  -- 에러율 (목표: < 0.5%)
  ROUND((COUNT(*) FILTER (WHERE status_code >= 500)::numeric / COUNT(*)) * 100, 3) AS error_rate_percent,
  -- AI 타임아웃/429 실패율 (목표: < 2%)
  ROUND((COUNT(*) FILTER (WHERE error_code LIKE 'RATE_429%')::numeric / NULLIF(COUNT(*), 0)) * 100, 3) AS ai_rate_limit_percent
FROM request_logs
GROUP BY DATE_TRUNC('week', created_at)
ORDER BY week_start DESC;
```

## 7. 인시던트 대응 런북 (Runbook)

**시나리오 1: Gemini API 전면 장애**
1. **감지**: `#alerts-ai` 채널에 502/504 에러 알림 급증 또는 Axiom 대시보드에서 Gemini API 실패율 10% 초과 알람.
2. **1차 대응**: Google Cloud 상태 페이지 확인. 외부 장애 확정 시 Vercel 환경 변수 `FALLBACK_AI_MODE`를 `true`로 변경 후 재배포 (수동 입력 UI 모드로 전환).
3. **사용자 공지**: 앱 내 상단 배너에 "현재 AI 서버망 장애로 수동 입력만 가능합니다." 긴급 공지 활성화.
4. **복구 확인**: API 정상 응답 여부를 모니터링하고, 구글 복구 공지 확인 후 원래 환경 변수로 롤백 배포.
5. **사후 분석**: 타임아웃 발생 건에 대한 고객 데이터 유실 여부 파악 및 재안내(CS).

**시나리오 2: Vercel 배포 후 PDF 생성 실패 급증**
1. **감지 기준**: `#alerts-critical`에 PDF 생성 관련 500 에러(예: `SYS_500_PDF_RENDER_FAIL`)가 10분 내 5건 이상 발생.
2. **즉시 롤백 판단 기준**: 이전 배포 버전 대비 해당 에러가 급증했고, 원인 파악에 10분 이상 소요될 것으로 예상될 때.
3. **롤백 절차**: 
   - Vercel Dashboard 접속 -> Deployments 탭.
   - 에러 발생 이전의 안정적인(Successful) 최신 배포본 선택.
   - 우측 메뉴에서 "Promote to Production" (또는 "Rollback") 클릭.
4. **분석**: 로컬 환경에서 해당 커밋 체크아웃 후 브라우저(html2pdf) 호환성 결함 확인 및 수정.

**시나리오 3: Supabase DB 용량 Free Tier 임박 (80% 초과)**
1. **감지**: Supabase Usage Alert 이메일 및 `#alerts-critical` 웹훅 수신 (DB 크기 400MB 도달).
2. **긴급 데이터 정리 대상 식별**: 오래된 `audit_log` 내역(1개월 이상 경과분), 비활성 사용자 세션의 더미 레코드 식별 쿼리 실행.
3. **정리 실행**: 법적 보존 연한에 해당하지 않는 임시 데이터/로그 삭제 스크립트 실행 또는 외부 Object Storage 백업 후 DB 삭제.
4. **유료 플랜 전환 승인 절차**: 데이터 증가 추이 그래프 캡처 및 대표이사(S-2)에게 Pro 플랜($25/월) 결제 승인 기안 즉각 상신.

**시나리오 4: 보안 취약점 긴급 패치 필요**
1. **취약점 접수**: npm audit 경고, KISA 권고, 모의 해킹 결과 등을 통해 취약점(예: 의존성 패키지의 RCE 결함) 감지.
2. **심각도 분류**: CVSS 스코어 기반 (High/Critical 시 24시간 내 패치 의무).
3. **핫픽스 배포**: 
   - `hotfix/security-patch` 브랜치 생성 및 패키지 업데이트.
   - 로컬 E2E 테스트 통과 즉시 main 머지 및 Vercel 강제 배포.
4. **감사 로그 검토**: 취약점이 존재했던 기간 동안 `audit_log` 테이블을 조회하여 비정상적인 권한 탈취 시도(FORB_403) 이력 존재 여부 전수 검사.

================================================================
## 완료 기준 체크리스트

| 항목 | 완료 여부 |
|------|:---:|
| T3-001: Phase 1/2 비교 표 완성 | [x] |
| T3-001: DataImportSource 추상 인터페이스 전체 코드 포함 | [x] |
| T3-001: 혁신바우처 API Route Handler 명세 완성 | [x] |
| T3-001: 마이그레이션 롤백 절차 명시 | [x] |
| NFR-MON: SLI/SLO 5개 이상 정의 완료 | [x] |
| NFR-MON: 구조화 로그 포맷 JSON 예시 포함 | [x] |
| NFR-MON: trackedGeminiCall 래퍼 함수 코드 포함 | [x] |
| NFR-MON: daily_ai_performance View SQL 코드 포함 | [x] |
| NFR-MON: weekly_slo_report View SQL 코드 포함 | [x] |
| NFR-MON: 4개 인시던트 런북 단계별 완성 | [x] |
================================================================
