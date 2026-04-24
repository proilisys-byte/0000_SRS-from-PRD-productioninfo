# F1-RH-001_audit_route_handler.md

## 1. 문서 목적
본 문서는 PRO ILI SMART 플랫폼의 Smart Audit Core(F1)에서 세션 및 리포트 관련 요청을 처리하는 서버 진입점(Route Handler)의 설계 기준과 명세를 정의한다. 클라이언트의 요청을 가장 먼저 받아 인증, 인가, 데이터 검증을 수행하고 도메인 로직으로 위임하는 책임을 규정한다.

## 2. Route Handler 설계 원칙
**표 1. Route Handler 설계 원칙**
| 원칙 | 내용 |
|---|---|
| **경계 책임 집중** | Route Handler는 비즈니스 로직 본체가 아닌 서버 입구 경계로서 기능한다. |
| **선행 차단(Fail-Fast)** | 인증, 권한, Tenant/Site, 입력값 오류를 비즈니스 로직 진입 전에 즉시 차단(4xx)한다. |
| **Command/Query 명확한 분리** | 상태를 변경하는 Command와 단순 조회하는 Query를 동일한 방식으로 다루지 않는다. |
| **투명한 DTO 매핑** | 모든 입출력은 `API-003_audit_report_dto.md` 기준을 따라 매핑되며, 내부 모델을 직접 노출하지 않는다. |
| **추적성 보장** | 공통 에러 변환 및 감사 로그 연결을 통해 모든 진입점의 추적성을 보장한다. |

## 3. 적용 범위 정의
**표 2. 적용 범위 정의**
| 범위 | 내용 |
|---|---|
| **Sprint 1 포함 (구현 대상)** | 세션(생성, 목록/상세 조회, 임시저장/수정, 제출, 재개, 종료/취소), 리포트(생성 요청, 목록/상세 조회, 최신본/이력본 조회, Superseded 포함 조회), 기본 오류 응답 및 감사 로그 연계 |
| **제외 (Sprint 2 이상)** | Vision AI 분석 연동, NC 시정조치 자동화 패키지, COPQ 확장, 실제 비즈니스 로직 및 UI 구현, DB 스키마 설계 |

## 4. Smart Audit 서버 진입점 개요
**표 3. Smart Audit 서버 진입점 맵**
| 진입점 분류 | 설명 |
|---|---|
| **Session Command Entry** | 세션 생성, 수정, 상태 전이(제출, 재개, 취소) 등 상태를 변경하는 진입점 |
| **Session Query Entry** | 세션 목록 및 단건 상세를 조회하는 진입점 |
| **Report Command Entry** | Submitted 상태의 세션을 바탕으로 리포트 생성을 트리거하는 진입점 |
| **Report Query Entry** | 리포트 조회(최신본, 이력본 등) 진입점 |
| **STT-assisted Input Entry** | 음성 인식 결과를 Session 텍스트 폼에 추가하는 보조 입력 진입점 |
| **Audit/Trace Context Entry** | 헤더의 `x-trace-id`, Tenant 정보를 추출하여 전체 흐름에 Context를 제공하는 진입점 |
| **Error Translation Entry** | 내부 예외를 `API-001` 공통 에러 스키마로 변환하여 반환하는 출구 진입점 |

## 5. Command / Query 분리 기준
**표 4. Command / Query 분리 기준**
| 구분 | 목적 | 권한/상태 영향 | 트랜잭션/로깅 |
|---|---|---|---|
| **Command** | 생성, 수정, 임시저장, 제출, 재개, 종료, 취소, 리포트 생성 | 상태 전이와 권한 영향을 직접 받음. 유효하지 않은 상태 전이는 즉시 409 차단. | 멱등성 보장, 트랜잭션 필요, 변경에 대한 필수 감사 로그 기록 대상. |
| **Query** | 목록, 상세, 상태 기반 조회, 최신본 조회, 이력 조회 | 읽기 중심이므로 상태 전이에 영향 없음. 단, Tenant/Site 범위 통제 필수. | 캐싱 적용 가능, 감사 로그는 선택적(조회 민감도에 따라), 읽기 전용 복제본 라우팅 허용. |

