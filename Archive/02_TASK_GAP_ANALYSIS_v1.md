# 02_TASK_GAP_ANALYSIS_v1.md

# 02_TASK_GAP_ANALYSIS_v1: Task Gap Analysis & Next Steps

## 1. 문서 목적
본 문서는 PRO ILI SMART 플랫폼의 PRD 및 SRS를 기준으로, 현재 도출된 TASKS 문서 체계의 완결성을 평가하고 개발 착수를 위해 보완/추가되어야 할 태스크를 식별하는 데 목적이 있다. 이를 통해 병목 현상을 사전에 방지하고 프론트엔드, 백엔드, QA 팀이 즉시 다음 개발 단계를 무중단으로 진행할 수 있는 구체적인 가이드를 제공한다.

## 2. 분석 대상 및 전제
- **분석 대상 문서**: 현재 `e:\0000_SRS-from-PRD-productioninfo\TASKS` 내에 작성 완료된 상위 기준 문서, Sprint 1 공통 기준 문서, Bulk Import 문서군, Smart Audit F1 문서군
- **진행 상태 요약**: Bulk Import 및 Smart Audit 핵심 로직(Command, Query, DTO, RH, NFR) 문서의 90% 이상이 작성되어 기능 명세는 매우 탄탄한 상태.
- **처리 원칙**: 
  1. 명확히 완료된 기능의 중복 문서 생성은 배제한다.
  2. 개발(특히 UI 및 초기 셋업) 착수 시 블로커(Blocker)가 될 수 있는 누락 요소(공통 레이아웃, 시드 데이터, 목록 화면 등)를 최우선으로 발굴한다.

## 3. 현재 TASKS 인벤토리 요약

| 구분 | 문서 수/상태 | 판단 | 비고 |
|---|---:|---|---|
| 상위 기준 문서 | 6개 (PRD, SRS 등) | 닫힘 | 아키텍처 및 요구사항 기준 확립 완료 |
| Sprint 1 공통 기준 | 7개 (AUTH, SCHEMA 등) | 보완 필요 | 인증 정책은 있으나, 구체적 구현 명세(Login UI/API) 부족 |
| Bulk Import | 11개 (ADM, API, UI 등) | 닫힘 | End-to-End 명세(UI~DB~NFR) 완비 |
| Smart Audit Core | 5개 (F1-C, F1-Q, RH, DTO) | 닫힘 | 핵심 백엔드/API 규격 완비 |
| UI | 3개 (UI-010, UI-061, ADM-062) | 보완 필요 | 공통 레이아웃 및 F1 목록 조회(List) 화면 누락 |
| TEST / MOCK | 4개 / 2개 | 보완 필요 | F1 목록 조회를 위한 Mock API 누락 |
| NFR (Monitoring 등) | 4개 | 닫힘 | Bulk Import 및 Smart Audit 운영 기준 완비 |

## 4. 보완/누락 갭 분석

| 카테고리 | 현재 상태 | 갭 내용 | 영향도 | 처리 방향 |
|---|---|---|---|---|
| **공통/레이아웃** | 없음 | App Shell(GNB, LNB, Tenant Switcher) 정의 누락 | High | **신규 생성** |
| **공통/인증** | 정책 문서(`COM-AUTH_v1`) 존재 | 실제 Login 화면 및 세션 발급 API(RH) 구현 문서 누락 | High | **신규 생성** |
| **UI/화면** | `UI-010`(Workspace) 존재 | 세션을 검색/필터링하는 `Audit Session List` 화면 누락 | High | **신규 생성** |
| **데이터/테스트** | SCHEMA 문서 존재 | Smart Audit 테스트를 위한 Seed Data 스크립트 누락 | Medium | **신규 생성** |
| **Mock/API** | Report Mock(`MOCK-002`) 존재 | List Query 결과를 반환할 Mock Endpoint 누락 | Medium | **신규 생성** |
| **기존 문서 보완** | `F1-Q-001` 존재 | UI 목록 컴포넌트와의 매핑(Pagination, Filter) 설명 부족 | Low | **보완** |

## 5. 다음 개발 단계용 우선 TASKS 제안

