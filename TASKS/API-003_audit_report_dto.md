# API-003_audit_report_dto.md

## 1. 문서 목적
본 문서는 PRO ILI SMART의 Audit Session 및 Report 생명주기 관리에서 사용되는 데이터 전송 객체(DTO)의 구조, 타입, 제약조건을 정의한다. `F1-RH-001`에서 정의된 라우트 핸들러가 실제로 주고받을 입출력 페이로드의 공식 계약(Contract)을 수립하는 것이 목적이다.

## 2. DTO 설계 원칙
**표 1. DTO 설계 원칙**
| 원칙 | 내용 |
|---|---|
| **공식 계약(Contract) 존중** | 화면 렌더링용 임시 데이터가 아니라 백엔드, 프론트엔드, QA가 공유하는 유일한 SSOT(Single Source of Truth) 스키마. |
| **Command/Query 응답 분리** | 데이터를 삽입/수정하는 Command의 결과와 데이터를 나열/확인하는 Query의 응답 형태를 목적에 맞게 분리. |
| **명확한 상태 및 이력 표현** | Superseded, Reopened 등 비즈니스 생명주기 상태를 필드로 명확히 표현하여 클라이언트 판단을 도움. |
| **공통 에러 스키마 연계** | 모든 예외 DTO는 `API-001_common_error_schema.md` 포맷과 완전히 호환되게 구성. |

## 3. 적용 범위 정의
**표 2. 적용 범위 정의**
| 범위 | 내용 |
|---|---|
| **Sprint 1 포함 (구현 대상)** | Session 생성/수정/상태전이 요청/응답, Session 목록/상세 응답, Report 생성/목록/최신본/이력본 응답, 상태/검증결과/공통메타 DTO |
| **제외 (Sprint 2 이상)** | 구현 내부 클래스/ORM 엔티티 구조, DB 스키마, Vision AI 파라미터/결과셋 확장 DTO |

## 4. DTO 개체 분류
**표 3. DTO 개체 분류**
| 대분류 | 하위 개체 (목적) |
|---|---|
| **Session Request DTO** | Command (Create, Update, Submit 등), Query (목록 필터/페이징) |
| **Session Response DTO** | Summary (목록 조회용), Detail (상세 단건용) |
| **Report Request DTO** | Command (리포트 생성), Query (이력, 최신본 필터) |
| **Report Response DTO** | Summary (버전 목록용), Detail (리포트 본문 포함) |
| **Shared / Base DTO** | Status Enum, Version History, Error Validation, Common Metadata |

## 5. Audit Session 관련 DTO 목록
**표 4. Audit Session DTO 목록**
| DTO 명칭 | 목적 및 설명 |
|---|---|
| `CreateSessionRequest` | 신규 감사 세션 생성을 위한 필수 정보 페이로드 |
| `UpdateSessionRequest` | 진행 중 세션 정보 갱신 (STT 텍스트, 임시저장 내용 포함) |
| `ChangeSessionStateRequest` | Submit, Cancel, Reopen 등 상태 전이 목적의 페이로드 |
| `SessionListQueryRequest` | 세션 목록 페이징 및 상태, 날짜 필터링 파라미터 |
| `SessionSummaryResponse` | 목록 조회 시 반환되는 경량화된 세션 정보 |
| `SessionDetailResponse` | 단건 상세 조회 시 반환되는 딥(Deep) 데이터 구조 |

## 6. Audit Report 관련 DTO 목록
**표 5. Audit Report DTO 목록**
| DTO 명칭 | 목적 및 설명 |
|---|---|
| `GenerateReportRequest` | 특정 Submitted 세션 기반 리포트 생성 트리거 옵션 |
| `ReportListQueryRequest` | 특정 Workspace/Session 내 리포트 목록/이력 필터링 |
| `ReportSummaryResponse` | 최신본/이력본 목록 제공을 위한 경량 데이터 (본문 미포함) |
| `ReportDetailResponse` | 단건 리포트의 전체 마크다운/구조화 데이터 반환 |
| `ReportHistoryResponse` | 특정 세션의 리포트 버전 히스토리(Superseded 포함) 배열 구조 |

