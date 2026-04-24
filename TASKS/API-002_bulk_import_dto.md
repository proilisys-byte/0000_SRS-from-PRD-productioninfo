# [API-002] Bulk Import DTO & Contract Definition

## 1. 문서 목적
본 문서는 **PRO ILI SMART** 프로젝트의 핵심 기능인 Bulk Import(대량 업로드) 처리 과정에서 사용되는 데이터 전송 객체(DTO, Data Transfer Object)와 API 계약(Contract)을 정의한다. 

Bulk Import 도메인에서 DTO가 먼저 정의되어야 하는 이유는 다음과 같다.
- **일관된 기준 제공**: 관리자 처리(ADM-C), 관리자 조회(ADM-Q), UI(UI), 테스트(TEST) 로직이 동일한 데이터 구조를 바라보아야만 "부분 성공", "실패 내역 확인" 등 복잡한 상태를 오차 없이 처리할 수 있다.
- **후속 문서와의 관계**: 본 문서에서 정의된 DTO를 바탕으로 API 라우트 핸들러 설계, UI 컴포넌트 프로퍼티 설계, QA 테스트 케이스 명세가 파생된다.

## 2. DTO 설계 원칙

### 표 1. DTO 설계 원칙
| 원칙 | 설명 | 적용 이유 | 관련 문서 | Sprint 우선순위 |
| :--- | :--- | :--- | :--- | :--- |
| **일관성** | 모든 API 응답은 `API-001_common_error_schema.md` 기반의 성공/실패 래퍼를 유지한다. | 클라이언트 예외 처리의 단일화 | API-001_common_error_schema.md | Sprint 1 필수 |
| **최소 충분성** | 화면에 필요하거나 상태 추적에 필수적인 필드만 포함하고, 내부 DB 식별자 등은 최소 노출한다. | 결합도 저하 및 보안 강화 | DATA-SCHEMA_v1.md | Sprint 1 필수 |
| **사용자/UI 친화성** | 부분 성공, 실패 건수 등 UI 진행바 및 통계 노출에 즉시 사용 가능한 형태를 제공한다. | 프론트엔드 파싱 부담 최소화 | UI-061_bulk_import_admin_dashboard.md | Sprint 1 필수 |
| **운영 추적 가능성** | 각 요청은 고유의 `job_id`와 `trace_id`를 지니며 Audit Log에 매핑되어야 한다. | 사고 발생 시 즉각적인 원인 규명 | DATA-AUDIT_LOG_v1.md | Sprint 1 필수 |
| **멀티테넌시 반영** | 테넌트(Tenant) 및 사이트(Site) 경계를 명확히 식별할 수 있는 맥락 정보가 포함되어야 한다. | 데이터 오염 및 권한 탈취 방지 | COM-AUTH_v1.md | Sprint 1 필수 |

## 3. 적용 범위 정의

### 표 2. 적용 범위 정의
| DTO 영역 | 설명 | Sprint 적용 시점 | 필수 여부 | 비고 |
| :--- | :--- | :--- | :--- | :--- |
| **업로드 요청/응답** | 파일 메타데이터 전달 및 비동기/동기 Job 생성 응답 | Sprint 1 | 필수 | 파일 본문은 Multipart/form-data 또는 Signed URL 처리 (본 문서는 메타 중심) |
| **배치 상태/요약 조회** | 생성된 Job의 진행률, 성공/실패 통계 조회 | Sprint 1 | 필수 | UI 대시보드 연동용 |
| **오류 상세 조회** | 행(Row) 및 필드(Field) 단위의 검증 실패 사유 | Sprint 1 | 필수 | 사용자 시정(NC) 조치용 |
| **재처리 요청** | 실패한 건에 대한 재검증/재처리 지시 | Sprint 2 | 선택적 확장 | Sprint 1은 재업로드로 우회 가능하나 계약은 선점 |
| **AI 검증 결과 DTO** | 업로드 데이터에 대한 Vision AI/NLP 검증 결과 메타 | Sprint 3 | - | 아웃오브스코프 |

## 4. Bulk Import 주요 API/업무 시나리오 맵

