# TEST-F1-002_audit_report_generation_test.md

## 1. 문서 목적
본 문서는 PRO ILI SMART 플랫폼의 Smart Audit F1 라인 중 '감사 리포트(Audit Report)'의 생성 규칙, 중복 방지, 이력(버전) 처리 및 예외 상황에 대한 검증 기준을 정의한다. 이를 통해 리포트가 허용된 상태의 세션에서만 생성되고, 데이터 무결성이 보장되며, Reopen 등 변경 발생 시 올바르게 상태가 전이되는지(Superseded 처리) 확인한다.

## 2. 테스트 설계 원칙
**표 1. 테스트 설계 원칙**

| 원칙 | 설명 | 적용 기준 |
| :--- | :--- | :--- |
| **상태 의존성 강제** | 세션(Session)의 상태에 종속적인 리포트 생성 권한 검증 | 세션 상태 불일치 시 HTTP 409 반환 필수 |
| **멱등성 및 중복 제어** | 동일 세션에 대한 동시 리포트 생성 요청 시 1건만 성공하고 나머지는 차단 | DB Lock 또는 Unique Constraint 검증 |
| **버전 무결성 (Version Control)** | Reopen 후 재제출 시 기존 리포트는 읽기 전용의 '과거본(Superseded)'으로 강제 격리 | 목록 조회 시 최신본과 이력본 명확히 구분 |
| **외부 의존성 격리(향후 대비)** | (Sprint 1 수동 생성 기준) DB 트랜잭션 무결성 최우선 검증 | 데이터 누락, 타임아웃 상황 모사 |

## 3. 적용 범위 정의
**표 2. 적용 범위 정의**

| 범위 | 포함 대상 | 제외 대상 (Sprint 2+) |
| :--- | :--- | :--- |
| **테스트 대상 (Sprint 1)** | 리포트 생성 요청(수동 트리거), 생성 가능/차단 상태 검증, 리포트 상태 전이(Generated, Superseded), 단건/목록 조회 | Vision AI 분석 결과 포함 검증, PDF 다운로드 성능 |
| **검증 영역** | 세션-리포트 정합성, 권한 통제, 에러 스키마(`API-001`), 감사 로그(`trace_id`) | 자동 예약 발송 메일 검증 |

## 4. 전제 조건
**표 3. 전제 조건**

| 항목 | 상세 내용 |
| :--- | :--- |
| **기초 데이터** | 각 상태별 세션 사전 준비 (Draft 1건, Submitted 2건, Closed 1건, Cancelled 1건) |
| **인증 정보** | 권한별 토큰 준비 (Auditor, Viewer, 타 Site 계정) |

## 5. 리포트 생성 가능 상태 검증
**표 4. 생성 가능 상태 검증**

| TC ID | 대상 세션 상태 | 테스트 액션 | 기대 결과 |
| :--- | :--- | :--- | :--- |
| REP-V-01 | Submitted | [리포트 생성] API 호출 | HTTP 201, 상태: `Generated`, Session ID 매핑 확인 |
| REP-V-02 | Finalized | [리포트 생성] API 호출 | HTTP 201, 상태: `Generated` |
| REP-V-03 | Closed | [리포트 생성] API 호출 | HTTP 409 Conflict (종료된 세션은 신규 리포트 생성 불가) |

## 6. 리포트 생성 차단 상태 검증
**표 5. 생성 차단 상태 검증**

| TC ID | 대상 세션 상태 | 테스트 액션 | 기대 결과 |
| :--- | :--- | :--- | :--- |
| REP-I-01 | Draft | [리포트 생성] API 호출 | HTTP 409 Conflict, "Session is not submitted" 에러 |
| REP-I-02 | InProgress | [리포트 생성] API 호출 | HTTP 409 Conflict |
| REP-I-03 | Cancelled | [리포트 생성] API 호출 | HTTP 409 Conflict |

## 7. Generated / Superseded / 최신본 / 이력본 검증
**표 6. Generated / Superseded 검증**

| TC ID | 시나리오 | 테스트 액션 | 기대 결과 |
| :--- | :--- | :--- | :--- |
| REP-S-01 | 최신본 조회 | 최신 생성된 리포트 ID로 GET 호출 | HTTP 200, is_latest: true, 상태: `Generated` |
| REP-S-02 | 이력본 조회 | 과거 Reopen 이전의 리포트 ID로 GET 호출 | HTTP 200, is_latest: false, 상태: `Superseded` |

## 8. 중복 생성 방지 검증
**표 7. 중복 생성 방지 검증**

| TC ID | 시나리오 | 테스트 액션 | 기대 결과 |
| :--- | :--- | :--- | :--- |
| REP-D-01 | 동시 다발적 생성 요청 | 동일 Session ID로 리포트 생성 API를 동시에 5회 비동기 호출 | 1개 요청만 201 Created, 나머지 4개는 409 Conflict 반환 |
| REP-D-02 | 이미 존재하는 최신본에서 재요청 | `Generated` 리포트가 존재하는 세션에서 재요청 | HTTP 409 Conflict, "Latest report already exists" 에러 |

## 9. Reopen 이후 버전 처리 검증
**표 8. Reopen 이후 버전 처리 검증**