## 6. Audit Session Route Handler 정의
**표 5. Audit Session Route Handler 정의**
| 목적 | 요청 유형 | 선행 검증 | 허용 상태 | 주요 입력 | 주요 출력 | 감사 로그 | 고위험 | 대표 실패 에러 |
|---|---|---|---|---|---|---|---|---|
| 세션 생성 | Command | 권한(`audit_editor` 이상), Tenant 범위 | 해당 없음 | Title, AuditDate, LineId | Session ID, Draft 상태 DTO | 필수 | N | 403 (권한/Tenant 위반) |
| 세션 목록 조회 | Query | 권한(`audit_viewer` 이상), Tenant 범위 | 전체 상태 | Filter, Sort, Pagination 커서 | Session 목록 DTO, Meta | 선택 | N | 400 (잘못된 페이징 파라미터) |
| 세션 상세 조회 | Query | 권한, Tenant 범위, ID 유효성 | 전체 상태 | Session ID | Session 상세 DTO | 필수 | N | 404 (존재하지 않는 세션) |
| 세션 임시저장/수정 | Command | 작성자 본인 또는 `audit_editor`, Tenant | Draft, InProgress | 수정할 필드(STT 포함) 데이터 | 업데이트된 Session DTO | 필수 | N | 409 (Submitted 이후 상태에서 수정 시도) |
| 세션 제출 | Command | `audit_editor` 이상, 필수 필드 누락 검사 | Draft, InProgress | Session ID | Submitted 상태 DTO | 필수 | Y | 409 (필수 데이터 누락, 상태 위반) |
| 세션 재개 | Command | `audit_approver`, Tenant 범위 | Finalized, Closed | Session ID, Reopen 사유 | Reopened 상태 DTO | 필수 | Y | 403 (Approver 권한 없음) |
| 세션 종료/취소 | Command | `audit_creator`, `admin`, Tenant 범위 | Draft, InProgress | Session ID, 취소 사유 | Cancelled 상태 DTO | 필수 | Y | 409 (이미 제출된 세션 취소 시도) |

## 7. Audit Report Route Handler 정의
**표 6. Audit Report Route Handler 정의**
| 목적 | 요청 유형 | 선행 검증 | 허용 상태 | 주요 입력 | 주요 출력 | 감사 로그 | 고위험 | 대표 실패 에러 |
|---|---|---|---|---|---|---|---|---|
| 리포트 생성 요청 | Command | `audit_editor` 이상, 중복 생성 여부 | 원본 세션이 Submitted | Session ID, 옵션(Sprint 1) | Generating 상태 DTO (202 Accepted) | 필수 | Y | 409 (세션 미제출, 중복 생성 시도) |
| 리포트 목록 조회 | Query | `audit_viewer` 이상, Tenant 범위 | Generated, Finalized | Filter, Pagination | 최신본 위주 Report 목록 DTO | 선택 | N | 400 (검증 실패) |
| 리포트 상세 조회 | Query | `audit_viewer` 이상, Tenant 범위 | Generated, Finalized, Superseded | Report ID | Report 상세 DTO | 필수 | N | 404 (리포트 없음), 403 (타 Tenant) |
| 최신본 조회 | Query | `audit_viewer` 이상, Tenant 범위 | Generated, Finalized | Session ID | 해당 세션의 최신 Report DTO | 필수 | N | 404 (생성된 리포트 없음) |
| 이력본 조회 | Query | `audit_viewer` 이상, Tenant 범위 | 전체 상태 | Report/Session ID | Superseded 포함 이력 목록 DTO | 필수 | N | 404 (세션 없음) |