### 표 3. Bulk Import 주요 시나리오 맵
| 시나리오 | 설명 | 입력 DTO | 출력 DTO | 관련 문서 | 비고 |
| :--- | :--- | :--- | :--- | :--- | :--- |
| **업로드 요청** | 사용자가 파일을 업로드하여 배치를 생성 | `BulkImportUploadRequestDto` | `BulkImportUploadResponseDto` | ADM-BULK_v1.md | Job ID 반환 |
| **배치 상태 조회** | 진행 중인 Job의 현재 상태 확인 | Job ID (Path Variable) | `BulkImportJobSummaryDto` | UI-061_bulk_import... | 폴링 또는 SSE 연계 |
| **결과 요약 조회** | 완료된 Job의 성공/실패/총건수 조회 | Job ID (Path Variable) | `BulkImportResultStatsDto` | ADM-Q-001_bulk... | 통계 노출 |
| **실패 항목 조회** | 오류가 발생한 데이터의 상세 사유 조회 | Job ID, Pagination params | `BulkImportValidationResultDto` | ADM-Q-001_bulk... | 행/필드 단위 오류 |
| **재처리 요청** | 실패 항목에 한해 재시도 지시 | `BulkImportRetryRequestDto` | `BulkImportRetryResponseDto` | ADM-C-001_bulk... | 부분 재처리 |

## 5. 핵심 DTO 목록

### 표 4. 핵심 DTO 목록
| DTO명 | 역할 | Sprint 적용 시점 | 필수 여부 | 관련 기능 | 비고 |
| :--- | :--- | :--- | :--- | :--- | :--- |
| `BulkImportUploadRequestDto` | 업로드 대상 파일의 메타데이터 및 타입 정의 | Sprint 1 | 필수 | 파일 업로드 | |
| `BulkImportUploadResponseDto` | 접수된 Job의 식별자 및 초기 상태 | Sprint 1 | 필수 | 파일 업로드 | |
| `BulkImportJobSummaryDto` | Job의 진행 상태 및 처리 통계 | Sprint 1 | 필수 | 상태 조회 | |
| `BulkImportValidationResultDto` | 전체 검증 결과 래퍼 (성공, 실패 목록) | Sprint 1 | 필수 | 오류 조회 | |
| `BulkImportRowErrorDto` | 특정 데이터 행(Row)에서 발생한 오류 세부 내역 | Sprint 1 | 필수 | 오류 조회 | |
| `BulkImportFieldErrorDto` | 특정 필드(컬럼) 레벨의 검증 실패 내역 | Sprint 1 | 필수 | 오류 조회 | |
| `BulkImportRetryRequestDto` | 재처리 대상 식별자 및 옵션 | Sprint 2 | - | 재처리 | |
| `BulkImportResultStatsDto` | 성공/실패/총합 카운트 정보 | Sprint 1 | 필수 | 공통 통계 | |
| `BulkImportFileMetaDto` | 업로드된 원본 파일에 대한 기본 정보 | Sprint 1 | 필수 | 공통 정보 | |

## 6. 요청 DTO 정의

### 표 5. 요청 DTO 필드 정의
| DTO명 | 필드명 | 설명 | 타입 | 필수 여부 | 예시 | 비고 |
| :--- | :--- | :--- | :--- | :--- | :--- | :--- |
| **BulkImportUploadRequestDto** | `import_type` | 업로드 대상 도메인 유형 | Enum(String) | Y | `"MASTER_DATA"`, `"INSPECTION"` | 도메인 분기 처리용 |
| | `file_meta` | 업로드할 파일 정보 | `BulkImportFileMetaDto` | Y | - | |
| | `site_id` | 데이터가 귀속될 현장/공정 ID | UUID(String) | N | `"uuid-1234"` | 테넌트 내부 분리용 |
| **BulkImportRetryRequestDto** | `job_id` | 원본 업로드 작업 식별자 | UUID(String) | Y | `"job-uuid"` | |
| | `target_row_indices` | 재처리할 특정 행 번호 배열 (비어있으면 전체 실패건) | Array[Number] | N | `[2, 5, 11]` | 선택적 부분 재처리 |
| **BulkImportFileMetaDto** | `file_name` | 원본 파일명 | String | Y | `"2026_q1_data.csv"` | |
| | `file_size` | 파일 크기 (bytes) | Number | Y | `1024500` | 용량 제한 검증용 |
| | `mime_type` | 파일 형식 | String | Y | `"text/csv"` | |

## 7. 응답 DTO 정의