## 7. 요청 DTO 정의
**표 6. 요청 DTO 필드 정의**
| DTO명 | 주요 필드 | 필수 여부 | 형식/제약 | 상태/권한 의존 | 대표 검증 에러 |
|---|---|---|---|---|---|
| `CreateSessionRequest` | `title`, `auditDate`, `lineId` | 모두 필수 | string, ISO8601, UUID | `audit_editor` 필요 | 누락, Date 형식 위반 |
| `UpdateSessionRequest` | `sttRawText`, `findings` | 선택 | string, object[] | Draft/InProgress 전용 | 허용되지 않은 필드 조작 |
| `ChangeSessionStateReq` | `action` (SUBMIT, REOPEN 등), `reason` | `action` 필수 | Enum, string | 현재 상태 기반 차단 | 전이 불가 상태 (409) |
| `GenerateReportRequest` | `sessionId`, `includeSTT` | `sessionId` 필수 | UUID, boolean | Session 상태 Submitted 필수 | 타겟 세션 상태 미달 |
| `SessionListQueryReq` | `page`, `limit`, `stateFilter` | 모두 선택 | int(min 1), Enum | `audit_viewer` 이상 | Limit 범위(>100) 초과 |

## 8. 응답 DTO 정의
**표 7. 응답 DTO 필드 정의**
| 응답 종류 | 목적 | 주요 포함 내용 | 특징 |
|---|---|---|---|
| **Command 성공 응답** | 생성/수정/제출 결과 | 변경된 Entity의 ID, 갱신된 `updatedAt`, 최종 `state` | 성공 상태(200/201) 및 최소 필요 식별자 반환 |
| **비동기 진행중 응답** | 리포트 생성 큐 인입 | `taskId`, `status: GENERATING`, `estimatedTime` | 클라이언트가 진행률을 폴링(Polling)할 수 있는 키 제공 |
| **목록 (Query) 응답** | 데이터 리스팅 | `data` 배열 (Summary DTO), `meta` (페이징 정보) | 본문(Content) 필드를 배제하여 응답 속도 최적화 |
| **상세 (Query) 응답** | 단건 조회 | 전체 Detail DTO, 연관 히스토리 정보 요약 | 상세 분석을 위한 텍스트, 첨부파일 메타데이터 포함 |

## 9. 목록형 DTO 정의 (Summary Response)
**표 8. 목록형 DTO 구조**
| 필드명 | 타입 | 설명 |
|---|---|---|
| `id` | string(UUID) | 레코드 고유 식별자 |
| `title` | string | 세션 또는 리포트 제목 |
| `state` | Enum | 현재 비즈니스 상태 라벨 |
| `isLatest` | boolean | 리포트 목록의 경우 최신본 여부 표기 |
| `findingCount` | integer | (세션) 식별된 발견사항 개수 (Summary 전용) |
| `createdAt` / `updatedAt`| string | ISO8601 타임스탬프 |

## 10. 상세형 DTO 정의 (Detail Response)
**표 9. 상세형 DTO 구조**
| 데이터 블록 | 설명 및 포함 필드 |
|---|---|
| **Header / Meta** | `id`, `workspaceId`, `creatorId`, `traceId`, `version` 등 메타 및 권한 추적 정보 |
| **Summary** | 제목, 일자, 상태, 요약 등 상단 표시 정보 |
| **Findings / Details** | `sttRawText` (세션), 배열 형태의 `findings` (세션), 마크다운 형태의 `content` (리포트) |
| **Attachments** | 이미지, 음성 파일 등의 파일 스토리지 식별자 배열 |
| **Audit Trail** | 생성일, 수정일, 상태 전이 일시 (`submittedAt`, `generatedAt` 등) |

