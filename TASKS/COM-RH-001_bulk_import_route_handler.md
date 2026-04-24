# [COM-RH-001] Bulk Import Route Handler 처리 명세

## 1. 문서 목적

*   **문서 필요성:** Bulk Import 관리자 기능의 서버 진입점(Route Handler)에서 수행해야 하는 인증, 권한, 검증, DTO 매핑, 응답, 오류 처리, 감사 로그 기록의 표준화된 흐름을 정의한다.
*   **Sprint 1 중요성:** Bulk Import는 Sprint 1 운영 착수 가능성을 좌우하는 핵심 기능이다. 견고한 서버 입구(Boundary)를 구축하여, 오염된 데이터 유입을 막고 일관된 응답과 운영 추적성을 보장한다.
*   **후속 문서와의 관계:** 본 문서는 정의된 API 규격(`API-002`)과 관리자 비즈니스 규칙(`ADM-C/Q-001`), 그리고 프론트엔드(`UI-061`) 및 테스트(`TEST-ADM-001`) 문서를 연결하는 실질적 서버 구현 지침서이다. 향후 작성될 Mock(`MOCK-001`)과 모니터링(`NFR-MON-001`) 설계의 기준이 된다.

---

## 2. Route Handler 설계 원칙

입구로서의 책임을 명확히 하고, 인증/권한, 일관성, 운영 추적성을 최우선으로 설계한다.

### 표 1. Route Handler 설계 원칙
| 원칙 | 설명 | 적용 이유 | 관련 문서 | Sprint 우선순위 |
| :--- | :--- | :--- | :--- | :--- |
| 책임 분리 (Thin Handler) | 라우트 핸들러는 비즈니스 로직을 포함하지 않고, 검증/매핑/라우팅에 집중함 | 코드 유지보수성 향상 및 비즈니스 계층 독립성 보장 | `05_SRS_v1` | P1 (Sprint 1) |
| 인증 선행 (Fail Fast) | 비즈니스 로직 호출 전 가장 먼저 토큰 및 세션 유효성을 검증함 | 불필요한 시스템 리소스 낭비 및 잠재적 보안 위협 사전 차단 | `COM-AUTH_v1` | P1 (Sprint 1) |
| tenant/site 범위 보호 | 토큰에서 추출한 정보와 요청 파라미터를 대조하여 횡적 이동 원천 차단 | 엔터프라이즈 SaaS 멀티테넌시 데이터 격리의 핵심 | `COM-AUTH_v1` | P1 (Sprint 1) |
| 공통 에러 스키마 일관성 | 모든 예외 및 오류는 약속된 공통 에러 구조로 변환하여 반환 | 클라이언트(UI) 처리 일관성 보장 및 에러 원인 식별 용이성 | `API-001` | P1 (Sprint 1) |
| 상태 변경 추적 (Audit) | 데이터 상태를 변경하는 모든 요청(성공/실패)을 Audit Log로 남김 | 보안 락업 및 책임 추적성 확보 | `DATA-AUDIT_LOG_v1` | P1 (Sprint 1) |

---

## 3. 적용 범위 정의

Sprint 1 핵심 핸들러 위주로 In-Scope를 정의하며, 오케스트레이션 및 과도한 추상화는 Out-of-Scope로 둔다.

### 표 2. 적용 범위 정의
| 핸들러 영역 | 설명 | Sprint 적용 시점 | 필수 여부 | 비고 |
| :--- | :--- | :--- | :--- | :--- |
| Command (생성/취소/재처리) | 상태를 변경하는 고위험/쓰기 작업 라우팅 및 검증 | Sprint 1 | Y | Audit Log 필수 |
| Query (목록/상세/오류) | 상태를 변경하지 않는 읽기 작업 라우팅 및 검증 | Sprint 1 | Y | 페이징/필터링 입력 검증 포함 |
| 통합 BFF 계층 분리 | 모바일/웹 등 다중 채널별 응답 최적화를 위한 BFF 계층 | Sprint 3 이후 | N | 현재는 단일 API 구조 유지 |
| 서드파티 웹훅 수신 핸들러 | 외부 시스템 연동용 커스텀 벌크 임포트 수신 | Sprint 4 이후 | N | Sprint 1 범위 외 |

