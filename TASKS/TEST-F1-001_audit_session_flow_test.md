# TEST-F1-001_audit_session_flow_test.md

## 1. 문서 목적
본 문서는 PRO ILI SMART 플랫폼의 Smart Audit F1 라인 중 '감사 세션(Audit Session)'의 전체 수명 주기(Lifecycle) 및 상태 전이 흐름을 검증하기 위한 QA 인수 테스트(Acceptance Test) 기준을 정의한다. 이를 통해 권한 제어, 상태 불일치 방지, 데이터 무결성 및 공통 에러 처리가 의도대로 작동하는지 검증한다.

## 2. 테스트 설계 원칙
**표 1. 테스트 설계 원칙**

| 원칙 | 설명 | 적용 기준 |
| :--- | :--- | :--- |
| **상태 기반 독립 검증** | 세션의 각 상태(Draft, InProgress 등)에서 허용/차단되는 액션을 독립적으로 검증 | 상태 머신 규칙 위반 시 무조건 Fail 처리 |
| **RBAC / Multi-tenant 최우선** | 사용자 권한 및 소속 (Tenant/Site) 횡단 접근 시나리오 강제 삽입 | 403 Forbidden 응답 및 격리 수준 검증 |
| **추적 가능성 (Traceability)** | 모든 테스트 결과에는 `trace_id` 및 Audit Log 검증 단계 포함 | 에러 응답 및 로그 정합성 교차 확인 |
| **Black-box + Gray-box** | API 응답(Black-box) 검증 및 데이터베이스 상태/감사 로그 기록(Gray-box) 동시 검증 | 단순 응답 200 OK 여부로 Pass 처리 불가 |

## 3. 적용 범위 정의
**표 2. 적용 범위 정의**

| 범위 | 포함 대상 | 제외 대상 (Sprint 2+) |
| :--- | :--- | :--- |
| **테스트 대상 (Sprint 1)** | 세션 생성/수정, 임시저장, 제출, 재개(Reopen), 종료/취소 상태 전이, STT 보조 데이터 조회 | Vision AI 데이터 검증, 복합 결재선 승인 흐름 |
| **검증 영역** | RBAC 접근 제어, 데이터 격리 (Tenant/Site), API 에러 스키마 준수 (`API-001`), 상태 규칙 무결성 | 성능/부하 테스트 (NFR 별도 문서) |

## 4. 전제 조건
**표 3. 전제 조건**

| 항목 | 상세 내용 |
| :--- | :--- |
| **테스트 계정 세팅** | Tenant-A/Site-1 소속 Auditor, Viewer 계정 / Tenant-B 소속 Auditor 계정 사전 생성 |
| **기초 데이터** | 유효한 대상 자산(Target/Eq) 정보 사전 적재 (Seed Data 연계) |
| **환경 설정** | 테스트용 DB (Supabase Branch 또는 별도 스키마) 연결 및 Mock 인증 토큰 유효성 확보 |

## 5. 역할별 테스트 관점
**표 4. 역할별 테스트 관점**

| 역할 (Role) | 주요 검증 관점 | 예상 결과 |
| :--- | :--- | :--- |
| **Auditor (소속 Site)** | 세션 전체 흐름 정상 동작 여부 (Happy Path) | 성공 응답 (200/201) 및 상태 전이 허용 |
| **Auditor (타 Site)** | 타 Site 대상 세션 생성 및 조회 시도 | 차단 (403/404) 및 상태 불변 |
| **Viewer** | 쓰기 액션(생성/수정/제출/재개) 시도 | 차단 (403 Forbidden) |
| **System Admin** | 데이터 조회 범위 한계 검증 | Tenant 상관없이 전체 Read-Only 허용 (수정 차단) |

## 6. 세션 생성 시나리오
**표 5. 세션 생성 시나리오**

| TC ID | 테스트 시나리오 | 입력값/조작 | 기대 결과 |
| :--- | :--- | :--- | :--- |
| SESS-C-01 | 정상 생성 (Auditor) | 올바른 target_id, 필수 필드 입력 | HTTP 201, 상태: `Draft`, Session ID 반환 |
| SESS-C-02 | 권한 부족 생성 시도 (Viewer) | 올바른 target_id, 필수 필드 입력 | HTTP 403 Forbidden |
| SESS-C-03 | 타 Tenant Site 자산으로 생성 시도 | 타 Tenant의 target_id 입력 | HTTP 403 또는 404 Not Found (격리) |
| SESS-C-04 | 유효하지 않은 데이터 입력 | target_id 누락 또는 문자열 포맷 오류 | HTTP 400 Bad Request, API-001 스키마 에러코드 검증 |

