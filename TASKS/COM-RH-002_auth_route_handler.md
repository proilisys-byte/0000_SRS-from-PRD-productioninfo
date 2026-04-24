---
name: Feature Task
about: PRD&SRS 기반의 구체적인 개발 태스크 명세
title: "[API] COM-RH-002: Authentication Route Handler"
labels: 'api, rh, spec, priority:high'
assignees: ''
---

## 🎯 Summary
- 기능명: [COM-RH-002] Authentication Route Handler
- 목적: 플랫폼 진입을 위한 인증(Login, Logout, Token Refresh, Me) API를 구현하여, 모든 후속 기능(Bulk Import, Smart Audit)이 Tenant 컨텍스트를 획득할 수 있도록 한다. Next.js App Router의 Route Handlers를 활용하여 구현되며, 클라이언트 컴포넌트나 모바일 뷰에서 호출하기 위한 API 명세이다.

## 🔗 References (Spec & Context)
> 작업 시작 전 아래 문서를 반드시 먼저 읽고 정합성을 유지할 것.
- PRD/SRS 문서: `00_PRD_v1.md`, `05_SRS_v1.md`
- 공통 인증 체계: `COM-AUTH_v1.md`
- 공통 에러 스키마: `API-001_common_error_schema.md`
- 데이터 스키마: `DATA-SCHEMA_v1.md`

## 🧭 Scope
### In Scope
- `POST /api/auth/login`: ID/Password 기반 검증 및 Supabase Auth 토큰 발급
- `POST /api/auth/logout`: 현재 세션 무효화 및 쿠키 삭제
- `GET /api/auth/me`: 현재 유저 프로필 정보 및 RBAC 역할 반환
- `POST /api/auth/refresh`: 만료된 액세스 토큰 갱신
- 토큰 페이로드 내 `tenant_id`, `role` 주입 로직 및 Tenant Context 추출

### Out of Scope
- OAuth/SSO 통합 (Sprint 2+)
- 비밀번호 찾기/재설정 UI 로직
- 사용자 계정 생성 (Admin 도메인)

## 🧱 Preconditions
- 선행 완료 문서:
  - `COM-AUTH_v1.md`
- 선행 의존성:
  - DB 스키마(`users`, `tenants`)
- 시작 가능 조건:
  - 인증 정책이 확립되어 API 규격으로 구체화할 준비 완료

## 📄 Detailed API Specification

### 1. `POST /api/auth/login`

**Request Body (Zod 스키마)**
```typescript
import { z } from 'zod';

export const LoginSchema = z.object({
  email: z.string().email("유효한 이메일 주소를 입력해주세요."),
  password: z.string().min(8, "비밀번호는 최소 8자 이상이어야 합니다."),
});
export type LoginRequest = z.infer<typeof LoginSchema>;
```

**Response (성공)**
```json
{
  "success": true,
  "data": {
    "user": {
      "id": "uuid",
      "email": "user@example.com",
      "role": "admin",
      "tenant_id": "uuid"
    }
  }
}
```

**Response (실패)**
```json
{
  "success": false,
  "error": {
    "code": "AUTH_401_UNAUTHORIZED_ACCESS",
    "message": "이메일 또는 비밀번호가 일치하지 않습니다."
  }
}
```

**에러 코드 매핑 (`API-001_common_error_schema.md` 연동)**
* `VAL_400_VALIDATION_FAILED`: 잘못된 요청 형식 (Zod 검증 실패)
* `AUTH_401_UNAUTHORIZED_ACCESS`: 인증 실패 (비밀번호 불일치 등)
* `RATE_429_EXTERNAL_RATE_LIMIT` / `RATE_429_TOO_MANY_REQUESTS`: Rate Limiting 초과