---

## 4. Bulk Import 서버 진입점 맵

### 표 3. Bulk Import 서버 진입점 맵
| 핸들러/진입점 | 성격(Command/Query) | 설명 | 입력 | 출력 | 비고 |
| :--- | :--- | :--- | :--- | :--- | :--- |
| `POST /api/v1/bulk-imports` | Command | 신규 Bulk Import Job 생성 및 업로드 처리 시작 | Multipart Form Data, 메타데이터 | Job ID, 초기 상태 (202 Accepted) | API-002 준수 |
| `GET /api/v1/bulk-imports` | Query | 조건에 맞는 Job 목록 페이징 조회 | 쿼리 파라미터(기간, 상태 등) | Job 목록 배열, 페이징 메타 | ADM-Q-001 연계 |
| `GET /api/v1/bulk-imports/:jobId` | Query | 특정 Job의 상세 정보 및 처리 통계 조회 | 경로 변수(jobId) | Job 상세 정보 (DTO) | 권한 및 경계 검사 필수 |
| `GET /api/v1/bulk-imports/:jobId/failures` | Query | 부분 성공/실패 Job의 실패 행 단위 상세 정보 조회 | 경로 변수(jobId), 페이징 파라미터 | 에러 상세 배열, 페이징 메타 | UI 드릴다운용 |
| `POST /api/v1/bulk-imports/:jobId/retry` | Command | 실패한 건에 대한 수정된 데이터로 재처리 요청 | 경로 변수(jobId), Multipart Form Data | 신규 Job ID, 연관 메타 (202 Accepted) | 원본 Job 연결 |
| `POST /api/v1/bulk-imports/:jobId/cancel` | Command | 진행 중인 Job의 강제 중단 | 경로 변수(jobId), 취소 사유 | 취소 처리 상태 (200 OK) | 취소 가능 상태 확인 |
| `GET /api/v1/bulk-imports/summary` | Query | 대시보드 표시를 위한 요약 통계 조회 (선택적) | 쿼리 파라미터(기간, Site 등) | 총 건수, 상태별 비율 통계 | UI-061 지원 |

---

## 5. Command vs Query 분리 기준

요청의 성격에 따라 라우팅, 검증 강도, 감사 요구사항이 완전히 달라진다.

### 표 4. Command vs Query 분리 기준
| 구분 | 설명 | 대표 핸들러 | 검증 포인트 | 감사 로그 중요도 | 비고 |
| :--- | :--- | :--- | :--- | :--- | :--- |
| **Command** | 시스템의 상태를 변경하는 쓰기 액션 | 생성(업로드), 재처리, 취소 | 형식/비즈니스 룰 위반, 권한, 동시성 | **매우 높음** (성공/실패 모두 기록) | 트랜잭션 및 비동기 워커 연계 필수 |
| **Query** | 데이터를 반환하는 읽기 액션 | 목록 조회, 상세 조회, 에러 내역 조회 | 접근 권한, 페이징 규격, 악의적 파라미터 방어 | 보통 (인가 실패 및 민감 데이터 조회 시 기록) | 캐싱 전략 (선택적) 적용 가능 |

---

## 6. 요청 처리 공통 흐름

