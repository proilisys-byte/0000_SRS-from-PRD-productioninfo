# NFR-COMPLIANCE_v1.md

## 1. PIPA 동의 폼 구현 명세 (T4-002)

### A. 동의 폼 화면 구성
- **수집 항목**:
  - 음성 데이터 (필수): STT 변환 및 Smart Audit 매핑용
  - 위치 정보 (선택): 작업 수행 위치 검증 및 동선 최적화용
  - 작업 기록 (필수): 시스템 Audit Log 기록 및 이력 관리용
- **UI 컴포넌트 (shadcn/ui 기반)**:
  - `Dialog` 또는 `Sheet`: 화면 진입 시 강제 팝업 노출
  - `Checkbox`: 각 항목별 동의/거부 체크
  - `Button`: '동의 및 계속하기' (필수 항목 미동의 시 비활성화)
- **5개 국어 지원 구조**:
  - **라이브러리**: `next-intl` (선택 이유: Next.js App Router의 Server Component와 높은 호환성 및 라우팅 통합 지원, Vercel 환경에서 Edge Runtime 동작에 최적화)
  - **지원 언어**: 한국어, 영어, 베트남어, 인도네시아어, 태국어

### B. 동의 기록 DB 저장 명세
동의 이력은 법적 증빙을 위해 변경 불가한 형태로 관리되어야 합니다.
- **테이블 구조 (`user_consents`)**:
  - `user_id` (UUID): 사용자 식별자
  - `consent_type` (String): 'PIPA_VOICE', 'PIPA_LOCATION' 등
  - `consent_version` (String): 동의서 약관 버전 (예: 'v1.0')
  - `agreed_at` (Timestamptz): 동의 일시
  - `ip_address` (String): 접근 IP 주소
  - `device_fingerprint` (String): 기기 식별 정보
- **Insert-only 적용**:
  - **적용 여부**: 예
  - **이유**: 사용자의 동의 이력은 위/변조가 불가능해야 하는 컴플라이언스 핵심 데이터입니다. PostgreSQL의 Rule 또는 Trigger를 사용하여 `UPDATE` 및 `DELETE` 쿼리를 원천 차단하여 감사의 투명성을 확보합니다.

### C. Edge 디바이스 최초 진입 시 동의 강제화 로직
- **미동의 사용자 차단 미들웨어 (Next.js Middleware 패턴)**:
```typescript
import { NextResponse } from 'next/server';
import type { NextRequest } from 'next/server';

export async function middleware(req: NextRequest) {
  // 토큰 등에서 사용자의 PIPA 동의 여부(PIPA_VOICE) 확인 (JWT Custom Claim 권장)
  const hasConsented = req.cookies.get('pipa_consent_voice')?.value === 'true';
  
  if (req.nextUrl.pathname.startsWith('/dashboard') || req.nextUrl.pathname.startsWith('/api/stt')) {
    if (!hasConsented) {
      // 미동의 시 강제 동의 페이지 또는 팝업 노출용 라우트로 리다이렉트
      return NextResponse.redirect(new URL('/auth/consent', req.url));
    }
  }
  return NextResponse.next();
}
```
- **동의 철회 시 처리 방안**:
  - 설정(Settings) 메뉴에 '동의 철회' 버튼 제공.
  - 철회 시 기존 토큰/쿠키의 Claim 무효화 및 `user_consents` 테이블에 철회(Revoke) 기록 Insert (기존 레코드 수정 불가, 새로운 행으로 철회 상태 추가).
  - 이후 STT 기능 및 관련 대시보드 접근 즉각 차단.

### D. 노조 합의 지연 시 대안 시나리오 (미결 항목 해소)
- **시나리오 A: 합의 완료 후 정상 배포 (기본 경로)**
  - 기술 준비: 모든 STT 기능, 마이크 접속 권한, PIPA 동의 로직 활성화. `NEXT_PUBLIC_ENABLE_VOICE_FEATURE=true` 환경변수 설정.
- **시나리오 B: Sprint 4 종료 후 2주 추가 연장 (경영진 승인 조건)**
  - 기술 준비: 브랜치 병합 보류 또는 Feature Toggle을 통해 기능 숨김 상태 유지. DB 스키마는 배포하되 API 엔드포인트 접근 차단.
