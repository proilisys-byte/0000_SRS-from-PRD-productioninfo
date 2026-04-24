---
name: Feature Task
about: SRS 기반의 구체적인 개발 태스크 명세
title: "[Feature] AUTH-C-001: Supabase Auth 기반 회원가입·로그인 Server Action 구현"
labels: 'feature, backend, auth, priority:critical, sprint:1'
assignees: ''
---

## 🎯 Summary
- 기능명: [AUTH-C-001] Supabase Auth 기반 회원가입·로그인 Server Action 구현
- 목적: 사용자가 시스템에 안전하게 접근하기 위한 이메일 기반 회원가입 및 로그인 기능을 Supabase Auth 서비스와 연동하여 Next.js Server Action으로 구현한다. 이 태스크는 모든 Command 태스크의 인증 기반이 되며, 후속 패스워드 정책(AUTH-C-002), 계정 잠금(AUTH-C-003), 세션 관리(AUTH-C-004), RBAC(AUTH-C-005)의 선행 조건이다.

## 🔗 References (Spec & Context)
> 💡 AI Agent & Dev Note: 작업 시작 전 아래 문서를 반드시 먼저 Read/Evaluate 할 것.
- SRS 문서: [`05_SRS_v1.md#§4.2.4.1`] — 인증 요구사항 (MFA, SSO, 패스워드 정책, 세션 관리, 계정 잠금)
- SRS 문서: [`05_SRS_v1.md#§4.2.4.2`] — 인가 요구사항 (RBAC 2단계: Admin/User)
- SRS 문서: [`05_SRS_v1.md#§4.3.6`] — 데이터 무결성 시퀀스 (RBAC 권한 관리 흐름)
- SRS 문서: [`05_SRS_v1.md#§1.2.3`] — 기술 제약 (C-TEC-003: Supabase Auth 통합)
- SRS 문서: [`05_SRS_v1.md#§2`] — 이해관계자 역할 정의 (Admin, User)
- API 명세 참조: API-001 (Auth 도메인 DTO)
- 태스크 목록: [`TASKS/06_TASK_LIST_v1.md#Epic4`] — Auth & RBAC 태스크 목록
- 프로젝트 컨텍스트: [`PROJECT_CONTEXT.md`] — 디렉토리 구조, 환경 변수, 코딩 컨벤션

## 📁 Implementation Paths
- `src/app/actions/auth/signUp.ts` — 회원가입 Server Action
- `src/app/actions/auth/signIn.ts` — 로그인 Server Action
- `src/app/actions/auth/signOut.ts` — 로그아웃 Server Action
- `src/middleware.ts` — 인증 미들웨어
- `src/lib/supabase/server.ts` — 서버사이드 Supabase 클라이언트
- `src/lib/validations/auth.ts` — Zod 입력 검증 스키마
- `src/types/dto/auth.ts` — Auth DTO 타입 정의

## ✅ Task Breakdown (실행 계획)
- [ ] Supabase Auth 클라이언트 초기화 (`@supabase/supabase-js`, `@supabase/ssr`)
- [ ] Server-side Supabase 클라이언트 설정 (쿠키 기반 세션 관리)
- [ ] 회원가입 Server Action 구현 (`signUp.ts`)
  - Zod 입력 검증 (email, password, name, role)
  - `supabase.auth.signUp()` 호출
  - USER 테이블에 추가 프로필 정보 저장 (user_id, tenant_id, role 기본값 "User")
  - 이메일 확인 플로우 설정 (Supabase 이메일 인증)
- [ ] 로그인 Server Action 구현 (`signIn.ts`)
  - Zod 입력 검증 (email, password)
  - `supabase.auth.signInWithPassword()` 호출
  - 세션 쿠키 설정 및 리다이렉트
- [ ] 로그아웃 Server Action 구현 (`signOut.ts`)
  - `supabase.auth.signOut()` 호출
  - 세션 쿠키 제거 및 로그인 페이지 리다이렉트
- [ ] 인증 미들웨어 구현 (`middleware.ts`)
  - 보호된 라우트 접근 시 세션 유효성 자동 검증
  - 미인증 사용자 로그인 페이지 리다이렉트
  - 세션 갱신(refresh) 로직
