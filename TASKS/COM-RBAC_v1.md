# COM-RBAC_v1.md

## 1. Admin / User 2단계 권한 매트릭스 테이블

고객사(Tenant) 내부의 논리적 권한은 최고 관리자인 `Admin`과 현장 작업자인 `User`로 이원화됩니다.

| 기능 도메인 | Admin (품질팀장 등) | User (현장 반장 등) | 비고 |
| :--- | :---: | :---: | :--- |
| **Bulk Import (기준정보 업로드)** | O | X | 템플릿 업로드 및 구조 변경 권한 |
| **STT Zero-UI 데이터 입력** | O | O | 현장 작업 본연의 핵심 권한 |
| **Smart Audit 리포트 생성/조회** | O | O | 생성된 PDF의 열람 및 다운로드 |
| **긴급 NC 시정 자동화 처리** | O | X | 외부 감사에 대응하는 법적/공식적 조치 |
| **Lean/COPQ 대시보드 조회** | O | X | ROI 및 재무적 가치가 포함된 경영 정보 |
| **사용자/테넌트 관리** | O | X | 권한 부여 및 시스템 환경 설정 |

## 2. Next.js App Router 기준 라우트 보호 미들웨어 구조

페이지 렌더링 전, Edge 단위에서 권한을 검증하고 리다이렉트 처리하는 `middleware.ts` 예시 구조입니다.

```typescript
// middleware.ts
import { createMiddlewareClient } from '@supabase/auth-helpers-nextjs';
import { NextResponse } from 'next/server';
import type { NextRequest } from 'next/server';

export async function middleware(req: NextRequest) {
  const res = NextResponse.next();
  const supabase = createMiddlewareClient({ req, res });
  const { data: { session } } = await supabase.auth.getSession();

  const currentPath = req.nextUrl.pathname;

  // 1. 비로그인 사용자 리다이렉트
  if (!session && currentPath.startsWith('/dashboard')) {
    return NextResponse.redirect(new URL('/login', req.url));
  }

  // 2. RBAC 클레임 파싱 (JWT의 app_metadata 활용)
  const userRole = session?.user?.app_metadata?.role || 'user';

  // 3. Admin 전용 라우트 보호 로직
  const adminOnlyRoutes = ['/dashboard/bulk-import', '/dashboard/nc', '/dashboard/copq', '/dashboard/settings'];
  
  if (adminOnlyRoutes.some(route => currentPath.startsWith(route))) {
    if (userRole !== 'admin') {
      // 권한 없음 페이지로 리다이렉트
      return NextResponse.redirect(new URL('/dashboard/unauthorized', req.url));
    }
  }

  return res;
}

export const config = {
  matcher: ['/dashboard/:path*'], // 보호할 라우트 경로 지정
};
```

## 3. Supabase Auth와 RBAC 연동 방식 (JWT Claims 커스텀 방법)

1. **DB 내 프로필 테이블 트리거 연동**: 
   * 사용자가 Supabase Auth (`auth.users`)에 가입될 때 PostgreSQL Trigger를 발동시켜 public 스키마의 `users` 테이블에 프로필 레코드를 생성합니다.
2. **JWT Claims 커스텀 주입**: 
   * `users` 테이블의 권한(role)이 변경될 때 발동하는 별도의 트리거 또는 Edge Function을 구성합니다.
   * Supabase의 내부 프로시저인 `auth.jwt()`를 후킹하여 `app_metadata` 내부에 `{"role": "admin", "tenant_id": "uuid..."}` 형태의 커스텀 클레임(Claims)을 강제 주입합니다.
3. **인증 토큰 동기화**:
   * 이 방식을 채택하면 클라이언트는 추가적인 DB 쿼리 없이, 발급된 액세스 토큰(JWT)만 디코딩하여 자신이 속한 테넌트와 권한(`Admin` / `User`)을 확인할 수 있어, 라우터 미들웨어의 지연(Latency)을 최소화합니다.