### 표 6. 응답 DTO 필드 정의
| DTO명 | 필드명 | 설명 | 타입 | 필수 여부 | 예시 | 비고 |
| :--- | :--- | :--- | :--- | :--- | :--- | :--- |
| **BulkImportUploadResponseDto**| `job_id` | 생성된 백그라운드 작업 고유 식별자 | UUID(String) | Y | `"job-uuid"` | 후속 추적용 키 |
| | `status` | 초기 접수 상태 | Enum(String) | Y | `"PENDING"` | |
| | `estimated_rows` | 처리 예상 행 수 (헤더 제외) | Number | N | `1500` | 진행률 계산 보조 |
| **BulkImportJobSummaryDto** | `job_id` | 작업 식별자 | UUID(String) | Y | `"job-uuid"` | |
| | `status` | 현재 작업 상태 | Enum(String) | Y | `"PARTIAL_SUCCESS"` | |
| | `stats` | 처리 통계 | `BulkImportResultStatsDto` | Y | - | 건수 요약 |
| | `started_at` | 처리 시작 시각 | ISO Date | N | `"2026-..."` | |
| | `completed_at` | 처리 종료 시각 | ISO Date | N | `"2026-..."` | |
| **BulkImportResultStatsDto** | `total_rows` | 전체 대상 행 수 | Number | Y | `1000` | |
| | `success_count` | 성공적으로 DB에 반영된 행 수 | Number | Y | `950` | |
| | `failed_count` | 유효성/비즈니스 검증 실패 행 수 | Number | Y | `50` | |
| **BulkImportValidationResultDto**| `job_id` | 작업 식별자 | UUID(String) | Y | `"job-uuid"` | |
| | `stats` | 통계 요약 | `BulkImportResultStatsDto` | Y | - | |
| | `row_errors` | 실패한 행의 상세 내역 목록 (페이징 가능) | Array[`RowErrorDto`] | Y | `[...]` | |
| **BulkImportRowErrorDto** | `row_index` | 엑셀/CSV 원본의 실제 행 번호 (1-based) | Number | Y | `15` | 사용자 시정용 가이드 |
| | `raw_data` | 오류 발생 행의 원본 텍스트/JSON 덤프 | Object/String | N | `{"colA": "bad_val"}` | 원인 추적용 보조 데이터 |
| | `field_errors` | 해당 행에 속한 개별 필드 오류 목록 | Array[`FieldErrorDto`] | Y | `[...]` | |
| **BulkImportFieldErrorDto** | `field_name` | 오류가 발생한 헤더/속성명 | String | Y | `"inspector_name"` | |
| | `error_code` | 공통 에러 스키마에 부합하는 에러 코드 | String | Y | `"ERR_VALIDATION_001"` | |
| | `message` | 사용자 친화적 오류 메시지 | String | Y | `"검사자 이름은 필수입니다."` | |

## 8. 상태 및 결과 표현 구조

### 표 7. 상태 및 결과 표현 구조
| 구분 | 값 | 의미 | 사용 위치 | 비고 |
| :--- | :--- | :--- | :--- | :--- |
| **Job 상태 (`status`)** | `PENDING` | 대기열에 진입함 (처리 전) | Summary DTO | 최초 업로드 시점 |
| | `PROCESSING` | 데이터 파싱 및 DB 검증/적용 중 | Summary DTO | 진행 중 |
| | `SUCCESS` | 모든 데이터 행이 오류 없이 100% 처리 완료됨 | Summary DTO | 처리 종료 |
| | `PARTIAL_SUCCESS` | 일부 행은 반영 성공, 일부 행은 실패 (부분 처리 허용 시) | Summary DTO | **Bulk Import의 핵심 상태** |
| | `FAILED` | 치명적 오류(파일 포맷, 헤더 불일치)로 단 1건도 처리되지 못함 | Summary DTO | 처리 중단 |
| | `CANCELED` | 관리자에 의해 강제 중단됨 | Summary DTO | Sprint 2 고려 |

## 9. 검증 및 오류 표현 구조

### 표 8. 검증/오류 표현 구조
| 오류 레벨 | 설명 | 표현 DTO | 사용자 노출 여부 | 운영 활용 방식 | 비고 |
| :--- | :--- | :--- | :--- | :--- | :--- |
| **파일 레벨 오류** | 포맷 오조작, 용량 초과, 헤더 누락 등 전면 거부 사유 | 공통 Error Schema (`400 Bad Request`) | Y (토스트 메시지 등) | Job이 생성되지 않거나 즉시 FAILED 상태 전환 | `API-001_common_error_schema.md` 참조 |
| **행(Row) 레벨 오류** | 필수값 누락, 중복 키, 외래 키 참조 오류 등 건별 실패 | `BulkImportRowErrorDto` | Y (오류 상세 모달/그리드) | 실패 행만 분리하여 추출(Export)하거나 재처리 유도 | |
| **필드(Field) 레벨 오류** | 개별 셀 단위의 형식 불일치 (타입, 길이, 정규식 등) | `BulkImportFieldErrorDto` | Y (셀 하이라이트) | 정확히 어떤 열에서 문제가 생겼는지 지적 | |
| **보안/권한 오류** | 다른 테넌트/사이트 데이터를 조작 시도, 쓰기 권한 부족 | 공통 Error Schema (`403 Forbidden`) | Y | 보안 로그 연동 (Risk Event) | |