- [ ] 에러 처리: API-007 공통 에러 코드 체계 연동 (AUTH_EXPIRED, AUTH_INVALID 등)
- [ ] 단위 테스트 + 통합 테스트 (회원가입 → 로그인 → 세션 확인 → 로그아웃 E2E)

## 🧪 Acceptance Criteria (BDD/GWT)

**Scenario 1: 정상적인 회원가입**
- Given: 유효한 이메일(`newuser@proili.com`)과 패스워드(`SecureP@ss123!`)가 주어짐
- When: 회원가입 Server Action을 호출함
- Then: Supabase Auth에 사용자가 등록되고, USER 테이블에 프로필 레코드(role="User")가 생성되며, 이메일 인증 안내가 발송된다.

**Scenario 2: 중복 이메일 가입 시도**
- Given: 이미 Supabase Auth에 등록된 이메일(`exist@proili.com`)이 주어짐
- When: 해당 이메일로 회원가입을 시도함
- Then: 409 Conflict와 `code: "AUTH_DUPLICATE_EMAIL"` 에러를 반환하며, 기존 계정 정보가 노출되지 않는다.

**Scenario 3: 정상 로그인 및 세션 생성**
- Given: 등록 완료 및 이메일 인증된 사용자 계정이 존재함
- When: 올바른 이메일/패스워드로 로그인을 요청함
- Then: Supabase 세션이 생성되고, httpOnly 쿠키에 세션 토큰이 저장되며, 대시보드 페이지로 리다이렉트된다.

**Scenario 4: 잘못된 패스워드로 로그인 시도**
- Given: 등록된 이메일과 잘못된 패스워드가 주어짐
- When: 로그인을 시도함
- Then: 401 Unauthorized와 `code: "AUTH_INVALID_CREDENTIALS"` 에러를 반환하며, 어떤 필드가 틀렸는지 구체적으로 노출하지 않는다.

**Scenario 5: 미인증 상태에서 보호된 라우트 접근**
- Given: 로그인하지 않은 상태에서 `/dashboard` 페이지에 접근함
- When: 인증 미들웨어가 세션을 확인함
- Then: `/login` 페이지로 리다이렉트되며, 원래 접근하려던 URL이 `redirectTo` 파라미터로 보존된다.

**Scenario 6: 로그아웃 후 세션 무효화**
- Given: 로그인 상태의 사용자가 있음
- When: 로그아웃 Server Action을 호출함
- Then: Supabase 세션이 무효화되고, 쿠키가 제거되며, 이후 보호된 라우트 접근 시 로그인 페이지로 리다이렉트된다.

## ⚙️ Technical & Non-Functional Constraints
- 프레임워크: Next.js App Router Server Action + Supabase Auth (C-TEC-001, C-TEC-002, C-TEC-003)
- 세션 관리: httpOnly 쿠키 기반, 서버 사이드 세션 검증 필수
- 보안: 패스워드 평문 로깅 절대 금지, 에러 메시지에서 계정 존재 여부 유추 불가
- 비용: Supabase Free Tier MAU 제한(50,000명) 내 운영 (REQ-NF-018)
- RBAC 준비: USER 테이블 role 필드 기본값 "User", Admin 승격은 AUTH-C-005에서 구현
- 타입 안전: TypeScript strict 모드, Zod 스키마 검증

## 🏁 Definition of Done (DoD)
- [ ] 모든 Acceptance Criteria를 충족하는가?
- [ ] 회원가입·로그인·로그아웃 Server Action이 정상 동작하는가?
- [ ] 인증 미들웨어가 보호된 라우트를 올바르게 차단하는가?
- [ ] 단위 테스트 및 E2E 통합 테스트가 추가되었고 통과하는가?
- [ ] 에러 응답이 API-007 공통 에러 코드 체계를 준수하는가?
- [ ] 패스워드 등 민감정보가 로그에 평문으로 기록되지 않는가?
- [ ] ESLint / 정적 분석 경고가 없는가?

## 🚧 Dependencies & Blockers
- **Depends on:** DB-002 (USER 테이블 스키마), API-001 (Auth 도메인 DTO)
- **Blocks:** AUTH-C-002 (패스워드 정책), AUTH-C-003 (계정 잠금), AUTH-C-004 (세션 관리), AUTH-C-005 (RBAC 권한 변경), UI-001 (로그인 화면), 모든 Command 태스크 (인증 기반 필요)