## 8. 요청 처리 공통 흐름
**표 7. 요청 처리 공통 흐름**
| 단계 | 책임 설명 |
|---|---|
| **1. 요청 수신** | HTTP URL 파라미터, 쿼리 스트링, Request Body를 파싱한다. 헤더에서 `x-trace-id`를 추출(없으면 생성)한다. |
| **2. 인증 확인** | JWT 유효성 및 만료 여부를 검사한다. 실패 시 `401 Unauthorized` 반환. |
| **3. 역할/권한 판별** | 인증된 토큰에서 사용자의 Role(System Admin, Tenant Admin, Auditor 등)을 판별한다. |
| **4. Tenant/Site 확인** | URL 경로 또는 Body에 명시된 Tenant/Site ID가 사용자 Context와 일치하는지 교차 검증한다. 불일치 시 `403 Forbidden`. |
| **5. 파라미터 검증** | Zod 스키마를 사용하여 데이터 타입, 길이, 필수 여부를 검사한다. 실패 시 `400 Validation Error`. |
| **6. 상태 허용 확인** | Command 요청 시 대상 리소스의 DB 상태가 해당 액션을 허용하는지 검사한다. 실패 시 `409 State Conflict`. |
| **7. DTO/모델 매핑** | 검증을 통과한 파라미터를 내부 서비스 계층이 이해할 수 있는 요청 모델(Request DTO)로 매핑한다. |
| **8. 서비스 위임** | 도메인 로직, DB 트랜잭션을 처리하는 Service 계층 메서드를 호출한다. |
| **9. 결과 매핑** | Service의 결과물을 API 계약에 맞는 응답 DTO로 매핑한다. |
| **10. 추적/감사 연결**| 성공/실패 여부를 Audit Log 이벤트로 전송하고, Response Meta에 `trace_id`를 주입한다. |
| **11. 응답 반환** | 성공 2xx 반환, 실패 시 `API-001` 공통 에러 스키마로 변환하여 반환한다. |

## 9. 인증 / 권한 / tenant / site 범위 검증 기준
**표 8. 인증 / 권한 / tenant / site 검증 기준**
| 검증 항목 | 검증 기준 및 차단 규칙 |
|---|---|
| **인증(AuthN)** | 모든 엔드포인트는 JWT Bearer 검사 필수. (만료/조작 시 `401 Unauthorized`) |
| **권한(AuthZ)** | 역할(Role) 기반 통제. System/Tenant Admin은 모든 권한, Approver는 Reopen/Finalize 특권, Editor는 생성/수정/제출, Viewer/Observer는 읽기 전용. 부족 시 `403 Forbidden`. |
| **Tenant 격리** | 타 Tenant 리소스 접근 원칙적 차단. 요청 URL의 Workspace ID와 JWT의 Tenant ID 불일치 시 `403 Forbidden`. |
| **Site 범위 통제** | 사용자가 속한 Site(공장/라인) 데이터만 변경 가능. 조회는 Tenant 내에서 허용되더라도 변경(Command)은 Site 검증 필수. |
| **민감 정보 제한** | 읽기 권한(Viewer)이 있어도 시스템 내부 추적용 메타데이터나 관리자 전용 메모는 DTO 매핑 단계에서 필터링(제외)하여 노출을 제한한다. |

## 10. 입력 검증 및 DTO 매핑 기준
**표 9. 입력 검증 및 DTO 매핑 기준**
| 검증 대상 | 처리 기준 |
|---|---|
| **필수값 누락/형식** | Zod 스키마로 강제 검증. UUID 형식, ISO8601 날짜 형식 위반 시 즉시 400 반환. |
| **상태/전이 불일치** | 세션 제출(Submit) 요청 시 Findings 데이터가 비어있으면 409 반환. |
| **식별자 유효성** | Session/Report ID가 경로 파라미터와 Body에 중복 존재 시 일치 여부 확인. |
| **Tenant/Site Context** | 클라이언트가 Body에 임의의 Tenant ID를 주입해도 무시(Strip)하고 JWT에서 추출한 Context로 덮어씌움. |
| **STT 보조 입력** | 자동 생성된 STT 텍스트와 수동 수정 텍스트를 DTO 내에서 분리하여 수신할 수 있도록 매핑. |
| **API-003 충돌 방지** | Request 파라미터는 `API-003_audit_report_dto.md`의 Request DTO 스펙을 100% 준수하여 파싱. |