- **시나리오 C: 음성 수집 기능만 비활성화 후 부분 배포**
  - 기술 준비: `NEXT_PUBLIC_ENABLE_VOICE_FEATURE=false`로 설정 시 UI에서 마이크 버튼이 일반 텍스트 입력창(수동 타이핑)으로 Fallback 렌더링되도록 구현. 백엔드의 `/api/stt` 엔드포인트는 `503 Service Unavailable` 반환 처리.

---

## 2. 관리자 MFA 구현 명세 (T4-003)

### A. Supabase Auth TOTP 활성화 방법
- Supabase Dashboard > Authentication > Policies & Configuration 에서 "Enable Multi-Factor Authentication" 활성화.
- `supabase-js`의 `mfa.enroll()`, `mfa.challenge()`, `mfa.verify()` API를 활용하여 클라이언트 단에 설정 플로우 구축.

### B. Admin 최초 로그인 시 MFA 등록 강제 플로우
1. 사용자가 이메일/비밀번호로 1차 로그인.
2. 서버는 RBAC 권한을 검사하여 사용자 역할이 `Admin`인지 확인.
3. `Admin`인 경우, Supabase에서 `mfa.getAuthenticatorAssuranceLevel()`을 호출하여 `currentLevel` 확인.
4. MFA가 설정되지 않은 경우 (`currentLevel === 'aal1'`), 관리자 대시보드 접근을 막고 MFA 설정 페이지(`/auth/mfa-setup`)로 강제 리다이렉트.
5. QR 코드 스캔 및 OTP 입력 후 검증이 완료되어야 JWT 토큰 갱신.

### C. MFA 미등록 Admin의 대시보드 접근 차단 로직
```typescript
// middleware.ts 또는 전역 Layout의 보호 로직
import { createServerClient } from '@supabase/ssr';

export async function checkMfaRequirement(req: NextRequest, res: NextResponse) {
  const supabase = createServerClient(/* ... */);
  const { data: { user } } = await supabase.auth.getUser();
  const { data: { aal } } = await supabase.auth.mfa.getAuthenticatorAssuranceLevel();

  // 사용자 역할이 Admin인데 AAL2(MFA 완료 상태)가 아니면 차단
  if (user?.app_metadata?.role === 'admin' && aal?.currentLevel !== 'aal2') {
    return NextResponse.redirect(new URL('/auth/mfa-setup', req.url));
  }
}
```

---

## 3. 보안 요구사항 체크리스트

### A. AES-256 암호화 적용 대상 컬럼 목록
데이터베이스(Supabase) 내 저장 시 애플리케이션 레벨(또는 pgcrypto)에서 암호화가 필요한 민감 정보:
- `users.personal_phone_number` (개인 연락처)
- `audit_data_entries.raw_voice_hash` 또는 민감 음성 데이터의 스토리지 경로명
- 외부 API 연동 토큰 (예: ERP 연동용 Secret Keys)

### B. TLS 1.3 Vercel 기본 적용 확인 방법
- Vercel은 Edge Network를 통해 기본적으로 TLS 1.3을 지원함.
- 확인 방법: 배포된 도메인에 대해 SSL Labs (Qualys) 테스트를 구동하거나, 브라우저 개발자 도구의 Security 탭에서 Connection 설정이 `TLS 1.3, X25519, AES_256_GCM`으로 표시되는지 정기 확인.

### C. 모의 해킹 체크리스트 (OWASP Top 10 대응)
| 위협 카테고리 | 기술적 대응 방안 | 확인 상태 |
| :--- | :--- | :---: |
| **A01: Broken Access Control** | 미들웨어 기반 RBAC(Admin/User) 철저 분리 및 강제 검증, RLS 정책 적용 | [ ] |
| **A02: Cryptographic Failures** | TLS 1.3 적용, 비밀번호 Bcrypt/Argon2 (Supabase 기본), 민감 데이터 AES-256 암호화 | [ ] |
| **A03: Injection** | 클라이언트 직접 쿼리 금지. Prisma ORM 및 Server Actions로 파라미터 바인딩 강제 | [ ] |
| **A04: Insecure Design** | Zero-Trust 접근법 채택. Insert-only 룰을 통한 데이터 위변조 아키텍처 원천 방지 | [ ] |
| **A05: Security Misconfiguration** | 불필요한 HTTP 헤더 제거, Vercel 환경변수 분리(DEV/PRD), Supabase 디폴트 키 비노출 | [ ] |
| **A07: Identification and Auth Failures** | 관리자 계정 대상 MFA 강제, 세션 타임아웃 및 동시 로그인 제어 로직 적용 | [ ] |