### 표 5. 요청 처리 공통 흐름
| 단계 | 설명 | 입력 | 처리 | 출력 | 실패 시 대응 | 비고 |
| :--- | :--- | :--- | :--- | :--- | :--- | :--- |
| **1. 요청 수신** | 클라이언트로부터 HTTP 요청 수신 | HTTP Request | 라우터 진입 | Request 객체 전달 | 400 (형식 오류) | |
| **2. 인증 확인** | 토큰 유효성 및 만료 여부 확인 | Header Bearer Token | JWT 파싱 및 세션 검증 | Actor Context | 401 Unauthorized | COM-AUTH_v1 |
| **3. 권한 확인** | 해당 엔드포인트 접근 역할 확인 | Actor Context | Role(System/Tenant/Site) 대조 | 통과/차단 | 403 Forbidden | Audit Log (인가 실패) |
| **4. 범위 확인** | 요청 데이터가 허용 테넌트/사이트 범위 내인지 확인 | Actor Context, 파라미터 | tenant_id, site_id 교차 검증 | 통과/차단 | 403 Forbidden | 테넌트 격리 핵심 |
| **5. 파라미터 검증** | Zod 등을 활용한 구조 및 타입 유효성 검사 | 파라미터, Body | 스키마 기반 유효성 확인 | 정제된 Typed DTO | 400 Bad Request | API-001 (Validation Error) |
| **6. 서비스 호출** | 핵심 비즈니스 로직(Service/Usecase) 계층으로 위임 | Typed DTO | 의존성 주입된 서비스 메서드 실행 | 비즈니스 처리 결과 | 409 Conflict, 500 등 | 서비스 계층 내 에러 throw |
| **7. 응답 변환** | 서비스 결과를 API 규격에 맞는 DTO로 맵핑 | 비즈니스 결과 | Response DTO 변환, 민감정보 제거 | 최종 응답 DTO | (포맷팅 에러) 500 | |
| **8. 에러 매핑** | 처리 중 발생한 예외를 공통 규격으로 포장 | 내부 Exception | Error Mapper 실행 (코드/메시지 매핑) | API-001 스키마 | - | trace_id 포함 |
| **9. 감사 기록** | 최종 결과를 기반으로 Audit Log 전송 | Actor, Job ID, Status | Audit Event 발행 (Insert) | (비동기 로깅 완료) | 로그 누락 방지(Fallback) | DATA-AUDIT_LOG |

---

## 7. 엔드포인트/핸들러 단위 정의

### 표 6. 핸들러 단위 정의
| 핸들러명 | 책임 | 주요 입력 DTO | 주요 출력 DTO | 권한 요구 | 비고 |
| :--- | :--- | :--- | :--- | :--- | :--- |
| `createBulkImportJob` | 파일 검증, S3/스토리지 임시 저장, 비동기 Job 생성 | `Multipart`, `tenant_id`, `site_id`, `import_type` | `CreateJobResponse` (job_id, status) | Admin, Manager | 비동기 워커 큐 연동 202 응답 |
| `getBulkImportJobList` | 권한 범위 내 Job 목록 필터링 조회 | `ListQueryDto` (page, limit, status) | `JobListResponse` (items, metadata) | Admin, Manager | 테넌트/사이트 ID 강제 주입 |
| `getBulkImportJobDetail` | 단일 Job 상세 통계 및 메타 반환 | `jobId` (경로변수) | `JobDetailResponse` | 해당 Job 소유 권한 | 접근 권한 철저 확인 |
| `getBulkImportJobFailures` | 부분 성공 Job의 행별 오류 내역 페이징 제공 | `jobId`, `page`, `limit` | `FailureListResponse` (errors 배열) | 해당 Job 소유 권한 | 에러 로우 대량 조회 시 성능 고려 |
| `retryBulkImportJob` | 실패 건 수정 데이터 병합 및 신규 Job 할당 | `jobId`, `Multipart` | `CreateJobResponse` (신규 job_id) | Admin, Manager | 원본 Job의 `status` 검증 필수 |
| `cancelBulkImportJob` | PENDING / IN_PROGRESS Job 강제 중단 | `jobId`, `reason` | `ActionResponse` (success: boolean) | System Admin, 생성자 | 취소 불가 상태 시 400 반환 |

---

## 8. 인증/권한/범위 검증 기준

### 표 7. 인증/권한/범위 검증 기준
| 검증 항목 | 설명 | 적용 대상 핸들러 | 실패 시 응답 방향 | 비고 |
| :--- | :--- | :--- | :--- | :--- |
| 인증 (Authentication) | 토큰 누락, 만료, 변조 검사 | 전체 공통 | 401 Unauthorized | API Gateway 또는 미들웨어 위임 권장 |
| 역할 (Role Authorization) | 사이트 운영자 이상의 권한 확인 | 전체 (상세 수준 차등) | 403 Forbidden | `message`: "접근 권한이 없습니다." |
| 테넌트/사이트 범위 보호 (Tenant Isolation) | 토큰 내 `tenant_id`와 조회하려는 Job의 `tenant_id` 일치 확인 | 조회 및 처리 (Query & Command) | 403 또는 404 (존재 숨김) | 타 테넌트 데이터는 404로 응답하여 식별자 스캐닝 방어 권장 |
| 소유권 (Ownership) | 재처리/취소는 관리자 또는 해당 Job 생성자만 허용 | `retry`, `cancel` | 403 Forbidden | |

