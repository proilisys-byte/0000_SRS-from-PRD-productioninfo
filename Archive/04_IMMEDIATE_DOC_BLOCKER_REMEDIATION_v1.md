# 04. 즉시 구현 블로커 문서 식별 및 보완 계획 (Immediate Doc Blocker Remediation)

## 1. 문서 목적
본 문서는 PRO ILI SMART 프로젝트의 다음 개발 단계에서 즉각적인 코드 구현을 가로막는 문서 레벨의 병목(Blocker)을 식별하고, 이를 해소하기 위한 우선 작성 및 보완 전략을 제시하는 데 목적이 있습니다. 실제 `TASKS` 폴더 내 파일 유무 및 내용 충실도를 기반으로 작성되었습니다.

## 2. 검토 기준 문서
- 필수 기반 문서: `00_PRD_v1.md`, `05_SRS_v1.md`, `02_TASK_GAP_ANALYSIS_v1.md`, `COM-AUTH_v1.md`
- 우선 점검 대상:
  - `COM-RH-002_auth_route_handler.md`
  - `UI-000_app_shell_layout.md`
  - `UI-001_login_page.md`
  - `UI-010_audit_workspace_page.md`
  - `MOCK-002_audit_report_mock_endpoint.md`
  - `TEST-F1-001_audit_session_flow_test.md`
  - `TEST-F1-002_audit_report_generation_test.md`

## 3. 즉시 블로커 판단 기준
- **미작성 (Missing):** 실제 디렉토리에 해당 파일이 존재하지 않아, 의존성 트리에 치명적인 구멍을 내는 경우.
- **보완 필요 (Weak Content):** 파일은 존재하나, DTO, 상태 관리 스키마, Validation 룰, 구체적 에러 매핑 등 개발자가 즉각적으로 코드로 전환할 수 있는 수준의 상세 내용이 부족한 경우. (일반적으로 10KB 미만의 추상적 명세)

## 4. 현재 블로커 문서 현황

| 분류 | 문서명 | 현재 상태 | 블로커 판단 사유 |
|---|---|---|---|
| 기반 명세 | `00_PRD_v1.md` | 미작성 | 비즈니스 요구사항 및 기능 정의의 원천 부재 |
| 기반 명세 | `05_SRS_v1.md` | 미작성 | 기술 요구사항 및 비기능 제약사항 원천 부재 |
| 기반 명세 | `02_TASK_GAP_ANALYSIS_v1.md` | 미작성 | 기존 분석 이력 추적 불가 |
| 공통 설계 | `COM-AUTH_v1.md` | 미작성 | 모든 인증/인가 로직의 공통 정책 및 세션 관리 기준 부재 |
| 구현 명세 | `COM-RH-002_auth_route_handler.md` | 보완 필요 | 상세 DTO, 토큰 만료 정책, 쿠키 보안 속성 및 DB 쿼리 명세 부족 |
| 구현 명세 | `UI-000_app_shell_layout.md` | 보완 필요 | 전역 상태 관리(Zustand) 스키마, 하위 라우팅 주입 방식 구체화 부족 |
| 구현 명세 | `UI-001_login_page.md` | 보완 필요 | 클라이언트 Validation 규칙(Zod), 에러 코드 매핑 테이블 부족 |
| 구현 명세 | `UI-010_audit_workspace_page.md` | 양호 | 기본 구조 및 내용 구비 (추가 검토 필요) |
| 구현 명세 | `MOCK-002_audit_report_mock_endpoint.md` | 보완 필요 | 테스트를 위한 상세 JSON 페이로드 구조 및 에러 케이스 Mock 부족 |
| 구현 명세 | `TEST-F1-001_audit_session_flow_test.md` | 보완 필요 | E2E 테스트 스텝 및 Assert 구문 레벨의 구체성 부족 |
| 구현 명세 | `TEST-F1-002_audit_report_generation_test.md` | 보완 필요 | 리포트 생성 검증 조건 및 Mock 데이터 연동 시나리오 부족 |

## 5. 즉시 작성/보완 필요 문서 목록