## 10. 공통 필드 및 메타데이터 기준

### 표 9. 공통 메타데이터 기준
| 필드명 | 설명 | 필수 여부 | 출처 | 관련 문서 | 비고 |
| :--- | :--- | :--- | :--- | :--- | :--- |
| `tenant_id` | 소속 기업 식별자 | Y | Auth Context (JWT) | COM-AUTH_v1.md | DTO Body가 아닌 인증 컨텍스트에서 추출하여 강제 주입 |
| `site_id` | 소속 공장/현장 식별자 | Y/N | JWT or Request Body | COM-AUTH_v1.md | 관리자 범위에 따라 처리 |
| `actor_id` | 업로드 수행 사용자 UUID | Y | Auth Context (JWT) | COM-AUTH_v1.md | |
| `job_id` | 배치 처리 단위의 식별자 | Y | 시스템 생성 (UUID) | ADM-BULK_v1.md | 상태 추적 기준 키 |
| `trace_id` | 모니터링/로그용 식별자 | Y | 시스템/APM 생성 | NFR-MON-001... | 클라이언트 <-> 서버 <-> 워커 통일 |

## 11. 감사 로그 및 보안 연계

### 표 10. 감사 로그 및 보안 연계 기준
| 항목 | 설명 | DTO 반영 여부 | 관련 문서 | 비고 |
| :--- | :--- | :--- | :--- | :--- |
| **이벤트 로깅** | Bulk Import `SUCCESS` / `PARTIAL_SUCCESS` 발생 시 Audit Log에 요약 기록 | `stats` (총건수, 성공/실패건수) | DATA-AUDIT_LOG_v1.md | DB Insert와 동일 트랜잭션 내 기록 권장 |
| **고위험 작업 추적** | `actor_id` 및 `job_id` 기반으로 누가 어떤 배치를 올렸는지 추적 가능 | `actor_id`, `job_id` | DATA-AUDIT_LOG_v1.md | `target_resource` = "BULK_JOB", `resource_id` = `job_id` 매핑 |
| **민감정보 비노출** | `RowErrorDto.raw_data`에 PII(개인정보), 비밀번호 등 포함 금지 | 마스킹 또는 포함 금지 원칙 준수 | COM-AUTH_v1.md | |

## 12. 후속 문서 연결 기준

### 표 12. 후속 문서 연결 기준
| 후속 문서명 | 연결 이유 | 이 문서에서 넘겨줄 기준 정보 | 작성 우선순위 |
| :--- | :--- | :--- | :--- |
| `ADM-C-001_bulk_import_job_management.md` | 배치 워커(서버리스/큐)의 처리 규약 정의 | `status` 생명주기 및 응답/검증 결과 DTO 형식 | 높음 |
| `ADM-Q-001_ বাস্তবায়_import_job_list_query.md` | 관리자의 이력 조회 API 및 필터 정의 | `BulkImportJobSummaryDto` 구조 | 높음 |
| `UI-061_bulk_import_admin_dashboard.md` | 관리자 화면 진행률 바, 요약, 에러 그리드 프로퍼티 | `ResultStatsDto`, `RowErrorDto` 구조 | 높음 |
| `TEST-ADM-001_bulk_import_job_flow_test.md` | `PARTIAL_SUCCESS` 등 예외 시나리오 Mocking 및 Assert 기준 | 부분 성공 구조, 필드 오류 구조 | 보통 |

## 13. Sprint 기준 우선순위

### 표 11. Sprint 우선순위 정리
| DTO/구조 | Sprint | 이유 | 선행조건 | 후속 영향 | 비고 |
| :--- | :--- | :--- | :--- | :--- | :--- |
| **Upload Request/Response** | 1 | 핵심 파일 업로드 진입점 | 공통 Auth/DB 설정 | 백그라운드 Job 큐 | 필수 |
| **Job Summary / Stats** | 1 | 상태 조회 및 운영 가능성 확보 (최소한의 가시성) | Job 테이블 스키마 | UI 진행바 컴포넌트 | 필수 |
| **Validation / Row Error** | 1 | 사용자 스스로 에러를 시정할 수 있게 하는 필수 정보 | 검증 로직, 에러 코드 표준 | 에러 노출 그리드 | 필수 |
| **Retry Request** | 2 | 단순 재업로드가 아닌 '실패건 선택적 재처리'는 S2 고도화 | Job 상세 스냅샷 확보 | 부분 재처리 로직 | 선택적 확장 |

