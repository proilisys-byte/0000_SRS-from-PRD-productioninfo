---
name: Feature Task
about: SRS 기반의 구체적인 개발 태스크 명세
title: "[Feature] MOCK-006: 프론트엔드 개발용 Mocking API 엔드포인트 세팅"
labels: 'feature, frontend, mock, priority:high, sprint:0'
assignees: ''
---

## 🎯 Summary
- 기능명: [MOCK-006] 프론트엔드 개발용 Mocking API 엔드포인트 세팅 (MSW 또는 Next.js Route Handler)
- 목적: 백엔드 API 구현이 완료되기 전에 프론트엔드 개발을 병렬로 착수할 수 있도록, API-001~006에서 정의된 DTO 계약에 기반한 Mocking API를 구축한다. MSW(Mock Service Worker) 또는 Next.js Route Handler 기반의 모킹 레이어를 통해 실제 API와 동일한 요청/응답 구조를 제공하여, UI 개발 시 백엔드 의존성을 완전히 제거한다.

## 🔗 References (Spec & Context)
> 💡 AI Agent & Dev Note: 작업 시작 전 아래 문서를 반드시 먼저 Read/Evaluate 할 것.
- SRS 문서: [`05_SRS_v1.md#§3.3`] — API & Data Interaction Overview (6개 핵심 API)
- SRS 문서: [`05_SRS_v1.md#§6.1`] — API 엔드포인트 상세 명세 (존재 시)
- SRS 문서: [`05_SRS_v1.md#§1.2.3`] — 기술 제약 (C-TEC-002: Server Actions / Route Handlers)
- 관련 태스크: API-001~006 (도메인별 DTO 정의)
- 관련 태스크: API-007 (공통 에러 코드 체계)
- 관련 태스크: MOCK-001~005 (Seed/Fixture 데이터)
- 태스크 목록: [`TASKS/06_TASK_LIST_v1.md#Epic3`] — Mock Data 태스크 목록
- 프로젝트 컨텍스트: [`PROJECT_CONTEXT.md`] — 디렉토리 구조, 환경 변수, 코딩 컨벤션

> ⚠️ **ISSUE-04 반영 (DTO Draft 2단계 전략):**
> - **1단계 (Sprint 0A):** DB 스키마 초안 기반 DTO Draft 작성 → MOCK-006 선행 구성 가능
> - **2단계 (Sprint 0B):** DB 스키마 확정 후 DTO 확정 → MOCK-006 응답 타입 최종 정렬
> - AI 에이전트에게 지시 시 "DTO Draft 기반으로 먼저 구성" 명시 필요

## 📁 Implementation Paths
- `mock/handlers/auth.ts` — Auth 도메인 Mock 핸들러
- `mock/handlers/audit.ts` — Audit 도메인 Mock 핸들러
- `mock/handlers/nc.ts` — NC 도메인 Mock 핸들러
- `mock/handlers/lean.ts` — Lean 도메인 Mock 핸들러
- `mock/handlers/edge.ts` — Edge 도메인 Mock 핸들러
- `mock/handlers/templates.ts` — Template 도메인 Mock 핸들러
- `mock/data/` — MOCK-001~005 Seed 데이터 참조
- `mock/index.ts` — MSW 설정 엔트리포인트

## ✅ Task Breakdown (실행 계획)
- [ ] Mocking 전략 결정 및 초기 설정
  - 옵션 A: MSW (Mock Service Worker) — 브라우저/Node.js 양쪽 인터셉트
  - 옵션 B: Next.js Route Handler (`/api/mock/*`) — 서버 사이드 모킹
  - 환경 변수(`NEXT_PUBLIC_USE_MOCK=true`) 기반 모킹 On/Off 토글
- [ ] Auth 도메인 Mock API (API-001 기반)
  - `POST /api/v1/auth/signup` — 성공/중복 이메일 응답
  - `POST /api/v1/auth/login` — 성공/실패 응답 + 모킹 세션 토큰
  - `GET /api/v1/auth/session` — 현재 사용자 세션 정보
- [ ] Audit Report 도메인 Mock API (API-002 기반)
  - `POST /api/v1/audit/reports` — 리포트 생성 성공 응답 (스트리밍 모킹 포함)
  - `GET /api/v1/audit/reports/{report_id}` — 리포트 상세 조회 응답
- [ ] Zero-UI Edge Sync Mock API (API-003 기반)
  - `POST /api/v1/edge/sync` — 동기화 성공/실패 응답
  - `GET /api/v1/edge/devices/{id}/status` — 디바이스 상태 응답
