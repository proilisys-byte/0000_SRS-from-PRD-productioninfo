# MOCK-002_audit_report_mock_endpoint.md

## 1. 문서 목적
본 문서는 PRO ILI SMART 플랫폼의 Smart Audit F1 라인 관련 UI 개발 및 QA 테스트의 병렬 진행을 위한 Mock Endpoint(응답 시뮬레이터) 기준을 정의한다. 실제 백엔드 비즈니스 로직(Route Handler)이 완성되기 이전에 프론트엔드와 QA가 `API-003_audit_report_dto.md`와 `API-001_common_error_schema.md`에 명시된 데이터 규격 및 상태 전이를 테스트할 수 있도록 예측 가능한 가짜 응답(Mock Response) 구조와 시나리오 트리거 방식을 제공한다.

## 2. Mock 설계 원칙
**표 1. Mock 설계 원칙**

| 원칙 | 설명 | 적용 기준 |
| :--- | :--- | :--- |
| **DTO 스키마 100% 준수** | 반환되는 JSON은 `API-003` DTO의 타입, 필수값 제약을 완벽히 준수해야 함 | Zod 스키마 검증 통과 수준의 더미 데이터 |
| **상태 시나리오 제어** | 헤더(Header) 또는 특정 파라미터를 통해 성공, 에러, 지연 등의 응답 패턴 강제 트리거 가능 | `X-Mock-Scenario` 헤더 활용 |
| **일관된 식별자 매핑** | trace_id, session_id, tenant_id 등 상관관계가 있는 ID는 요청-응답 간 논리적 연계 유지 | 난수 생성 시 Prefix 부착 (`mock-session-001` 등) |
| **에러 표준화** | 실패 응답은 무조건 `API-001` 에러 스키마 규격으로 반환 | HTTP 상태 코드와 payload 일치화 |

## 3. 적용 범위 정의
**표 2. 적용 범위 정의**

| 범위 | 포함 대상 | 제외 대상 |
| :--- | :--- | :--- |
| **지원 영역** | 세션(Session) CRUD 및 상태 변경 응답, 리포트(Report) 생성/조회 응답, 목록 페이징 응답 | 복잡한 DB 트랜잭션 롤백 시뮬레이션 |
| **활용 부서** | Frontend (컴포넌트 바인딩 및 상태 트랜지션 UI 확인), QA (테스트 자동화 스크립트 뼈대 작성) | Backend (실 구현체 아님) |

## 4. Mock Endpoint 범위
**표 3. Mock Endpoint 범위**

| Base URL (예시) | Endpoint | Method | 역할 |
| :--- | :--- | :--- | :--- |
| `/api/mock/v1` | `/audit/sessions` | GET, POST | 세션 목록 조회 및 신규 생성 |
| `/api/mock/v1` | `/audit/sessions/{id}` | GET, PATCH | 세션 상세 조회 및 수정(임시저장/제출 등) |
| `/api/mock/v1` | `/audit/sessions/{id}/reports` | POST | 리포트 수동 생성 요청 |
| `/api/mock/v1` | `/audit/reports/{id}` | GET | 단일 리포트 (최신/이력) 조회 |

## 5. Session 관련 Mock 응답
**표 4. Session 관련 Mock 응답 (예시)**

```json
// GET /api/mock/v1/audit/sessions/mock-sess-123
{
  "session_id": "mock-sess-123",
  "target_id": "eq-500",
  "tenant_id": "tenant-A",
  "site_id": "site-1",
  "status": "InProgress",
  "created_by": "auditor-01",
  "checklist_data": { "item1": "pass", "item2": "fail" },
  "stt_data": { "raw_text": "오일 누유가 발견되었습니다." },
  "created_at": "2026-04-24T10:00:00Z",
  "updated_at": "2026-04-24T10:30:00Z"
}
```

## 6. Report 관련 Mock 응답
**표 5. Report 관련 Mock 응답 (예시)**

```json
// GET /api/mock/v1/audit/reports/mock-rep-999
{
  "report_id": "mock-rep-999",
  "session_id": "mock-sess-123",
  "tenant_id": "tenant-A",
  "status": "Generated",
  "is_latest": true,
  "summary_data": {
    "score": 85,
    "findings": ["오일 누유 조치 필요"]
  },
  "created_by": "auditor-01",
  "generated_at": "2026-04-24T11:00:00Z"
}
```

## 7. 목록 / 상세 / 생성 / 진행중 / 오류 응답 패턴
**표 6. 기본 응답 패턴**

| 시나리오 | HTTP 상태 | 반환 데이터 규칙 |
| :--- | :--- | :--- |
| **목록 (List)** | 200 OK | `data` 배열에 고정된 더미 객체 5개 반환, `meta.total_count` 100 등 페이징 구조 모사 |
| **상세 (Detail)** | 200 OK | URL Path의 ID값을 응답 객체의 ID 필드에 그대로 삽입하여 반환 (논리성 유지) |
| **생성 (Create)** | 201 Created | 임의의 UUID를 생성하여 반환 객체에 매핑, `status`는 'Draft' (세션) 또는 'Generated' (리포트) |
| **삭제/취소** | 200 OK | `{ "success": true, "message": "Cancelled" }` 등 기본 Ack 응답 반환 |

## 8. 상태별 Mock 시나리오 (Header Trigger)
클라이언트가 헤더에 `X-Mock-Scenario`를 전송하여 의도적으로 특정 엣지 케이스를 테스트할 수 있도록 지원한다.

