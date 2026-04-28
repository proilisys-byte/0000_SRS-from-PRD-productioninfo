# 05. Sprint 1 설계 결정 확정 항목 (Design Decision Lock)

## 1. 문서 목적
본 문서는 PRO ILI SMART 프로젝트의 Sprint 1 구현에 돌입하기 전, 프론트엔드/백엔드/QA 팀 간에 이견이 발생하거나 충돌할 수 있는 아키텍처 및 설계 의사결정을 식별하고 최종 확정(Lock)하기 위해 작성되었습니다.

## 2. 검토 입력 문서
- 기반 문서: `00_PRD_v1.md`, `05_SRS_v1.md`, `08_DECISION_LOG_v1.md` (현재 작성/보완 중)
- F1 도메인 문서: `F1-C-001`, `F1-C-002`, `F1-Q-001`, `F1-RH-001`
- DTO 및 Mock: `API-003_audit_report_dto.md`, `MOCK-003_audit_list_mock_endpoint.md`, `DATA-012_smart_audit_seed_data.md`
- UI 명세: `UI-011_audit_session_list.md`

## 3. Sprint 1 설계 결정이 필요한 이유
현재 작성된 명세서들(예: Query 명세와 DTO 명세) 간에 일부 불일치(예: Pagination 방식)가 발견되었습니다. 구현이 시작된 후 이러한 차이가 발견되면 API 재설계 및 컴포넌트 재작업으로 인해 치명적인 일정 지연이 발생합니다. 따라서 코딩 전 유일한 진실의 원천(SSOT)을 확정해야 합니다.

## 4. 확정 사항
기존 문서들을 통해 이미 합의가 완료되어 변경 없이 구현에 들어갈 설계 사항입니다.

| 확정 항목 | 내용 | 근거 문서 |
|---|---|---|
| **데이터 격리 (Multi-tenant)** | Tenant ID, Site ID 검증 없는 쿼리는 원천 차단하며 403 반환 | `F1-Q-001` |
| **생명주기 상태값 (Enum)** | DRAFT, IN_PROGRESS, SUBMITTED, FINALIZED, REOPENED, SUPERSEDED 유지 | `API-003` |
| **공통 에러 반환 구조** | 모든 API 예외는 `API-001_common_error_schema.md` 규격을 강제함 | 전체 통합 |
| **이력 관리 방식** | 수정 시 과거 데이터는 `isLatest=false` 처리 후 `SUPERSEDED`로 남김 | `F1-Q-001` |

## 5. 미확정 설계 결정 항목
관련 문서 간 충돌이 있거나 명확히 정의되지 않아 즉시 결정해야 하는 항목입니다.

| 항목 ID | 결정 필요 항목 | 발생 원인 / 병목 사유 |
|---|---|---|
| **DEC-001** | Pagination 구현 방식 통일 | `F1-Q-001`은 Offset 방식 명시, `API-003`은 `cursor` 필드 존재. UI 구현(무한스크롤 vs 페이징 바) 시 충돌 |
| **DEC-002** | 다국어/에러 메시지 처리 주체 | 백엔드가 메시지를 내려줄지, 에러 코드만 주면 클라이언트가 렌더링할지 미정 |
| **DEC-003** | STT 텍스트 및 첨부파일 스토리지 전략 | DB 텍스트 저장 방식과 파일 스토리지 분리 기준 미비 (데이터베이스 부하) |
| **DEC-004** | 권한/세션 만료 처리 로직 | 토큰 만료 시 Silent Refresh 방식 vs 즉시 로그인 리다이렉트 방식 미결정 |

## 6. 항목별 선택지 비교