---

## 9. 입력 검증 및 DTO 매핑 기준

### 표 8. 입력 검증 및 DTO 매핑 기준
| 입력 유형 | 검증 항목 | DTO 반영 방식 | 오류 처리 방식 | 비고 |
| :--- | :--- | :--- | :--- | :--- |
| 경로 파라미터 | `jobId` UUID v4 형식 여부 확인 | 정규화하여 DTO 속성 매핑 | 400 Bad Request (Invalid ID Format) | 불필요한 DB 조회 방지 |
| 쿼리 파라미터 | 페이징(`page`, `limit`), 날짜 형식 정합성 확인 | 숫자형 및 Date 객체 변환 (`ListQueryDto`) | 기본값(Fallback) 사용 또는 400 반환 | 최대 limit (예: 100) 강제 적용 |
| Multipart Form | 파일 확장자(.csv, .xlsx), 최대 용량, 필수 메타(site_id) | 바이너리 스트림 분리 및 메타 `CreateJobDto` 매핑 | 400 또는 413 (Payload Too Large) | 클라이언트 조작 대비 엄격 검사 |
| 보안 컨텍스트 주입 | 클라이언트 파라미터를 신뢰하지 않고, 토큰 정보 덮어쓰기 | `tenant_id`, `actor_id`를 DTO에 강제 주입 | - | **보안 락업 필수 항목** |

---

## 10. 응답 반환 기준

### 표 9. 응답 반환 기준
| 응답 유형 | 설명 | 포함 DTO | 공통 메타 포함 여부 | 비고 |
| :--- | :--- | :--- | :--- | :--- |
| 비동기 수락 응답 | 생성, 재처리 시 즉각 반환 (202 Accepted) | `job_id`, `status` (PENDING) | Y (trace_id 등) | Job ID로 폴링/웹소켓 추적 유도 |
| 목록 응답 | 페이징 처리된 목록 반환 (200 OK) | `items` 배열, `pagination` (total, page 등) | Y | |
| 빈 결과 목록 응답 | 검색 조건에 맞는 Job이 없을 때 반환 | 빈 `items` 배열 (`[]`), total: 0 | Y | 404가 아닌 빈 배열 응답이 표준 |
| 액션 완료 응답 | 취소 성공 시 반환 (200 OK) | `success`, `message`, 갱신된 `status` | Y | 상태 전이 결과 명확히 반환 |
| 민감 필드 마스킹 | 응답 DTO 변환 시 내부 물리적 ID, 스택트레이스 등 제외 | 최종 Response DTO | N/A | DTO Mapper의 핵심 역할 |

---

## 11. 오류 처리 및 에러 스키마 연계

### 표 10. 오류 처리 및 에러 스키마 연계
| 오류 유형 | 설명 | 반환 방식 | 공통 에러 코드 방향 | trace_id 포함 여부 | 비고 |
| :--- | :--- | :--- | :--- | :--- | :--- |
| 검증 오류 (Validation) | 파라미터, 바디 스키마 불일치 (Zod Error 등) | 400 Bad Request | `ERR_VALIDATION_FAILED` (상세 details 배열 첨부) | Y | 사용자가 수정 가능하도록 명확히 지시 |
| 권한/경계 오류 | 토큰 만료, 타 테넌트 접근, 허용 안된 Action | 401 / 403 | `ERR_AUTH_EXPIRED`, `ERR_ACCESS_DENIED` | Y | 해킹 시도로 간주, 로깅 강화 |
| 상태 전이 불가 오류 | 완료된 Job 취소 시도, 중복 재처리 시도 등 | 400 / 409 Conflict | `ERR_INVALID_JOB_STATE` | Y | 비즈니스 로직 위반 |
| 리소스 없음 | 존재하지 않는 jobId 조회 | 404 Not Found | `ERR_RESOURCE_NOT_FOUND` | Y | |
| 시스템 오류 (Internal) | DB 연결 실패, 스토리지 장애, 예기치 않은 예외 | 500 Internal Server Error | `ERR_SYSTEM_INTERNAL` | **Y (매우 중요)** | 사용자에게는 원인 감춤 (보안 락업) |