## 11. 상태 기반 허용 / 차단 규칙
**표 10. 상태 기반 허용 / 차단 규칙**
| 타겟 자원 상태 | 허용되는 동작 | 차단되는 동작 및 응답 코드 |
|---|---|---|
| **Draft / InProgress** | 조회, 수정, STT 텍스트 추가, 제출, 취소 | 리포트 생성 시도 차단 (`409 Conflict`) |
| **Submitted / Finalized** | 조회, (Approver에 의한) 재개, 리포트 생성/확정 | 세션 본문 수정 차단 (`409 Conflict`) |
| **Reopened** | 조회, 추가 수정, 재제출 | 기존 Finalized 리포트의 공식 상태 유지 차단 (재제출 시 기존 리포트는 Superseded 전이 대기) |
| **Generated (Report)** | 조회, (Approver에 의한) 확정, Superseded 전이 | 새로운 중복 리포트 생성 (`409 Conflict`) |
| **Superseded (Report)**| 이력 기반 조회 | 모든 변경/확정 동작 (`409 Conflict`), 삭제가 아닌 보존 상태로 유지 |
| **Closed / Cancelled** | 단순 조회 (이력 확인용) | 수정, 제출, 리포트 생성 등 모든 Command 동작 차단 |

## 12. 응답 반환 기준
**표 11. 응답 반환 기준**
| 응답 종류 | HTTP Status | 본문 및 헤더 특징 |
|---|---|---|
| **생성/수정/제출/재개 성공 (Command)** | `200 OK` 또는 `201 Created` | 동기 처리 완료 후 변경된 최종 상태 DTO 반환 |
| **종료/취소 성공** | `200 OK` | 상태가 Cancelled로 변경된 Summary DTO 반환 |
| **목록/상세/최신본/이력 조회 성공 (Query)** | `200 OK` | 페이징 메타데이터(`meta`) 포함 DTO 반환 |
| **리포트 생성 진행중 응답** | `202 Accepted` | 비동기 생성 큐에 인입되었음을 알리며, 상태는 `Generating`으로 표기 |
| **표준 실패 응답** | `4xx` 또는 `5xx` | `API-001`에 따른 표준 에러 객체 반환 (`code`, `message`, `traceId`) |
| **응답 일관성 공통** | - | 모든 응답 메타에 `trace_id`, `timestamp` 포함. UI와 TEST가 파싱하기 쉬운 일관된 형태 유지. |

## 13. 오류 처리 및 공통 에러 스키마 연계
**표 12. 오류 처리 및 공통 에러 연계**
| 시나리오 | 에러 코드 | 내부 처리 / 사용자 메시지 예시 |
|---|---|---|
| **인증 만료** | `401` | "토큰이 만료되었습니다. 다시 로그인해주세요." |
| **권한/Site 위반** | `403` | "해당 기능을 실행할 권한이 없습니다." / 내부 추적: Role 부족 |
| **Tenant 불일치** | `403 / 404` | 원칙상 404 처리하여 리소스 존재 여부 자체를 은닉 (보안 상 403 혼용 가능) |
| **리소스 부재** | `404` | "요청하신 세션/리포트를 찾을 수 없습니다." |
| **상태 불일치/전이 불가** | `409` | "현재 상태(Draft)에서는 리포트를 생성할 수 없습니다." |
| **중복 생성 요청** | `409` | "이미 생성 중이거나 완료된 리포트가 존재합니다." |
| **타임아웃/의존 실패** | `503 / 504` | "외부 시스템(STT 등) 연동 지연으로 실패했습니다." |
| **내부 서버 오류** | `500` | "서버 내부 오류가 발생했습니다. (Trace ID: X)" |

## 14. 감사 로그 및 보안 연계
**표 13. 감사 로그 및 보안 연계 기준**
| 로그 항목 | 연계 기준 및 원칙 |
|---|---|
| **Write 이벤트 기록** | 세션 생성, 수정, 제출, 재개, 취소 및 리포트 생성 트리거는 발생 즉시 Audit Log Queue로 발송. |
| **Read 이벤트 기록** | 민감한 이력 조회, 권한 없는 자의 반복 조회 시도는 보안성 이벤트로 분류하여 로깅. |
| **실패 요청 로깅** | 401, 403, 409(비정상적 상태 전이 시도) 에러는 보안 공격 가능성이 있으므로 반드시 로깅 대상 포함. |
| **Context 연계** | 모든 로그 데이터는 `trace_id`, `actor_id`, `tenant_id`, `site_id`, `session_id`, `report_id`를 포함해야 함. |
| **데이터 최소화** | 본문 내용 변경은 Delta(차이점) 위주로 저장하며, 불필요한 전체 Payload 스냅샷은 최소화하여 PIPA 준수. |