- [ ] NC Response 도메인 Mock API (API-004 기반)
  - `POST /api/v1/nc/cases` — NC 등록 + 시정 초안 응답
  - `GET /api/v1/nc/cases/{nc_id}` — NC 상세 조회
  - `PATCH /api/v1/nc/cases/{nc_id}/actions/{action_id}` — 시정 진행률 갱신
- [ ] Lean Diagnosis Mock API (API-005 기반)
  - `POST /api/v1/lean/diagnose` — COPQ 분석 결과 응답
  - `GET /api/v1/lean/reports/{id}/export` — PDF Export 모킹
- [ ] Template Registry Mock API (API-006 기반)
  - `GET /api/v1/templates` — 템플릿 목록 (ISO9001/14001/45001)
  - `GET /api/v1/templates/{id}` — 템플릿 상세
  - `POST /api/v1/templates` — 커스텀 템플릿 등록
- [ ] 공통 에러 응답 모킹 (API-007 기반)
  - 각 엔드포인트별 400/401/403/404/409/429/500 에러 시나리오
  - 쿼리 파라미터(`?_error=401` 등)로 에러 시나리오 트리거 지원
- [ ] 응답 지연 시뮬레이션 (`?_delay=2000` 파라미터로 네트워크 지연 재현)
- [ ] Mock 데이터 소스 연결 (MOCK-001~005 Seed 데이터 활용)
- [ ] 개발용 문서 작성: Mock API 사용법 가이드 (README 또는 Storybook 연동)

## 🧪 Acceptance Criteria (BDD/GWT)

**Scenario 1: Mock 모드 활성화**
- Given: 환경 변수 `NEXT_PUBLIC_USE_MOCK=true`가 설정됨
- When: 프론트엔드에서 `POST /api/v1/audit/reports` API를 호출함
- Then: 실제 백엔드가 아닌 Mock 핸들러가 응답하며, 사전 정의된 Audit 리포트 JSON을 반환한다.

**Scenario 2: Mock 모드 비활성화 시 실제 API 라우팅**
- Given: 환경 변수 `NEXT_PUBLIC_USE_MOCK=false`가 설정됨
- When: 동일 API를 호출함
- Then: 실제 백엔드 Server Action/Route Handler로 요청이 전달된다.

**Scenario 3: 에러 시나리오 시뮬레이션**
- Given: Mock 모드가 활성화된 상태임
- When: `POST /api/v1/auth/login?_error=401`을 호출함
- Then: API-007 에러 코드 체계에 맞는 401 Unauthorized 응답을 반환한다.

**Scenario 4: 응답 지연 시뮬레이션**
- Given: Mock 모드가 활성화된 상태임
- When: `GET /api/v1/templates?_delay=3000`을 호출함
- Then: 3초 후에 정상 응답을 반환하여 로딩 UI를 테스트할 수 있다.

**Scenario 5: DTO 계약 일치성**
- Given: API-002에서 정의한 AuditReportResponse DTO가 있음
- When: Mock API의 `GET /api/v1/audit/reports/{id}` 응답을 해당 DTO 타입으로 파싱함
- Then: TypeScript 타입 검증이 통과하고, 모든 필수 필드가 포함되어 있다.

## ⚙️ Technical & Non-Functional Constraints
- 프레임워크: Next.js App Router (C-TEC-001), MSW 2.x 또는 Route Handler 기반
- 환경 분리: Mock 코드는 프로덕션 빌드에 포함되지 않아야 함 (tree-shaking)
- DTO 일치: Mock 응답은 반드시 API-001~006에서 정의한 TypeScript DTO 타입을 준수
- 에러 포맷: API-007 공통 에러 응답 구조 100% 준수
- Seed 데이터: MOCK-001~005에서 생성한 Fixture 데이터를 Mock 응답에 활용

## 🏁 Definition of Done (DoD)
- [ ] 모든 Acceptance Criteria를 충족하는가?
- [ ] 6개 도메인(Auth, Audit, Edge, NC, Lean, Template) Mock API가 구현되었는가?
- [ ] Mock 모드 On/Off 토글이 환경 변수로 제어 가능한가?
- [ ] 에러 시나리오 및 지연 시뮬레이션이 동작하는가?
- [ ] Mock 응답이 DTO 타입과 정확히 일치하는가?
- [ ] 프로덕션 빌드에서 Mock 코드가 제거되는가?
- [ ] 사용법 문서가 작성되었는가?
- [ ] ESLint / 정적 분석 경고가 없는가?

## 🚧 Dependencies & Blockers
- **Depends on:** API-001~006 (도메인별 DTO 정의 완료), API-007 (에러 코드 체계)
- **Blocks:** 전체 UI 태스크 (UI-001~014, EDGE-UI-001~005) — 프론트엔드 개발 착수 가능 조건