---

## 12. 감사 로그 및 보안 연계 기준

### 표 11. 감사 로그 및 보안 연계 기준
| 이벤트 | 설명 | 기록 필드 | 중요도 | 관련 핸들러 | 비고 |
| :--- | :--- | :--- | :--- | :--- | :--- |
| Job 생성 요청 | Bulk Import 업로드 발생 | actor, tenant, site, import_type | 높음 | `createBulkImportJob` | |
| 무단 접근 시도 차단 | 권한 부족 또는 경계 밖 리소스 접근 차단 | actor, ip_address, target_resource | 매우 높음 | 전체 공통 (미들웨어 레벨) | 보안 위협 식별 |
| Job 취소 실행 | 관리자에 의한 작업 강제 취소 | actor, job_id, reason | 매우 높음 | `cancelBulkImportJob` | 운영 개입 증빙 |
| Job 재처리 실행 | 실패 건 병합 업로드 | actor, original_job_id, new_job_id | 높음 | `retryBulkImportJob` | |
| 시스템 크래시 에러 | 예상치 못한 500 에러 발생 | trace_id, error_stack, actor (if auth) | 매우 높음 | Global Error Handler | ELK/Datadog 등과 연계 |

---

## 13. UI / TEST / MOCK 영향

*   **`UI-061` 연결:** UI 문서에서 명시된 상태 배지(PENDING, COMPLETED 등)는 본 라우트 핸들러가 반환하는 응답 DTO의 `status` 필드 값과 1:1로 일치해야 한다.
*   **`TEST-ADM-001` 연결:** 통합 테스트 명세에 정의된 400/403/404 예외 시나리오는 본 문서의 <표 7>, <표 8>, <표 10> 방어 로직을 통해 실제 발생해야 한다.
*   **`MOCK-001` 연결:** 본 문서에서 정의된 진입점(URL)과 응답 반환 기준(비동기 202, 공통 에러 스키마)을 준수하여 Mock 서버를 구축해야 한다.

---

## 14. Sprint 기준 우선순위

### 표 12. Sprint 우선순위 정리
| 핸들러/규칙 | Sprint | 이유 | 선행조건 | 후속 영향 | 비고 |
| :--- | :--- | :--- | :--- | :--- | :--- |
| 인증/권한 미들웨어 기반 확립 | Sprint 1 | 보안 락업 및 테넌트 격리 필수 | `COM-AUTH_v1` | 모든 핸들러 안전성 보장 | |
| `create`, `list`, `detail` 핸들러 | Sprint 1 | 최소 운영 흐름 확보 | API 규격 확정 | MOCK 개발 착수 | 코어 |
| 에러 변환기 (Global Exception Handler) | Sprint 1 | 공통 에러 스키마 일관성 적용 | `API-001` | 클라이언트 예외 처리 일원화 | |
| `failures` 상세 조회 및 `retry` 핸들러 | Sprint 1 | 부분 성공 및 재처리 검증 요건 충족 | Job 테이블 설계 | 현업 데이터 품질 대응 | |
| `summary` 요약 대시보드 API | Sprint 2 | 필수는 아님, 편의성 제공 | 기본 데이터 누적 | UI 고도화 지원 | |

---

## 15. 완료 기준 (Definition of Done)

*   **문서 완료 조건:** 모든 표(1~13)가 충실히 작성되었고, Command/Query 분리 기준과 권한 검증 기준이 명확히 선언됨.
*   **핸들러 책임 정의 완료 조건:** 각 엔드포인트가 '무엇을 검증하고', '어떤 DTO를 주고받는지', '예외 시 어떻게 응답하는지'에 대한 모호함이 소거됨.
*   **후속 구현 전달 가능 조건:** 백엔드 개발자가 본 문서를 읽고 추가적인 비즈니스 규칙 확인 없이도 Router, Middleware, Controller(Handler) 뼈대 코드와 에러 매핑 코드를 작성할 수 있는 상태임.