**표 7. 상태별 Mock 시나리오**

| `X-Mock-Scenario` 값 | 대상 Endpoint | 트리거되는 행동 | 기대 효과 (FE/QA 테스트 목적) |
| :--- | :--- | :--- | :--- |
| `force-timeout` | ALL | 응답을 5초간 지연시킨 후 504 반환 | 클라이언트의 로딩 스피너 및 타임아웃 에러 처리 화면 확인 |
| `force-invalid-state` | PATCH Session | HTTP 409, Invalid State 에러 반환 | 상태 역행 시도에 대한 에러 토스트 모달 확인 |
| `report-superseded` | GET Report | `status`를 "Superseded", `is_latest`를 false로 덮어써서 반환 | 이력본 조회 시 읽기 전용 UI(수정 버튼 비활성화 등) 전환 확인 |
| `session-submitted` | GET Session | `status`를 "Submitted"로 반환 | [제출] 버튼 숨김 및 [재개/리포트생성] 버튼 활성화 로직 확인 |

## 9. 에러 코드별 Mock 기준
Mock 에러 응답은 반드시 `API-001_common_error_schema.md` 형식을 따른다.

**표 8. 에러 코드별 Mock 기준 (예시)**

| `X-Mock-Scenario` | HTTP | JSON Response Body (Mock) |
| :--- | :--- | :--- |
| `force-400` | 400 | `{ "error": { "code": "INVALID_INPUT", "message": "입력값 오류", "details": ["target_id 누락"], "trace_id": "mock-trace-123" } }` |
| `force-403` | 403 | `{ "error": { "code": "FORBIDDEN", "message": "권한 없음", "details": ["tenant 불일치"], "trace_id": "mock-trace-124" } }` |

## 10. trace_id / timestamp / actor / tenant / site / target 연계 기준
**표 9. 식별자 매핑 및 연계 기준**

| 항목 | Mock 생성 규칙 |
| :--- | :--- |
| **trace_id** | 요청 헤더에 `X-Trace-Id`가 있으면 그대로 반환, 없으면 `mock-trc-{난수}` 생성 |
| **tenant/site** | 토큰(Authorization) 파싱 로직을 모사할 수 없으므로, 요청 페이로드에 존재하는 값을 우선 반환 |
| **timestamp** | `created_at`, `updated_at` 필드는 Mock 서버가 요청을 받은 현재 ISO-8601 시각으로 동적 생성 |

## 11. RH / DTO / UI / TEST 정합성 기준
**표 10. RH / DTO / UI / TEST 정합성 기준**

| 연결 문서 | 연동 방식 및 제한 |
| :--- | :--- |
| `API-003` (DTO) | 반환값의 키(Key) 명칭, 타입(String, Enum)을 DTO 기준으로 강제 파싱/검증 |
| `UI-010` (UI) | 상태값 배지 색상 변화 확인을 위해 `X-Mock-Scenario`로 Draft/InProgress/Submitted 전환 지원 |
| `TEST-F1-001/002` | QA 스크립트 작성 시 Mock 서버 주소로 초기 테스트 실행 가능 구조 보장 |

## 12. Sprint 우선순위
**표 11. Sprint 우선순위 정리**

| Mock 기능 | 구현 필수 여부 (Sprint 1) | 비고 |
| :--- | :--- | :--- |
| 세션 CRUD 기본 응답 | P0 | 프론트엔드 폼 바인딩용 필수 |
| `X-Mock-Scenario` 에러 트리거 | P0 | 글로벌 에러 핸들링 검증용 필수 |
| 리포트 상태(최신/이력) Mocking | P1 | Superseded UI 분기 테스트용 |

## 13. 완료 기준 (Definition of Done)
**표 12. 완료 기준**

| 기준 | 검증 방법 |
| :--- | :--- |
| DTO 호환 | Mock 응답을 프론트엔드의 Zod/TypeScript 인터페이스가 에러 없이 파싱 성공해야 함 |
| 에러 스키마 호환 | 4xx 응답 발생 시 프론트엔드 공통 에러 인터셉터가 정상적으로 메시지를 추출해야 함 |
| 가용성 보장 | 로컬 개발 환경(MSW, Postman Mock Server 등)에서 네트워크 지연 없이 즉각 동작해야 함 |

## 14. 다음 단계 작성 가이드
- 프론트엔드 팀은 본 문서를 기반으로 MSW(Mock Service Worker) 핸들러 코드를 작성하거나 Postman Mock Server 환경을 구성한다.
- QA 팀은 백엔드 API 완료 전까지 본 Mock 엔드포인트를 타겟으로 기초 자동화 스크립트(Test Suite) 작성을 시작한다.

## 15. Gemini 3.1 Pro 자체 검토 체크리스트
- [x] 프론트엔드/QA 병렬 작업을 위한 응답 패턴이 충분히 구체적인가?
- [x] `API-003`(DTO) 및 `API-001`(Error Schema)과의 정합성을 훼손하는 설계가 없는가?
- [x] `X-Mock-Scenario` 등을 통한 의도적 예외 상황(Timeout, Invalid State) 유발 방법이 명시되었는가?
- [x] Mock이 실제 비즈니스 로직(RH)의 복잡성을 대체하지 않도록 역할 제한(표 2)이 선언되었는가?