**Supabase Auth 연동 코드 스니펫**
```typescript
import { createRouteHandlerClient } from '@supabase/auth-helpers-nextjs';
import { cookies } from 'next/headers';
import { NextResponse } from 'next/server';
import { LoginSchema } from '@/lib/schemas/auth';
import { handleRouteError, AppError } from '@/lib/errors';

export async function POST(request: Request) {
  try {
    const body = await request.json();
    const { email, password } = LoginSchema.parse(body);

    const supabase = createRouteHandlerClient({ cookies });
    
    const { data, error } = await supabase.auth.signInWithPassword({
      email,
      password,
    });

    if (error) {
      throw new AppError('AUTH_401_UNAUTHORIZED_ACCESS', '이메일 또는 비밀번호가 일치하지 않습니다.', 401);
    }

    return NextResponse.json({ success: true, data: { user: data.user } });
  } catch (error) {
    return handleRouteError(error);
  }
}
```

### 2. `POST /api/auth/logout`
**Response (성공)**: `{"success": true}`
* 로직: Supabase `auth.signOut()`을 호출하여 서버 측 세션을 종료하고 클라이언트의 토큰 쿠키를 삭제합니다.

### 3. `GET /api/auth/me`
**Response (성공)**: 토큰에서 추출된 유저 정보 및 `app_metadata` 내의 role을 JSON으로 반환.

### 4. `POST /api/auth/refresh`
* 로직: 클라이언트의 `refresh_token`을 기반으로 새로운 세션 토큰 발급. Supabase Auth Helpers가 자동으로 쿠키를 갱신합니다.

## 🏢 Tenant Context 설정 방식

1. **로그인 시 tenant_id 주입 방안**
   * 데이터베이스 트리거에 의해 사용자 생성 시 JWT `app_metadata`에 `tenant_id` 및 `role`이 포함되어 있습니다.
   * `signInWithPassword` 성공 시 발급되는 세션 JWT 내부에 이 값들이 존재하므로 추가 DB 쿼리 없이 쿠키에 자동 저장 및 전달됩니다.

2. **API 요청 시 tenant_id 추출 공통 헬퍼 함수**
```typescript
// lib/auth/get-tenant.ts
import { createServerComponentClient } from '@supabase/auth-helpers-nextjs';
import { cookies } from 'next/headers';

export async function getTenantContext() {
  const supabase = createServerComponentClient({ cookies });
  const { data: { session } } = await supabase.auth.getSession();
  
  if (!session?.user) return null;
  
  const tenantId = session.user.app_metadata?.tenant_id;
  const role = session.user.app_metadata?.role;
  
  return { tenantId, role, user: session.user };
}
```

## ✅ Task Breakdown (실행 계획)
- [ ] Zod 요청 Body 검증 DTO 작성 (email, password 등) 및 `LoginSchema` 적용
- [ ] `createRouteHandlerClient`를 활용한 Supabase Auth 연동 로그인/로그아웃 로직 구현
- [ ] JWT 페이로드 내 `app_metadata`(`tenant_id`, `role`) 추출 및 반환 로직 구현
- [ ] Secure HTTP-Only Cookie에 토큰 저장 처리 (Supabase Auth Helpers 자동화 점검)
- [ ] Rate Limiting (Upstash Redis) 및 CSRF 방어 미들웨어 필터링 적용
- [ ] 인증 실패 예외 처리 및 `handleRouteError`를 이용한 공통 에러 응답 연결

## 🧪 Acceptance Criteria (BDD / GWT)
Scenario 1: 정상 로그인 성공
- Given: 올바른 이메일과 패스워드가 제공되었다.
- When: 로그인 API를 호출한다.
- Then: 200 OK와 함께 쿠키에 JWT 토큰이 발급되고, 응답 바디에 사용자의 `tenant_id`와 `role` 정보가 포함된다.

Scenario 2: 인증 실패 (비밀번호 불일치)
- Given: 잘못된 패스워드가 제공되었다.
- When: 로그인 API를 호출한다.
- Then: 401 Unauthorized 에러가 `API-001` 규격(`AUTH_401_UNAUTHORIZED_ACCESS`)에 맞춰 반환된다.