| 항목 ID | 옵션 A | 옵션 B | 장단점 비교 |
|---|---|---|---|
| **DEC-001** | **Offset 기반 (Limit/Page)** | **Cursor 기반 (무한 스크롤)** | A: 구현 난이도 낮음, 특정 페이지 점프 가능. B: 대규모 데이터에서 성능 유리, 실시간 추가 시 중복 노출 없음 |
| **DEC-002** | **백엔드(Server) 번역** | **프론트엔드(Client) 매핑** | A: 클라이언트 로직 단순화. B: 트래픽 최적화, UI 문구 수정 시 백엔드 배포 불필요 (B2B SaaS에 유리) |
| **DEC-003** | **RDBMS 텍스트 저장** | **오브젝트 스토리지 저장** | A: 쿼리 용이성. B: DB 부하 최소화 및 비용 효율성 |
| **DEC-004** | **Silent Refresh (백그라운드)** | **만료 시 즉시 로그아웃** | A: 사용자 경험 우수, 모바일 친화적. B: 구현 난이도 낮고 보안 통제 엄격 |

## 7. 권장 결정안

1. **DEC-001 (Pagination):** Sprint 1의 빠른 딜리버리와 백엔드 보수적 안정성을 고려하여 **옵션 A (Offset 기반 페이징)** 확정 권장. API-003의 `cursor` 필드 삭제.
2. **DEC-002 (Error Message):** **옵션 B (클라이언트 매핑)** 권장. API는 철저하게 `error_code`만 반환하고 UI 단의 i18n 라이브러리에서 매핑.
3. **DEC-003 (STT/Attachments):** STT Text는 **RDBMS**(`text` 컬럼), 이미지/음성 파일은 **오브젝트 스토리지(S3 등)**로 분리 저장.
4. **DEC-004 (Auth Refresh):** 쿠키 기반의 **Silent Refresh(옵션 A)** 권장. 현장 작업 중 토큰 만료로 인한 데이터 유실 방지가 핵심 비즈니스 가치임.

## 8. 후속 영향 문서

| 권장안 | 수정 필요 문서 | 수정 내용 |
|---|---|---|
| DEC-001 | `API-003_audit_report_dto.md` | `meta.pagination` 객체 내 `cursor` 필드 제거, `page` 및 `limit` 명시 |
| DEC-002 | `UI-001_login_page.md` | `API-001`의 `error_code` 기반 클라이언트 다국어 사전 매핑 로직 추가 |
| DEC-003 | `F1-C-001_audit_session_management.md` | STT 텍스트 최대 글자 수 제약 추가 및 첨부파일 URL 저장 방식 명시 |
| DEC-004 | `COM-RH-002_auth_route_handler.md` | `HttpOnly` Refresh Token 쿠키 발급 및 갱신 라우트 로직 구체화 |

## 9. 결정 우선순위

| 순위 | 항목 ID | 담당자 (Decision Maker) | 타임라인 | 비고 |
|:---:|---|---|---|---|
| 1 | **DEC-001 (Pagination)** | Backend / Frontend Lead | 즉시 (Sprint 1 Day 1) | API DTO와 UI 컴포넌트 설계에 직접적 블로커 |
| 2 | **DEC-004 (Auth Refresh)** | System Architect | 즉시 | App Shell 및 라우팅 가드 설계 전 확정 필수 |
| 3 | **DEC-002 (Error Message)** | Frontend Lead | Sprint 1 1주차 내 | 컴포넌트 오류 렌더링 방식 기준 확립 |
| 4 | **DEC-003 (Storage)** | DBA / Backend Lead | Sprint 1 1주차 내 | DB 스키마 생성 및 S3 설정 전 확정 |

## 10. Definition of Done
- [x] 기존 명세서 간의 불일치(Pagination 등)를 식별하였는가?
- [x] 미확정 설계 항목에 대해 구체적 선택지와 장단점을 비교 제시하였는가?
- [x] 권장안 채택 시 수정해야 할 문서를 정확히 매핑하였는가?
- [x] Sprint 1 개발 착수에 필요한 가장 중요한 결정 사항들을 도출하였는가?