## 11. 상태 표현 DTO 정의
**표 10. 상태 표현 DTO 기준**
| 상태 (Enum) | 사용자 표시 라벨 | 운영 의미 | 허용 동작 | 차단 동작 |
|---|---|---|---|---|
| **DRAFT** | 작성 중 | 임시저장 데이터, 불완전함 | 내용 수정, 제출 시도 | 리포트 생성, 이력 조회 |
| **IN_PROGRESS**| 진행 중 | STT 등 입력이 진행되는 단계 | 수정, STT 추가, 제출 | 리포트 생성 |
| **SUBMITTED** | 제출 완료 | 입력 마감, AI 처리 대기 상태 | 리포트 생성, Reopen | 세션 내용 직접 수정 |
| **FINALIZED** | 확정 | 관리자가 최종 승인한 상태 | 이력 조회 | 모든 내용 변경, 신규 리포트 생성 |
| **REOPENED** | 재개됨 | 제출 후 수정 지시가 내려진 상태 | 내용 추가 수정, 재제출 | 기존 파생 리포트의 공식 상태 갱신 |
| **GENERATING** | 생성 중 (리포트) | 백그라운드 AI 요약 진행 중 | 조회, 폴링 | 중복 생성 요청, 수정 |
| **GENERATED** | 생성 완료 (리포트)| AI 요약이 완료된 최신본 리포트 | 확정, 이력 조회 | 내용 수정 (AI 출력은 불변) |
| **SUPERSEDED** | 이전 버전 (리포트)| Reopen/재제출로 인해 대체된 구버전 | 이력 조회 | 확정, 수정, 배포 |

## 12. 버전 / 이력 표현 DTO 정의
**표 11. 버전 / 이력 DTO 기준**
| 필드명 | 타입 | 설명 |
|---|---|---|
| `versionNo` | integer | 생성 순서에 따른 버전 번호 (1, 2, 3...) |
| `isLatest` | boolean | 현재 세션에서 가장 최신 버전인지 여부 (단 하나만 true) |
| `isSuperseded` | boolean | 새로운 리포트 생성에 의해 과거 데이터로 밀려났는지 여부 |
| `supersededReason` | string | 이전 버전으로 강등된 사유 (예: "Session Reopened & Resubmitted") |
| `sourceSessionId` | string | 파생 원본이 된 세션 ID |
| `generatedAt` / `invalidatedAt` | string | 생성 시점 및 Superseded 강등 시점 |

## 13. 에러 / 검증 결과 표현 DTO 기준
**표 12. 에러 / 검증 결과 DTO 기준**
(`API-001_common_error_schema.md`를 엄격히 준수하여 래핑한다.)
| 에러 발생 사유 | 내부 에러 코드 (Code) | HTTP Status | Response DTO `details` 구조 예시 |
|---|---|---|---|
| **필수값 누락/형식** | `VALIDATION_FAILED` | 400 | `[{"field": "title", "issue": "required"}]` |
| **상태 불일치** | `STATE_CONFLICT` | 409 | `{"expected": "SUBMITTED", "current": "DRAFT"}` |
| **권한 / 범위 오류** | `FORBIDDEN_ACCESS` | 403 | `{"requiredRole": "audit_editor"}` |
| **데이터 없음** | `RESOURCE_NOT_FOUND` | 404 | `{"resourceType": "AuditSession", "id": "req-id"}` |

## 14. 공통 메타데이터 정의 (Meta DTO)
**표 13. 공통 메타데이터 정의**
모든 최상위 Response DTO에 포함되는 `meta` 객체의 구조.
| 필드명 | 타입 | 설명 |
|---|---|---|
| `traceId` | string | 요청-응답-감사로그를 이어주는 분산 추적 식별자 (필수) |
| `timestamp` | string | 응답 생성 서버 시간 (ISO8601) |
| `actor` | object | 요청자 컨텍스트 (예: `{ "id": "user-1", "role": "audit_editor" }`) |
| `tenantContext`| object | 처리된 Tenant 및 Site 정보 |
| `pagination` | object | (목록 조회 시) `totalCount`, `currentPage`, `hasNext`, `cursor` |