## 14. 완료 기준(Definition of Done)
- **문서 완료 조건**: 요청, 상태, 오류, 부분 성공을 나타내는 DTO 명세가 명확히 정리됨.
- **계약 정의 완료 조건**: 공통 Error Schema와 본 DTO 간의 충돌 및 중복이 없음.
- **후속 문서 전달 가능 조건**: Frontend(UI 컴포넌트 설계) 및 Backend(Controller/Service 설계) 개발자가 API 명세서 작성용으로 바로 인용할 수 있는 수준.
- **테스트 가능 조건**: QA 엔지니어가 `PARTIAL_SUCCESS` 시나리오 응답 검증 케이스를 작성할 수 있음.

---

## 15. 다음 단계 작성 가이드

다음 순서로 상세 스펙 작성을 권장한다.

1. **`ADM-C-001_bulk_import_job_management.md`**: 정의된 DTO를 입력받아 실제 DB 처리(Chunk 단위 트랜잭션, 롤백/부분커밋)를 수행하는 워커 로직 명세
2. **`ADM-Q-001_bulk_import_job_list_query.md`**: 저장된 Job 상태 및 Error 내역을 페이징, 필터링하여 제공하는 조회 API 명세
3. **`UI-061_bulk_import_admin_dashboard.md`**: 본 계약을 기반으로 화면의 진행률, 오류 모달, 요약 카드를 렌더링하는 컴포넌트 명세
4. **`TEST-ADM-001_bulk_import_job_flow_test.md`**: DTO 구조와 공통 오류 코드를 활용한 E2E/통합 테스트 시나리오
5. **`COM-RH-001_bulk_import_route_handler.md`**: Next.js API Routes (또는 Hono 등)에서의 엔드포인트 바인딩 명세
6. **`MOCK-001_bulk_import_mock_endpoint.md`**: FE/QA 병렬 작업을 위한 MSW/Mocking 데이터 응답 구조
7. **`NFR-MON-001_bulk_import_monitoring.md`**: 실패/대용량 부하 발생 시 경보(Alert) 및 모니터링 연동 기준
8. **`DATA-011_bulk_import_seed_data.md`**: 테스트 환경에 적재할 기본 포맷/양식 파일 메타데이터

각 문서는 도메인 분리 원칙에 따라 "계약(DTO) -> 처리(C) -> 조회(Q) -> 표현(UI) -> 검증(TEST)"의 흐름을 갖는다.

---

### Gemini 3.1 Pro 자체 검토 체크포인트

1. **요청/응답 DTO 구분이 명확한가?** ✅ (Upload, JobSummary, Retry 등으로 구분 완료)
2. **부분 성공 구조가 빠지지 않았는가?** ✅ (`PARTIAL_SUCCESS` 상태값과 성공/실패 카운트 분리 정의)
3. **행 단위/필드 단위 오류 표현이 충분한가?** ✅ (`RowErrorDto`, `FieldErrorDto` 분리 정의)
4. **상태 표현이 운영 관점에서 usable 한가?** ✅ (PENDING, PROCESSING, SUCCESS, PARTIAL_SUCCESS, FAILED로 세분화)
5. **공통 에러 스키마와 충돌이 없는가?** ✅ (파일 레벨은 Common Error Schema, 내부 레벨은 별도 DTO로 역할 분담 명시)
6. **tenant/site/actor 맥락이 필요한 곳에 반영되었는가?** ✅ (공통 메타데이터 기준 섹션에 명시, JWT 기반 주입 원칙 포함)
7. **UI와 테스트에서 바로 사용할 수 있는 수준인가?** ✅ (Type/필수 여부/예시 값 제공으로 직관성 확보)
8. **Sprint 1 범위를 넘는 과도한 구조가 섞이지 않았는가?** ✅ (Retry, Canceled 등은 Sprint 2로 명확히 이관/분리 표기)
9. **감사 로그 연결 필드가 누락되지 않았는가?** ✅ (`job_id`, `actor_id`, `stats` 항목을 Audit Log 매핑 요소로 명시)
10. **재처리 흐름을 지원할 수 있는 계약인가?** ✅ (Sprint 2 확장이지만 `BulkImportRetryRequestDto`의 target_row_indices로 기반 마련)