## 15. 성능 / 멱등성 / 일관성 고려사항
**표 14. 성능 / 멱등성 / 일관성 기준**
| 항목 | 고려사항 및 기준 |
|---|---|
| **목록 조회 페이징** | 대용량 데이터 대응을 위해 커서 기반 페이지네이션(Cursor Pagination)을 1순위로 고려한다. |
| **멱등성(Idempotency)** | 리포트 생성, 세션 제출 등 주요 Command는 헤더의 `Idempotency-Key`를 확인하여 중복 처리를 방지한다. |
| **Generating 상태 재요청** | 이미 `Generating` 상태인 세션에 대해 다시 생성 요청 시, 생성 큐를 확인하여 중복 삽입을 막고 즉시 409를 반환한다. |
| **응답 일관성** | 프론트엔드(UI)의 SWR/React-Query, QA(TEST/MOCK) 검증 스크립트가 동일한 규격을 소비하도록 데이터 뎁스와 필드명을 고정한다. |

## 16. API / UI / TEST / MOCK 영향 및 후속 문서 연결 기준
**표 16. 후속 문서 연결 기준**
| 연결 문서 | 역할 및 영향 |
|---|---|
| **API-003_audit_report_dto.md** | 본 명세서에서 추출하고 변환하는 데이터의 구체적인 형태와 타입을 제공하는 **직접적인 계약서**. |
| **UI-010_audit_workspace_page.md** | RH가 반환하는 상태 코드(409 등)와 Error Message를 바탕으로 사용자에게 경고창/버튼 비활성화를 처리함. |
| **TEST-F1-001/002_xxx.md** | 테스트 코드는 본 문서의 `상태 기반 차단 규칙`과 `권한/Tenant 검증`이 의도대로 동작하여 4xx/5xx를 내뿜는지 Assert함. |
| **MOCK-002_audit_report_mock.md** | 프론트엔드 작업용 서버는 본 문서의 응답 반환 기준과 에러 매핑 규칙을 모방하여 동작함. |

## 17. Sprint 우선순위
**표 15. Sprint 우선순위 정리**
| 작업 단위 | Sprint | 우선순위 | 비고 |
|---|---|---|---|
| 세션 CRUD 및 상태 전이 입구 | Sprint 1 | High (P0) | F1 핵심 (STT 연계 포함) |
| Tenant 격리 및 RBAC Guard | Sprint 1 | High (P0) | 보안 치명결함 방지 |
| 리포트 단건 생성 및 최신본 조회 | Sprint 1 | High (P0) | 핵심 결과물 확인 목적 |
| 리포트 이력(Superseded) 조회 | Sprint 1 | Medium (P1) | 변경 이력 추적을 위한 기초 |
| Vision AI/NC 패키지 확장 진입점 | Sprint 2 | Low (P3) | 차기 스프린트로 이관 |

## 18. 완료 기준 (Definition of Done)
- [ ] 본 문서에 명시된 모든 진입점(Entry)에 대응하는 Route Handler 함수 서명이 정의되었는가?
- [ ] JWT / RBAC / Tenant 판별 Guard가 공통 흐름(Middleware) 수준에 적용되었는가?
- [ ] 상태 충돌 시나리오 및 에러 매핑이 `API-001`을 준수하여 100% 반영되었는가?
- [ ] 감사 로그 트리거(Context 파싱 로직 포함)가 Route Handler 내에 위치하는가?

## 19. 다음 단계 작성 가이드
서버의 진입점(Route Handler) 동작 기준이 확립되었으므로, 이어지는 `API-003_audit_report_dto.md`에서 각 라우트 핸들러가 주고받을 정확한 데이터 스키마와 구조(Contract)를 정의한다. 

## 20. Gemini 3.1 Pro 자체 검토 체크리스트
- [x] Route Handler가 비즈니스 로직과 경계(입구)로 명확히 분리되었는가?
- [x] Command와 Query의 취급 방식이 명시적으로 분리되었는가?
- [x] 상태, 권한, Tenant 기반의 차단(Fail-Fast) 규칙이 세밀하게 정의되었는가?
- [x] 공통 에러 매핑 및 감사 로그 연계 방안이 누락 없이 기술되었는가?
- [x] 타 문서(F1-C-001/002, UI/TEST/MOCK)와의 연결성과 영향도가 설명되었는가?