Scenario 3: 로그아웃 성공
- Given: 활성화된 로그인 세션 쿠키가 존재한다.
- When: 로그아웃 API를 호출한다.
- Then: 서버 세션이 무효화되고 클라이언트 쿠키가 삭제되며 200 OK가 반환된다.

Scenario 4: CSRF 및 Rate Limit 방어
- Given: 허용되지 않은 Origin에서 로그인 요청을 하거나 단시간 내 5회 이상 요청이 발생한다.
- When: 로그인 API를 호출한다.
- Then: 403 Forbidden(CSRF) 또는 429 Too Many Requests가 반환된다.

Scenario 5: 권한 정지 계정 접근
- Given: 비활성화(Inactive) 처리된 사용자 계정이다.
- When: 로그인 API를 호출한다.
- Then: 403 Forbidden 에러가 반환되며 접근 사유(Account Suspended)가 메시지에 포함된다.

## 🔐 Technical / Domain Constraints
- 권한/RBAC 규칙:
  - 로그인 성공 시 반환되는 User 객체 및 쿠키 JWT 페이로드에 반드시 `role`과 `tenant_id` 정보가 포함되어야 함.
- Rate Limiting:
  - 인증 엔드포인트 무차별 대입(Brute Force) 공격 방지를 위해 `@upstash/ratelimit` 적용 (1분당 최대 5회 시도 제한, 초과 시 `429 Too Many Requests` 반환).
- CSRF 방어 전략:
  - Route Handlers는 `Origin` 및 `Referer` 헤더가 화이트리스트(자사 도메인)와 일치하는지 미들웨어 레벨에서 사전 필터링 필수.
- HttpOnly 쿠키 설정:
  - Supabase Auth Helpers는 기본적으로 `HttpOnly`, `Secure`, `SameSite=Lax` 속성으로 쿠키를 발행. 이를 통해 XSS 공격 시 토큰 탈취 원천 차단.
- 공통 에러 연계:
  - 에러 처리 시 `API-001_common_error_schema.md`에 정의된 `AppError` 및 `handleRouteError` 활용 강제.

## 📦 Deliverables
- 산출 문서/코드/테스트/Mock/연계 결과:
  - `app/api/auth/[route]/route.ts` 하위 라우트 핸들러 코드 작성
  - `lib/schemas/auth.ts`에 Zod 스키마 작성
  - `lib/auth/get-tenant.ts` 공통 헬퍼 함수 작성

## 🏁 Definition of Done (DoD)
- [ ] PRD/SRS 및 관련 기존 TASK 문서와 충돌하지 않는가?
- [ ] Acceptance Criteria를 모두 통과하는가?
- [ ] 에러 반환 시 `handleRouteError` 포맷을 100% 준수하는가?
- [ ] Supabase Auth 기반 토큰이 HttpOnly 속성으로 안전하게 저장되는가?
- [ ] Rate Limiting 및 CSRF 방어 로직이 적용되어 있는가?

## 🚧 Dependencies & Blockers
- Depends on:
  - `COM-AUTH_v1.md`
  - `API-001_common_error_schema.md`
- Blocks:
  - `UI-001_login_page.md`
  - 모든 인가(Authorization)가 필요한 Route Handler 및 Server Action

## 📝 Notes for Dev / AI Agent
- 이 TASK의 핵심 초점:
  - 보안과 토큰 발급의 안정성이다. 향후 모든 API의 인증 미들웨어가 이 토큰에 의존한다.
- 구현 전 주의사항:
  - Access Token 탈취 방지를 위해 수명을 짧게 하고 Refresh Token을 적극 활용할 것.
- 관련 문서와의 경계:
  - 이 문서는 Backend API 명세이며, 클라이언트 폼 상태 관리 및 에러 토스트 노출 등의 UI 로직은 `UI-001`에서 담당한다.
