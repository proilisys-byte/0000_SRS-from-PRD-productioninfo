---
name: Feature Task
about: PRD&SRS 기반의 구체적인 개발 태스크 명세
title: "[UI] UI-000: App Shell & Common Layout"
labels: 'ui, spec, priority:high'
assignees: ''
---

## 🎯 Summary
- 기능명: [UI-000] App Shell & Common Layout
- 목적: Multi-tenant 지원 B2B SaaS 플랫폼에 최적화된 공통 UI 프레임(GNB, LNB, Tenant Switcher)을 구축하여 이후 파생되는 모든 페이지 개발의 병목을 해소한다. Tailwind CSS와 shadcn/ui 기반으로 일관된 디자인 시스템을 구성한다.

## 🔗 References (Spec & Context)
> 작업 시작 전 아래 문서를 반드시 먼저 읽고 정합성을 유지할 것.
- PRD 문서: `00_PRD_v1.md`
- SRS 문서: `05_SRS_v1.md`
- 프로젝트 컨텍스트: `00_PROJECT_CONTEXT_v1.md`
- 공통 인증 문서: `COM-AUTH_v1.md`

## 🧭 Scope
### In Scope
- Global Navigation Bar (GNB)
- Local Navigation Bar (LNB / Sidebar)
- Tenant / Site Switcher 컴포넌트
- 공통 인증 상태 검증 연동 체계 (Auth Guard / Middleware)
- Next.js App Router 기반의 라우트 그룹 `(auth)`, `(dashboard)` 분리 적용

### Out of Scope
- 개별 페이지(Smart Audit, Bulk Import 등)의 본문 컨텐츠
- 로그인 페이지 자체 로직

## 🧱 Preconditions
- 선행 완료 문서:
  - `02_TASK_GAP_ANALYSIS_v1.md`
- 선행 의존성:
  - `COM-AUTH_v1.md`
- 시작 가능 조건:
  - 전역 상태(토큰, 테넌트) 관리를 위한 공통 인증 기준이 확립되어 프레임워크 셋업 가능

## 📄 Detailed Architecture Specification

### 1. 레이아웃 구조 (컴포넌트 구성 요소)
PRO ILI SMART 플랫폼은 단일 테넌트 B2B SaaS에 최적화된 레이아웃을 사용합니다. Tailwind CSS와 shadcn/ui 기반의 컴포넌트로 구성하여 빠르고 일관된 디자인 시스템을 구축합니다.

* **GNB (Global Navigation Bar)** 
  * 화면 상단 고정 (Sticky Header 적용)
  * 영역 구성: 좌측(로고), 중앙(접속 중인 테넌트 명칭), 우측(알림 배지, 유저 아바타 드롭다운 - 프로필 및 로그아웃)
* **LNB (Left Navigation Bar)** 
  * 화면 좌측 고정 (사이드바 메뉴)
  * RBAC 역할 기반 동적 렌더링: 로그인 사용자의 권한(`Admin` 또는 `User`)에 따라 접근할 수 없는 메뉴(Bulk Import, NC 시정, COPQ 등)는 아예 렌더링되지 않습니다.
* **메인 콘텐츠 영역 (`children` 슬롯)** 
  * GNB와 LNB를 제외한 핵심 작업 및 대시보드 출력 영역
  * App Router의 기본 동작 방식인 `children` prop을 통해 개별 페이지 컴포넌트가 주입됩니다.
* **디자인 토큰 명시**
  * Tailwind CSS의 기본 Utility Class만을 활용하여 커스텀 CSS는 최소화합니다.
  * 테이블, 버튼, 폼, 다이얼로그 등 핵심 UI 요소는 사전에 설치된 `shadcn/ui` 컴포넌트를 100% 재사용합니다.

### 2. 파일 구조 (App Router 기반)
`app/` 디렉터리 내부에 Route Groups 기능인 `(auth)` 및 `(dashboard)` 괄호 폴더 구조를 활용하여 비로그인과 로그인 후의 공통 레이아웃을 안전하고 명확히 분리합니다.

```text
app/
├── (auth)/
│   ├── layout.tsx         # 비로그인 전용 레이아웃 (LNB 없음, 중앙 정렬 등 최소한의 UI)
│   └── login/
│       └── page.tsx       # 로그인 폼 페이지
├── (dashboard)/
│   ├── layout.tsx         # 로그인 후 공통 쉘 (GNB + LNB 컴포넌트 포함, 메인 콘텐츠 감싸기)
│   ├── page.tsx           # 기본 랜딩 페이지 (권한에 따라 다른 대시보드로 분기)
│   ├── bulk-import/       # (Admin 전용 라우트)
│   ├── nc/                # (Admin 전용 라우트)
│   └── copq/              # (Admin 전용 라우트)
└── middleware.ts          # 전역 라우트 보호 및 권한 검증 미들웨어
```