| 우선순위 | 문서명 | 조치 유형 | 영향도 |
|:---:|---|---|:---:|
| 1 | `00_PRD_v1.md` | 신규 작성 | Critical |
| 2 | `05_SRS_v1.md` | 신규 작성 | Critical |
| 3 | `COM-AUTH_v1.md` | 신규 작성 | Critical |
| 4 | `COM-RH-002_auth_route_handler.md` | 내용 보완 | High |
| 5 | `UI-001_login_page.md` | 내용 보완 | High |

## 6. 문서별 부족 내용 요약

| 문서명 | 부족 내용 및 병목 포인트 | 필요 조치 내용 |
|---|---|---|
| `00_PRD_v1.md` | 전체 기능 정의 및 페르소나 부재 | PRD 초안 생성 및 핵심 기능 목록 확정 |
| `05_SRS_v1.md` | 인프라, 보안, 아키텍처 제약사항 부재 | 기술 제약, NFR(비기능 요구사항) 구체화 |
| `COM-AUTH_v1.md` | 토큰 구조, RBAC 매트릭스, 세션 생명주기 정책 부재 | 인증 아키텍처 및 정책 수립 |
| `COM-RH-002_auth_route_handler.md` | 구체적인 입출력 DTO, JWT 서명 로직, 쿠키(HttpOnly) 옵션 누락 | Zod 스키마 연동 및 에러 반환 구조체 추가 |
| `UI-000_app_shell_layout.md` | GNB/LNB 동적 렌더링을 위한 전역 상태 스토어 명세 부재 | Zustand 등 상태 스토어 명세 및 Auth Guard Hook 명세 추가 |
| `UI-001_login_page.md` | Zod Validation 규칙, API 연동 시 에러 코드별 다국어 매핑 로직 누락 | 구체적 폼 스키마 정의 및 예외 처리 UI 플로우 추가 |
| `MOCK/TEST 문서` | E2E Assert 구문, 상태별 페이로드 상세 부재 | JSON 목업 및 Cypress/Playwright 명세 보완 |

## 7. 권장 작성 순서
1. **기반 확립 (Base Definition):** `00_PRD_v1.md` ➔ `05_SRS_v1.md`
2. **공통 아키텍처 (Core Architecture):** `COM-AUTH_v1.md`
3. **인증 파이프라인 (Auth Pipeline):** `COM-RH-002_auth_route_handler.md` ➔ `UI-001_login_page.md` ➔ `UI-000_app_shell_layout.md`
4. **품질 검증 명세 (QA & Mock):** Mock 데이터 및 E2E Test 명세 보완

## 8. 문서별 완료 기준

| 문서명 | 완료 기준 (Definition of Done) |
|---|---|
| 기반 문서 (PRD, SRS) | 프로젝트의 목적, 범위, 기술 스택, NFR이 명시되어 타 문서의 참조 기준이 됨 |
| 공통 문서 (COM-AUTH) | JWT Payload 스키마, RBAC 권한 표, 만료/갱신 시나리오가 확정됨 |
| 구현 명세 (RH, UI) | Zod/DTO 스키마, 명확한 상태 변화(State Transition), API-001 기반 에러 처리 로직이 코드 레벨로 구체화됨 |

## 9. 다음 단계 연결
- 본 블로커 해소 문서를 바탕으로 즉각적인 문서 작성/보완 태스크를 시작합니다.
- 기반 문서(`PRD`, `SRS`) 작성이 최우선으로 진행되어야 하며, 이후 인증 시스템에 대한 `COM-AUTH_v1.md`가 확정되어야 `COM-RH-002` 및 UI 프레임 구현이 가능합니다.

## 10. Definition of Done
- [x] TASKS 폴더 내 실제 파일을 점검하여 누락 및 부실 여부를 확인했는가?
- [x] PRD/SRS 범위를 벗어나지 않는 선에서 보완 필요 항목을 도출했는가?
- [x] 즉시 블로커 문서 리스트 및 권장 작성 순서가 명확하게 정리되었는가?