## 15. 감사 로그 / 추적 연계 기준
**표 14. 감사 로그 / 추적 연계 기준**
| 항목 | 기준 설명 |
|---|---|
| **추적 식별자(Trace ID)** | RH가 수집한 헤더의 `x-trace-id`는 반드시 응답 DTO의 `meta.traceId`에 주입되어야 하며, UI는 이를 에러 화면에 표출하여 트러블슈팅에 사용한다. |
| **고위험 요청 응답** | 세션 삭제(취소) 및 상태 전이(제출) 시, 응답 DTO에는 성공했음을 나타내는 `state` 반환과 별개로 Audit Log 저장용 Meta Context가 내부적으로 연결된다. |

## 16. API / UI / TEST / MOCK 영향 및 후속 문서 연결 기준
**표 16. 후속 문서 연결 기준**
| 연결 문서 | 역할 및 영향 |
|---|---|
| **F1-RH-001_audit_route_handler.md** | 본 문서의 DTO 스키마는 Route Handler 내부에서 입력 검증 파이프라인(Zod)으로 직접 변환되어 동작한다. |
| **UI-010_audit_workspace_page.md** | 프론트엔드는 본 DTO를 바탕으로 TypeScript Interface를 선언하며, 상태값(Enum)을 기준으로 버튼 활성화를 제어한다. |
| **TEST-F1-001/002_xxx.md** | 통합 테스트 시나리오에서 Expect/Assert로 검증하는 대상 구조가 본 DTO의 Response 형태가 된다. |
| **MOCK-002_audit_report_mock.md** | Mock Endpoint는 본 문서의 스키마와 100% 동일한 JSON을 정적으로 반환하도록 설계된다. |

## 17. Sprint 우선순위
**표 15. Sprint 우선순위 정리**
| 구분 | 내용 | 우선순위 |
|---|---|---|
| **Session Core DTO** | 생성, 수정, 제출 요청/응답, 목록/상세 응답, Enum 상태 구조 | High (P0) |
| **Report Core DTO** | 생성 요청/진행 응답, 상세 응답, 최신본/이력본 목록 응답 | High (P0) |
| **Error / Meta DTO** | 공통 에러 연계, 페이징/추적용 Meta 구조 통일 | High (P0) |
| **Vision Extension DTO**| 이미지 분석 결과, Bounding Box 메타데이터 구조 | Low (Sprint 2) |

## 18. 완료 기준 (Definition of Done)
- [ ] Session과 Report의 Command/Query DTO 구조가 명확히 분리되었는가?
- [ ] 상태 전이 및 버전 이력(Superseded) 표현 필드가 명세에 모두 반영되었는가?
- [ ] API-001 공통 에러 및 추적(meta) 정보가 누락 없이 적용되었는가?
- [ ] 백엔드와 프론트엔드가 공유할 수 있는 레벨의 타입/스키마 명세가 완성되었는가?

## 19. 다음 단계 작성 가이드
요청을 처리하는 입구(RH)와 계약 규격(DTO) 설계가 완료되었다. 다음 단계는 이 계약을 소비하여 사용자에게 제공할 화면인 `UI-010_audit_workspace_page.md` 문서와, 이를 철저히 검증할 `TEST` 및 `MOCK` 문서를 작성하는 것이다.

## 20. Gemini 3.1 Pro 자체 검토 체크리스트
- [x] Session DTO와 Report DTO가 서로 침범하지 않고 명확히 분리되었는가?
- [x] Command 응답과 Query 응답의 설계 목적 차이가 목록형/상세형 구조에 반영되었는가?
- [x] Superseded 및 버전 관리 필드가 이력 DTO에 올바르게 포함되었는가?
- [x] 에러/메타/상태 DTO가 공통 표준 및 RH 문서와 강하게 정합성을 맺고 있는가?
- [x] 프론트엔드와 QA팀이 이 문서를 보고 인터페이스/Mock을 작성할 수 있을 만큼 구체적인가?