| 우선순위 | 제안 파일명 | 카테고리 | 목적 | 관련 PRD/SRS 근거 | 선행 태스크 | 복잡도(H/M/L) | 상태 |
|---|---|---|---|---|---|---|---|
| 1 | `UI-000_app_shell_layout.md` | UI | 테넌트/사이드바 등 공통 UI 프레임 확립 | PRD (Multi-tenant UI) | `COM-AUTH_v1.md` | M | 신규 필요 |
| 2 | `COM-RH-002_auth_route_handler.md` | API/RH | 로그인, 토큰 발급, Tenant Context 설정 API 명세 | SRS (Auth) | `COM-AUTH_v1.md` | H | 신규 필요 |
| 3 | `UI-001_login_page.md` | UI | 로그인 및 초기 권한 라우팅 화면 | PRD (Auth) | `COM-AUTH_v1.md` | L | 신규 필요 |
| 4 | `UI-011_audit_session_list.md` | UI | Smart Audit 세션 목록 조회, 검색, 필터링 화면 | SRS (F1 조회 기능) | `F1-Q-001_...` | M | 신규 필요 |
| 5 | `MOCK-003_audit_list_mock_endpoint.md`| MOCK | `UI-011` 개발을 위한 Pagination Mock 데이터 제공 | SRS (F1 병렬 개발) | `API-003_...` | L | 신규 필요 |
| 6 | `DATA-012_smart_audit_seed_data.md` | DATA | F1 통합 테스트 및 개발용 가상 데이터셋 셋업 | TEST-S1_ACCEPTANCE | `DATA-SCHEMA_v1.md` | M | 신규 필요 |
| 7 | `F1-Q-001_audit_report_query.md` | F1-Q | 페이지네이션 및 상태 필터링 파라미터 규격 구체화 | SRS (F1 조회 기능) | - | L | 보완 필요 |

## 6. 즉시 실행 권장 배치

| 배치 | 포함 TASKS | 이유 | 선행 조건 |
|---|---|---|---|
| **1차 배치** (Foundation) | `UI-000`, `UI-001`, `COM-RH-002` | 모든 도메인(Bulk, F1)이 공통으로 사용할 앱 쉘과 로그인 진입점이 마련되어야 UI/프론트엔드 작업이 블록되지 않음 | 프로젝트 Context 이해 |
| **2차 배치** (F1 Frontend) | `UI-011`, `MOCK-003` | 이미 백엔드 로직(`F1-Q-001`)은 있으므로, 이를 소비할 목록 화면 UI와 병렬 개발용 Mock이 필요함 | 1차 배치 UI 구조 |
| **3차 배치** (QA & Data) | `DATA-012`, `F1-Q-001`(보완) | 실제 통합 테스트 수행을 위한 데이터 기반 마련 및 쿼리 파라미터 정교화 | 스키마 안정화 |

## 7. 보완 필요 문서 상세 메모
- `F1-Q-001_audit_report_query.md`: 쿼리 모델 자체는 정의되었으나, `UI-011`에서 사용할 검색 필터(상태별, 기간별, 담당자별) 및 페이지네이션(Cursor vs Offset) 스키마가 다소 추상적일 수 있음. DTO 수준에서 검색 조건을 명확히 하는 보완(Revision)이 권장됨.

## 8. Sprint 우선순위 판단
- **Sprint 1 즉시 대상**: `UI-000` (App Shell), `UI-001` (Login), `COM-RH-002` (Auth API) -> **가장 시급한 병목 지점**
- **Sprint 1 전반 대상**: `UI-011` (Audit List), `MOCK-003`
- **Sprint 1 후반 대상**: `DATA-012` (Seed Data)
- **Sprint 2 이관 대상**: NC 시정조치용 UI/API 명세는 현재 갭 분석에서는 의도적으로 배제함 (Sprint 1 범위에 맞춤)

## 9. Definition of Done for TASK Extraction
- [x] 현재 문서 폴더의 모든 파일을 검토하여 카테고리별 매핑 완료
- [x] PRD/SRS 요구사항 중 프론트엔드/테스트 진입을 막는 누락(블로커) 항목 식별 완료
- [x] 새로 생성해야 할 태스크들의 파일명을 명명 규칙(UI-, MOCK-, DATA- 등)에 맞게 정의 완료
- [x] 기존 완료 문서와 중복되는 태스크가 없음을 교차 검증 완료
- [x] 결과물이 구체적이고 바로 태스크 발행/개발 할당이 가능한 형태임을 확인

## 10. Gemini 3.1 Pro 자체 검토 체크리스트
- [x] 기존 완료 문서와 중복 추출하지 않았는가? (Bulk Import 및 F1 Core는 배제함)
- [x] 파일명 체계가 우리 프로젝트 규칙과 맞는가? (UI-xxx, COM-RH-xxx 등 체계 준수)
- [x] PRD/SRS 근거가 누락되지 않았는가? (Multi-tenant UI, Auth 규격 반영)
- [x] 개발 직결성이 높은 TASK가 상단에 배치되었는가? (로그인과 공통 레이아웃을 1순위로 배치)
- [x] 보완/신규/후순위 구분이 명확한가? (보완 1건, 신규 6건 식별)
- [x] TEST/Mock/NFR가 과도하거나 부족하지 않은가? (UI 병렬 개발을 위한 필수 Mock/Seed만 추출)
- [x] Sprint 2 이상이 무리하게 들어오지 않았는가? (철저히 Sprint 1의 Auth/F1 List에 집중함)