---

## 16. 다음 단계 작성 가이드

서버 진입 규격이 확립되었으므로, 이를 기반으로 독립적 개발과 모니터링을 준비하는 문서가 필요하다.

### 표 13. 후속 문서 연결 기준
| 후속 문서명 | 연결 이유 | 이 문서에서 넘겨줄 기준 정보 | 작성 우선순위 |
| :--- | :--- | :--- | :--- |
| `MOCK-001_bulk_import_mock_endpoint.md` | 프론트엔드 병렬 개발 착수 | 핸들러 URL, Command 202 응답, Query DTO 형태, 오류 반환 패턴 | 1 |
| `NFR-MON-001_bulk_import_monitoring.md` | 서버 입구 및 내부 오류 관제 | 공통 에러 스키마의 trace_id, Audit Log 매핑 키, 500 에러 처리 기준 | 2 |
| `DATA-011_bulk_import_seed_data.md` | 핸들러 동작 검증을 위한 데이터 준비 | DTO 검증용 정상/오류/부분성공 샘플 데이터 요구사항 | 3 |

### 각 문서의 필요성 (한 줄 설명)
1. **MOCK-001:** 본 핸들러 명세대로 동작하는 가짜 서버를 띄워, 백엔드 구현이 끝나기 전에 프론트엔드가 UI-061 연동 테스트를 완료하게 하기 위해 필요하다.
2. **NFR-MON-001:** 핸들러 단에서 발생한 401, 403, 500 에러와 Audit Log가 시스템 로그에 어떻게 적재되고 대시보드로 관제될지 설계하기 위해 필요하다.
3. **DATA-011:** MOCK 서버 주입 및 백엔드 핸들러 유닛/통합 테스트 구동 시 필요한 정형화된 Seed Data를 구축하기 위해 필요하다.

---

### Gemini 3.1 Pro 자체 검토 체크포인트
- [x] Command(상태 변경/비동기 응답)와 Query(데이터 조회/동기 응답) 분리 기준이 명확하게 반영되었는가? (표 4)
- [x] 핸들러 흐름 중 토큰 검증, 권한, 테넌트 경계 범위 검증 위치가 "비즈니스 로직 실행 전(Fail Fast)"으로 명시되었는가? (표 5)
- [x] API-002 등 기존 DTO 문서의 구조와 어긋나지 않는 입력/응답 매핑 기준이 존재하는가? (표 6, 8, 9)
- [x] 핸들러 내에서 발생하는 모든 예외가 API-001(공통 에러 스키마)로 변환되어 일관성 있게 반환되는가? (표 10)
- [x] 데이터 상태 변경(성공/실패) 및 권한 초과 접근 시 Audit Log를 남겨야 한다는 보안 락업 기준이 반영되었는가? (표 11)
- [x] TEST-ADM-001에 정의된 오류 시나리오를 이 핸들러에서 400/403/404/500 에러로 어떻게 응답할지 대응되는가? (13절)
- [x] 라우트 핸들러가 비즈니스 로직을 통째로 구현하는 거대한 클래스가 되지 않도록 '책임 분리(Thin Handler)' 원칙이 선언되었는가? (표 1)
- [x] Sprint 1 필수 범위인 기본 흐름(생성, 목록, 상세, 재처리, 취소)에 집중하고, 무리한 외부 연동은 후속으로 넘겼는가? (표 2)
- [x] 클라이언트 파라미터를 무조건 신뢰하지 않고 서버 측 보안 컨텍스트(토큰 기반 tenant_id 등)를 강제 주입하는 방어로직이 있는가? (표 8)
- [x] 구체적인 코드(TS/SQL) 없이 개발자가 프레임워크에 맞춰 구현할 수 있도록 언어 독립적이고 선언적인 아키텍처 문서로 완성되었는가? (9절)