| TC ID | 시나리오 | 테스트 액션 단계 | 기대 결과 |
| :--- | :--- | :--- | :--- |
| REP-R-01 | Reopen 수행 및 과거본 격리 | 1. Submitted 세션에서 리포트(R1) 생성<br>2. 해당 세션 Reopen 호출<br>3. R1 상태 조회 | 3단계 수행 시 R1 상태가 `Generated` -> `Superseded`로 자동 전이됨 확인 |
| REP-R-02 | 재제출 후 신규 리포트 생성 | 1. 위 1~2단계 수행<br>2. 세션 다시 Submitted<br>3. 신규 리포트(R2) 생성<br>4. 세션의 리포트 목록 조회 | 2개의 리포트 반환 (R1: Superseded, R2: Generated/is_latest=true) |

## 10. 권한 / tenant / site 검증
**표 9. 권한 / tenant / site 검증**

| TC ID | 테스트 관점 | 입력값/조작 | 기대 결과 |
| :--- | :--- | :--- | :--- |
| REP-A-01 | 권한 부족 (Viewer) 생성 | Viewer 계정으로 생성 API 호출 | HTTP 403 Forbidden |
| REP-A-02 | 타 Tenant 리포트 조회 | 타 Tenant의 Report ID로 GET 호출 | HTTP 404 Not Found (격리) |

## 11. 에러 시나리오 검증
**표 10. 에러 시나리오 검증**

| TC ID | 시나리오 | 입력값/조작 | 기대 결과 (`API-001` 준수) |
| :--- | :--- | :--- | :--- |
| REP-E-01 | 존재하지 않는 세션 ID | 무효한 session_id로 생성 요청 | HTTP 404, code: "RESOURCE_NOT_FOUND" |
| REP-E-02 | 데이터 생성 타임아웃 모사 | (의도적 지연 주입) | HTTP 504 또는 500, code: "INTERNAL_SERVER_ERROR" |

## 12. 감사 로그 및 trace_id 연계 검증
**표 11. 감사 로그 및 trace_id 연계 검증**

| 액션 | 확인 대상 필드 | 기대 결과 (DB 검증) |
| :--- | :--- | :--- |
| **생성 완료** | `Audit_Log` / `trace_id` | action_type: 'REPORT_GENERATED', target_id: report_id, trace_id 헤더값 일치 |
| **Superseded 전이**| `Audit_Log` | action_type: 'REPORT_SUPERSEDED' 로그 자동 적재 확인 (Reopen 이벤트 시 트리거) |

## 13. 증빙 자료 기준
**표 12. 증빙 자료 기준**
- 테스트 자동화 리포트 (Jest/Postman 결과)
- REP-R-01, REP-R-02 (Reopen 후 버전 처리) 시나리오의 단계별 DB 데이터 상태 캡처본
- 409 Conflict 발생 시의 에러 응답 본문 로그 (trace_id 포함)

## 14. 결함 심각도 기준
**표 13. 결함 심각도 기준**

| 심각도 | 조건 | 조치 기한 |
| :--- | :--- | :--- |
| **Critical** | Draft/InProgress 세션에서 리포트 생성됨, 타 Tenant 리포트 접근 허용, 동시 요청 시 중복 생성됨 | 즉시 (배포 불가) |
| **High** | Reopen 수행 시 기존 리포트가 Superseded 처리되지 않음 (데이터 무결성 위반) | Sprint 내 해결 |
| **Medium** | 에러 상세(details) 누락, 정렬 기준(생성일시 역순) 불일치 | 차기 핫픽스 |

## 15. Sprint 우선순위
**표 14. Sprint 우선순위 정리**

| 항목 | 우선순위 | 비고 |
| :--- | :--- | :--- |
| 생성 가능/차단 상태 검증 (REP-V, REP-I) | P0 (필수) | 상태 머신 무결성 직결 |
| Reopen 버전 처리 (REP-R) | P0 (필수) | 이력 관리 핵심 기능 |
| 동시 생성 방지 (REP-D) | P1 | DB Lock/Constraint 검증 필요 |

## 16. 완료 기준 (Definition of Done)
**표 15. 완료 기준**

| 완료 조건 | 확인 방법 |
| :--- | :--- |
| 전체 테스트 케이스 작성 및 통과 | 자동화 도구 실행 결과 All Pass 확인 |
| API-001 스키마 100% 매핑 | 모든 예외 케이스에서 표준 에러 응답 반환 확인 |
| Reopen에 따른 Superseded 격리 보장 | DB 쿼리 결과 과거 리포트의 `is_latest` 값이 false 로 업데이트됨 확인 |

## 17. 다음 단계 작성 가이드
- QA 팀은 표 8 (REP-R 시나리오) 등 다단계 상태 전이 검증을 위해, 단일 요청 툴 대신 E2E 또는 시나리오 기반 테스트 스크립트 작성에 착수한다.

## 18. Gemini 3.1 Pro 자체 검토 체크리스트
- [x] 리포트 생성 가능 세션 상태(Submitted/Finalized)가 명확히 통제되는가?
- [x] Reopen 시 기존 리포트를 과거본(Superseded)으로 격리하는 버전 관리 시나리오가 검증되는가?
- [x] 동시성 문제(중복 생성) 차단에 대한 검증 관점이 반영되었는가?
- [x] 권한 제어 및 공통 에러 스키마 준수가 명시되었는가?