## 7. 세션 수정 / 임시저장 시나리오
**표 6. 세션 수정 / 임시저장 시나리오**

| TC ID | 테스트 시나리오 | 입력값/조작 | 기대 결과 |
| :--- | :--- | :--- | :--- |
| SESS-U-01 | Draft 상태 세션 내용 수정 | checklist 등 일부 내용 수정 | HTTP 200, 상태: `InProgress`로 전이됨 검증 |
| SESS-U-02 | InProgress 상태 세션 임시저장 | 추가 내용 수정 | HTTP 200, 상태: `InProgress` 유지 검증 |
| SESS-U-03 | 권한 없는 사용자(타인)가 수정 시도 | 타인이 작성한 세션 ID로 PATCH | HTTP 403 Forbidden |
| SESS-U-04 | 종료된(Closed) 세션 수정 시도 | Closed 상태 Session ID로 PATCH | HTTP 409 Conflict, "Invalid state transition" 에러 |

## 8. 세션 제출 시나리오
**표 7. 세션 제출 시나리오**

| TC ID | 테스트 시나리오 | 입력값/조작 | 기대 결과 |
| :--- | :--- | :--- | :--- |
| SESS-S-01 | 필수값 충족 상태에서 제출 | checklist 작성 완료 | HTTP 200, 상태: `Submitted` 전이 검증 |
| SESS-S-02 | 필수값 미달 상태에서 제출 시도 | checklist 필수항목 누락 | HTTP 400 Bad Request (Validation Error) |
| SESS-S-03 | 이미 제출된 세션 다시 제출 시도 | Submitted 상태 Session ID 재호출 | HTTP 409 Conflict |

## 9. 세션 재개(Reopen/Resume) 시나리오
**표 8. 세션 재개 시나리오**

| TC ID | 테스트 시나리오 | 입력값/조작 | 기대 결과 |
| :--- | :--- | :--- | :--- |
| SESS-R-01 | Submitted 상태 세션 Reopen | Reopen API 호출 | HTTP 200, 상태: `InProgress` 전이 검증 |
| SESS-R-02 | Closed 상태 세션 Reopen 시도 | Closed 상태 Session ID 호출 | HTTP 409 Conflict (종료된 세션 재개 불가) |
| SESS-R-03 | 리포트가 생성된 세션 Reopen 시도 | 리포트가 존재하는 Submitted 세션 호출 | HTTP 200, 상태: `InProgress`, 기존 리포트 상태 연동 검증(`TEST-F1-002` 참조) |

## 10. 세션 종료 / 취소 시나리오
**표 9. 세션 종료 / 취소 시나리오**

| TC ID | 테스트 시나리오 | 입력값/조작 | 기대 결과 |
| :--- | :--- | :--- | :--- |
| SESS-X-01 | 정상 종료 (Close) | Submitted 세션 Close API 호출 | HTTP 200, 상태: `Closed` 전이 검증 |
| SESS-X-02 | 작성 중 취소 (Cancel) | InProgress 세션 Cancel API 호출 | HTTP 200, 상태: `Cancelled` 전이 검증 |
| SESS-X-03 | 취소된 세션 수정 시도 | Cancelled 세션 ID로 PATCH 호출 | HTTP 409 Conflict |

## 11. 상태 전이 검증 기준
**표 10. 상태 전이 검증 기준**

| 허용되는 상태 전이 (Valid) | 차단되는 상태 전이 (Invalid) |
| :--- | :--- |
| Draft -> InProgress | Draft -> Submitted (필수값 누락 시) |
| InProgress -> Submitted | Submitted -> Draft |
| Submitted -> InProgress (Reopen) | Closed -> InProgress |
| InProgress -> Cancelled | Cancelled -> Draft / InProgress |
| Submitted -> Closed | - |

## 12. 권한 / tenant / site 검증 기준
**표 11. 권한 / tenant / site 검증 기준**

