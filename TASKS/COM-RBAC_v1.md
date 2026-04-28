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

---

## 4. Supabase RLS(Row Level Security) 정책 SQL

Admin/User 역할에 따라 데이터 접근을 DB 레벨에서 차단하는 Row-Level Security 정책입니다.

### A. 기본 테넌트 격리 정책 (모든 테이블 공통)
```sql
-- 사용자는 자신의 tenant_id에 해당하는 데이터만 조회 가능
CREATE POLICY "Tenant isolation" ON audit_sessions
FOR ALL
USING (
  tenant_id = (auth.jwt() ->> 'tenant_id')::uuid
);

-- 동일 정책을 audit_data_entries, bulk_import_batches 등 모든 비즈니스 테이블에 적용
CREATE POLICY "Tenant isolation" ON audit_data_entries
FOR ALL
USING (
  tenant_id = (auth.jwt() ->> 'tenant_id')::uuid
);
```

### B. Admin 전용 쓰기 정책
```sql
-- Bulk Import: Admin만 Insert 가능
CREATE POLICY "Admin only bulk import" ON bulk_import_batches
FOR INSERT
WITH CHECK (
  (auth.jwt() ->> 'role') = 'admin'
);

-- 세션 승인/거부: Admin만 COMPLETED/ARCHIVED 상태 변경 가능
CREATE POLICY "Admin only finalize session" ON audit_sessions
FOR UPDATE
USING (
  (auth.jwt() ->> 'role') = 'admin'
  OR status NOT IN ('COMPLETED', 'ARCHIVED')
);
```

### C. User 역할 자기 데이터 제한 정책
```sql
-- User는 자신에게 할당된 세션만 수정 가능
CREATE POLICY "User own session only" ON audit_sessions
FOR UPDATE
USING (
  (auth.jwt() ->> 'role') = 'admin'
  OR assignee_id = auth.uid()
);

-- User는 자신이 생성한 데이터 엔트리만 Insert 가능
CREATE POLICY "User own entries only" ON audit_data_entries
FOR INSERT
WITH CHECK (
  created_by = auth.uid()
);
```

---

## 5. 권한 변경 감사 로그 (REQ-FUNC-024 연계)

모든 권한 변경 행위는 `audit_log` 테이블에 Insert-only 방식으로 기록됩니다.

### A. 기록 대상 이벤트

| 이벤트 | 설명 | 기록 필드 | 중요도 |
|:---|:---|:---|:---:|
| `ROLE_CHANGED` | 사용자 역할 변경 (User→Admin 또는 Admin→User) | `target_user_id`, `previous_role`, `new_role`, `changed_by` | Critical |
| `TENANT_ASSIGNED` | 사용자 테넌트/사이트 배정 변경 | `target_user_id`, `previous_tenant_id`, `new_tenant_id` | High |
| `SESSION_REASSIGNED` | 세션 담당자 재할당 | `session_id`, `previous_assignee`, `new_assignee` | Medium |

### B. 트리거 구현
```sql
-- 역할 변경 시 자동 감사 로그 적재
CREATE OR REPLACE FUNCTION fn_log_role_change()
RETURNS TRIGGER AS $$
BEGIN
  IF OLD.role IS DISTINCT FROM NEW.role THEN
    INSERT INTO audit_log (
      action_type, user_id, target_table, target_id,
      payload, status
    ) VALUES (
      'ROLE_CHANGED', auth.uid(), 'users', NEW.id,
      jsonb_build_object(
        'previous_role', OLD.role,
        'new_role', NEW.role
      ),
      'PASS'
    );
  END IF;
  RETURN NEW;
END;
$$ LANGUAGE plpgsql SECURITY DEFINER;

CREATE TRIGGER trg_role_change_audit
AFTER UPDATE ON users
FOR EACH ROW
EXECUTE FUNCTION fn_log_role_change();
```

---

## 6. Edge Case 처리

### A. 역할 변경 시 기존 세션 무효화 정책

| 시나리오 | 처리 방식 | 구현 위치 |
|:---|:---|:---|
| Admin → User 강등 | 현재 활성 세션(JWT)의 `role` 클레임이 구버전이므로, 다음 토큰 갱신(Refresh) 시점에 새 역할 반영. **즉시 차단이 필요한 경우** Supabase `auth.admin.updateUserById()`로 모든 Refresh Token 강제 폐기 | Server Action |
| User → Admin 승격 | 기존 세션은 User 권한 유지. 로그아웃 후 재로그인 시 Admin 클레임 반영. UX상 '권한이 변경되었습니다. 다시 로그인해주세요' 토스트 알림 | Middleware + UI |
| 동시 로그인 세션 처리 | 역할 변경 시 해당 사용자의 모든 활성 세션에 `TOKEN_REFRESH_REQUIRED` 플래그를 Redis(또는 Supabase Realtime)로 브로드캐스트 | Edge Function |

### B. 비인가 접근 시 응답 규격 (API-001 준수)
```typescript
// 권한 부족 시 표준 에러 응답
{
  "error": {
    "code": "ACCESS_DENIED",
    "message": "이 작업을 수행할 권한이 없습니다.",
    "status": 403,
    "details": {
      "required_role": "admin",
      "current_role": "user",
      "attempted_action": "BULK_IMPORT_UPLOAD"
    }
  }
}
```

---

## 7. SRS 트레이서빌리티

| SRS 요구사항 ID | 요구사항 제목 | 본 문서 커버리지 |
|:---|:---|:---|
| REQ-NF-SEC-AUTHZ-001 | 미정의 권한 요청 100% 거부 | §2 미들웨어 + §4 RLS 정책 |
| REQ-NF-SEC-AUTHZ-002 | Cross-site 접근 0건 | §4.A 테넌트 격리 RLS |
| REQ-FUNC-024 | Insert-only 감사 로그 적재 | §5 감사 로그 트리거 |
| REQ-FUNC-026 | 2단계 권한 제어 (Admin/User) | §1 권한 매트릭스 |
| REQ-NF-SEC-AUTH-001 | MFA 적용률 100% (관리자) | §2 미들웨어에서 MFA 체크 연계 (상세는 `NFR-COMPLIANCE_v1.md` §2 참조) |