### 3. 파일별 역할 상세
* **`app/(auth)/layout.tsx`**: 외부 사용자나 미인증 사용자가 접근하는 화면의 뼈대입니다. 네비게이션이 노출되지 않으며 중앙 집중형 인증 폼 UI를 구성합니다.
* **`app/(dashboard)/layout.tsx`**: 인증을 통과한 사용자들의 공통 애플리케이션 쉘을 담당합니다. 내부적으로 `<TopNav />`와 `<SideNav />` 컴포넌트를 가져오며, `getTenantContext()` 헬퍼를 통해 유저 권한 정보를 자식 컴포넌트에 전달합니다.
* **`app/middleware.ts`**: 애플리케이션 최상단에서 동작하여, `/dashboard/*` 경로 진입 시 세션 여부와 `app_metadata`의 권한(Role)을 확인합니다. User 권한인 사용자가 Admin 전용 라우트로 우회 진입을 시도할 경우, `/dashboard/unauthorized` 페이지로 리다이렉트하는 로직을 수행합니다.

## ✅ Task Breakdown (실행 계획)
- [ ] GNB 및 로고, 사용자 프로필 영역 컴포넌트 개발 (`<TopNav />`)
- [ ] 권한(RBAC)에 따른 사이드바(LNB) 메뉴 동적 렌더링 로직 구현 (`<SideNav />`)
- [ ] Tenant / Site Switcher 드롭다운 컴포넌트 개발 및 전역 상태 연동
- [ ] Next.js App Router Route Group `(auth)`, `(dashboard)` 폴더 구조 및 레이아웃 파일 세팅
- [ ] 화면 진입 시 Auth Token 검증을 위한 `middleware.ts` 전역 설정
- [ ] shadcn/ui 기반 반응형 레이아웃 처리(모바일/태블릿 대응)

## 🧪 Acceptance Criteria (BDD / GWT)
Scenario 1: 정상 레이아웃 렌더링
- Given: 사용자가 로그인 상태이며 특정 Tenant에 속해 있다.
- When: 루트 페이지에 접근한다.
- Then: 선택된 Tenant 이름이 Switcher에 표시되고, 해당 권한에 맞는 LNB 메뉴가 노출된다.

Scenario 2: 권한 없는 메뉴 접근 제한
- Given: 권한이 제한된 사용자(예: Site Viewer)가 로그인했다.
- When: App Shell이 로드된다.
- Then: System Admin 전용 메뉴(예: 전사 통계)는 사이드바에 렌더링되지 않는다.

Scenario 3: Tenant 변경 시 전역 상태 업데이트
- Given: 다중 Tenant에 속한 사용자가 사이드바 Switcher를 클릭한다.
- When: 다른 Tenant를 선택한다.
- Then: 전역 상태의 `tenant_id`가 변경되고 페이지가 새로고침되거나 데이터가 Re-fetch된다.

Scenario 4: 미인증/비인가 접근 차단
- Given: 비로그인 유저 또는 일반 권한 유저가 존재한다.
- When: 인증이 필요한 `(dashboard)` 라우트 혹은 관리자 전용 경로로 직접 진입을 시도한다.
- Then: `middleware.ts`가 이를 차단하고 로그인 화면이나 `/dashboard/unauthorized`로 리다이렉트 시킨다.

## 🔐 Technical / Domain Constraints
- 디자인 토큰:
  - Custom CSS 작성을 지양하고 Tailwind CSS Utility Classes와 shadcn/ui 기반 구축 필수
- 권한/RBAC 규칙:
  - `COM-AUTH_v1.md`에 정의된 역할에 따라 사이드바 메뉴 가시성 및 Middleware 진입 통제
- tenant/site 범위 규칙:
  - App Shell 레벨에서 현재 선택된 `tenant_id`와 `site_id`를 하위 페이지 컴포넌트로 주입해야 함
- 성능 / 운영 제약:
  - 화면 전환 시 깜빡임(FOUC) 방지, SSR/SSG 환경(Next.js 등) 호환성 고려

## 📦 Deliverables
- 산출 문서/코드/테스트/Mock/연계 결과:
  - `app/(auth)/layout.tsx`, `app/(dashboard)/layout.tsx` 
  - `app/middleware.ts` 
  - GNB/LNB 등 Layout Components (`<TopNav />`, `<SideNav />`)

## 🏁 Definition of Done (DoD)
- [ ] PRD/SRS 및 관련 기존 TASK 문서와 충돌하지 않는가?
- [ ] Acceptance Criteria를 모두 반영했는가?
- [ ] 상태/권한/tenant-site/에러/감사로그 기준이 누락되지 않았는가?
- [ ] `middleware.ts`를 활용한 권한 통제가 정상 작동하는가?

## 🚧 Dependencies & Blockers
- Depends on:
  - `COM-AUTH_v1.md`
- Blocks:
  - `UI-010_audit_workspace_page.md` (하위 페이지)
  - `UI-011_audit_session_list.md` (하위 페이지)
  - 모든 `(dashboard)` 그룹 내 하위 페이지 구현

## 📝 Notes for Dev / AI Agent
- 이 TASK의 핵심 초점:
  - 모든 UI의 뼈대가 되므로 테넌트 컨텍스트 관리와 Auth Middleware의 강건함이 가장 중요하다.
- 구현 전 주의사항:
  - 하드코딩된 메뉴를 피하고 권한/라우트 설정 배열 기반으로 동적 렌더링할 것.
- 관련 문서와의 경계:
  - App Shell은 '껍데기'와 '네비게이션'만 책임지며 내부 컨텐츠 로직은 철저히 배제한다.