| 검증 항목 | 상세 기준 | 확인 방법 |
| :--- | :--- | :--- |
| **Tenant 격리** | DB 쿼리 레벨에서 Tenant 분리가 작동하는가? | Token 헤더 변조를 통한 타 Tenant 세션 ID 호출 시 404 리턴 검증 |
| **Site Scope** | 동일 Tenant라도 타 Site 접근이 통제되는가? | 토큰 페이로드 상의 허용된 Site ID 목록 교차 검증 |

## 13. STT 보조 입력 검증 기준
**표 12. STT 보조 입력 검증 기준**

| 검증 항목 | 기대 결과 |
| :--- | :--- |
| **데이터 무결성** | GET Session 호출 시 STT 원본 텍스트 데이터가 수정되지 않고 보존되어 반환됨 |
| **수정 불가 확인** | PATCH Session 호출 페이로드에 STT 필드를 포함하여 전송 시 무시되거나 에러 반환 (Read-Only) |

## 14. 감사 로그 기록 검증
**표 13. 감사 로그 기록 검증**

| 액션 | 확인 대상 필드 | 기대 결과 (DB 검증) |
| :--- | :--- | :--- |
| **생성/수정/제출/취소** | `Audit_Log` 테이블 적재 여부 | action_type 일치, session_id (target), actor_id 정상 기록, timestamp 일치 |
| **Trace 연계** | `trace_id` 기록 여부 | API 요청 헤더의 `X-Trace-Id`와 Audit Log DB의 `trace_id` 일치 확인 |

## 15. 공통 에러 응답 검증
**표 14. 공통 에러 응답 검증**

| 시나리오 | 기대되는 `API-001` 기반 에러 응답 구조 확인 |
| :--- | :--- |
| 400 Bad Request | code: "INVALID_INPUT", details에 필드명 명시 여부 |
| 403 Forbidden | code: "FORBIDDEN", details에 사유(Role/Site 제한) 명시 여부 |
| 409 Conflict | code: "INVALID_STATE", details에 "current_state", "requested_action" 명시 여부 |

## 16. 증빙 자료 기준
- 테스트 자동화 리포트 (Jest/Postman Newman 실행 로그)
- 에러 응답 시 반환된 JSON 페이로드 스크린샷 (trace_id 포함)
- 해당 트랜잭션의 Audit Log DB 레코드 덤프 캡처

## 17. 결함 심각도 기준
**표 15. 결함 심각도 기준**

| 심각도 | 조건 | 조치 기한 |
| :--- | :--- | :--- |
| **Critical** | 타 Tenant 데이터 조회/수정 가능, 상태 제어 우회 가능, 서버 패닉(500) | 즉시 (배포 불가) |
| **High** | 공통 에러 스키마 미준수, Audit Log 기록 누락, 필수값 검증 우회 | Sprint 내 해결 |
| **Medium** | 에러 메시지(details) 불명확, 특정 상태의 중복 액션 처리 불일치 | 차기 핫픽스 |
| **Low** | 오타, 내부 디버그 메시지 노출 등 | 차기 Sprint 반영 |

## 18. Sprint 우선순위
Sprint 1에서는 SESS-C, SESS-U, SESS-S, SESS-X 시나리오의 Happy Path 및 권한 격리(Tenant/Site) 검증을 최우선으로 진행하며, 자동화 스크립트에 필수 포함시킨다.

## 19. 완료 기준 (Definition of Done)
- 표 5~9의 모든 시나리오에 대한 검증 케이스(Postman/Jest) 작성 완료
- 모든 케이스 실행 결과 Pass (Critical/High 결함 0건)
- `trace_id` 기반 로그 추적 가능성 시연 성공

## 20. 다음 단계 작성 가이드
- QA 팀은 본 문서를 기반으로 실제 API Endpoint(`MOCK-002` 참조)를 호출하는 Postman Collection 작성 및 CI 파이프라인(GitHub Actions) 연동 스크립트를 작성한다.

## 21. Gemini 3.1 Pro 자체 검토 체크리스트
- [x] 세션의 모든 상태(Draft ~ Cancelled) 전이 흐름이 빠짐없이 설계되었는가?
- [x] 권한 제어(Multi-tenant, Site-based RBAC) 검증 관점이 명확한가?
- [x] API 응답뿐만 아니라 Audit Log 기록 등 Gray-box 검증 기준이 포함되었는가?
- [x] STT 보조 데이터의 읽기 전용 속성이 테스트 항목에 반영되었는가?
