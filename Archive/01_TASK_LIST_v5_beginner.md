# PRO ILI SMART — 개발 태스크 목록 v5 (초보 설계자용, 완벽 가이드)

| 항목 | 내용 |
|:---|:---|
| **기준 문서** | TASK_LIST_v4_beginner.md |
| **작성일** | 2026-04-23 |
| **개정 이력** | v4 → **v5(완벽 가이드: 완료기준, 제약조건, 응답형식, 권한표 추가)** |
| **목적** | 초보 개발자가 아무런 추가 질문 없이 곧바로 개발과 테스트를 진행할 수 있도록 모든 명세를 100% 구체화 |

---

## 1. 역할별 API / 화면 접근 권한 표 (RBAC)

| API / 화면 | Admin | Manager | User | Guest |
|:---|:---:|:---:|:---:|:---:|
| `POST /api/v1/auth/login` (로그인) | ✅ | ✅ | ✅ | ✅ |
| `GET /api/v1/templates` (템플릿 목록) | ✅ | ✅ | ✅ | ❌ |
| `POST /api/v1/audit/sessions` (감사 세션) | ✅ | ✅ | ❌ | ❌ |
| `POST /api/v1/audit/reports` (감사 보고서) | ✅ | ✅ | ❌ | ❌ |
| `POST /api/v1/nc/cases` (NC 등록) | ✅ | ✅ | ❌ | ❌ |
| `PATCH /api/v1/nc/actions/:id` (시정 조치) | ✅ | ✅ | ✅ | ❌ |
| `POST /api/v1/edge/sync` (Edge 연동) | ✅ | ✅ | ❌ | ❌ |
| `POST /api/v1/lean/diagnose` (Lean 진단) | ✅ | ✅ | ❌ | ❌ |
| `GET /api/v1/logs` (감사 로그 조회) | ✅ | ❌ | ❌ | ❌ |
| 대시보드 화면 (`/dashboard`) | ✅ | ✅ | ✅ | ❌ |
| NC 관리 화면 (`/nc`) | ✅ | ✅ | ✅ (조회/수정) | ❌ |
| 환경 설정 화면 (`/settings`) | ✅ | ❌ | ❌ | ❌ |

---

## 2. DB 테이블 제약조건 정의서

| 테이블 | 컬럼 | 타입 | Nullable | Unique | FK | Default | DeleteRule |
|:---|:---|:---|:---:|:---:|:---|:---|:---|
| **SITE** | id | uuid | NO | YES | - | gen_random_uuid() | - |
| | name | text | NO | NO | - | - | - |
| | created_at | timestamp | NO | NO | - | now() | - |
| **USER** | id | uuid | NO | YES | - | gen_random_uuid() | - |
| | auth_id | uuid | NO | YES | auth.users.id | - | CASCADE |
| | site_id | uuid | NO | NO | SITE.id | - | CASCADE |
| | role | text | NO | NO | - | 'USER' | - |
| **RAW_DATA** | id | uuid | NO | YES | - | gen_random_uuid() | - |
| | site_id | uuid | NO | NO | SITE.id | - | CASCADE |
| | payload | jsonb | NO | NO | - | '{}' | - |
| **TEMPLATE** | id | uuid | NO | YES | - | gen_random_uuid() | - |
| | version | int | NO | NO | - | 1 | - |
| | schema | jsonb | NO | NO | - | '{}' | - |
| **AUDIT_SESSION** | id | uuid | NO | YES | - | gen_random_uuid() | - |
| | site_id | uuid | NO | NO | SITE.id | - | CASCADE |
| | template_id| uuid | NO | NO | TEMPLATE.id | - | RESTRICT |
| | due_date | timestamp | YES| NO | - | - | - |
| **AUDIT_REPORT** | id | uuid | NO | YES | - | gen_random_uuid() | - |
| | session_id | uuid | NO | NO | AUDIT_SESSION.id | - | CASCADE |
| | content | jsonb | NO | NO | - | '{}' | - |
| **NC_CASE** | id | uuid | NO | YES | - | gen_random_uuid() | - |
| | site_id | uuid | NO | NO | SITE.id | - | CASCADE |
| | due_date | timestamp | NO | NO | - | - | - |
| | is_completed | boolean | NO | NO | - | false | - |
| **CORRECTIVE** | id | uuid | NO | YES | - | gen_random_uuid() | - |
| **_ACTION** | nc_id | uuid | NO | NO | NC_CASE.id | - | CASCADE |
| | is_completed | boolean | NO | NO | - | false | - |
| **LEAN_DIAG** | id | uuid | NO | YES | - | gen_random_uuid() | - |
| | site_id | uuid | NO | NO | SITE.id | - | CASCADE |
| **AUDIT_LOG** | id | uuid | NO | YES | - | gen_random_uuid() | - |
| | user_id | uuid | NO | NO | USER.id | - | SET NULL |
| **SUBMISSION** | id | uuid | NO | YES | - | gen_random_uuid() | - |
| **_LOG** | payload | jsonb | NO | NO | - | '{}' | - |

---

## 3. Task 선행/후행 흐름도 (Dependency Graph)

```text
[Phase 0: 환경 설정]
ENV-001(Next.js) → ENV-002(Supabase) → ENV-003(Prisma) → ENV-004(Postman)

[Phase 1: DB & API 기초]
ENV-004 → DB-001~011(스키마 생성)
DB-011 → ARCH-001(미들웨어) → ARCH-002(에러코드)
ARCH-002 → API-007s(공통응답) → API-R01~R05(DTO)
API-R05 → MOCK-001~005(시드 데이터)
MOCK-005 → CRUD-001~013(API 라우트)
CRUD-013 → UI-001a~005s(기본 UI)

[Phase 2: 보안 & 로깅]
UI-005s → UI-001b(실제 로그인) → AUTH-001s~004s(권한/세션)
AUTH-004s → VAL-001~004(검증)
VAL-004 → LOG-001~003(감사 로그)

[Phase 3: 비즈니스 로직]
LOG-003 → SA-001s~004s(규칙 매핑)
SA-004s → NC-001s~003s(시정 조치)
NC-003s → EDGE-001s~003s(엣지 연동)
EDGE-003s → LEAN-000s~003s(린 진단)
LEAN-003s → BULK-001s~003s(CSV 업로드)

[Phase 4: AI 확장]
BULK-003s → AI-MAP-001~003
AI-MAP-003 → AI-STR-001~003
AI-STR-003 → AI-STT-001, AI-VIS-001, AI-FALL-001
```

---

## 4. 공통 API 응답 규격

모든 API 구현 시 아래 JSON 규격을 100% 준수해야 합니다.

**✅ 성공 응답 (200 OK / 201 Created)**
```json
{
  "status": "success",
  "data": {
    "id": "123e4567-e89b-12d3-a456-426614174000",
    "message": "Operation completed"
  },
  "request_id": "req-uuid-9876"
}
```

**❌ 실패 응답 (400 / 401 / 403 / 404 / 500)**
```json
{
  "status": "error",
  "message": "NC Case not found",
  "error_code": "NC_NOT_FOUND",
  "request_id": "req-uuid-9876"
}
```
*(※ `error_code`는 반드시 `ARCH-002`에서 정의한 Enum 값 사용)*

---

## 5. DB 마이그레이션 안전 운영 가이드

### 🚦 Prisma 명령어 신호등 (위험도 가이드)
초보자가 혼동하기 쉬운 Prisma 명령어의 용도와 위험도를 직관적으로 구분합니다.

| 등급 | 명령어 | 실행 환경 | 용도 및 설명 | 실패/오용 시 리스크 |
|:---:|:---|:---:|:---|:---|
| 🟢 **안전** | `npx prisma migrate deploy` | **Prod** / Local | 마이그레이션 이력을 DB에 순차적으로 반영합니다. | 없음 (안전함, 데이터 유실 없음) |
| 🟡 **주의** | `npx prisma migrate dev` | **Local** 전용 | 스키마 변경 사항을 로컬 DB에 반영하고 이력을 생성합니다. | 기존 데이터 초기화(DROP) 경고 발생 가능 (테스트 데이터 유실) |
| 🔴 **위험** | `npx prisma db push` | **Local** 전용 | 마이그레이션 이력 없이 스키마를 강제로 덮어씁니다. | **운영(Prod) 환경 절대 금지!** 마이그레이션 이력 파괴 및 치명적 데이터 유실 |

### 🚑 DB 긴급 복구 매뉴얼 (Fallback)
운영 DB 마이그레이션 사고 발생 시 당황하지 않고 5분 내에 복구하기 위한 최후의 보루입니다.

1. **사전 준비 (백업 체계)**
   - Supabase의 PITR(Point-in-Time Recovery) 설정 확인
   - 주기적 백업 태스크 명시 (배포 및 마이그레이션 실행 직전 수동 스냅샷 생성)

2. **복구 시나리오 (단계별 대응)**
   - **Case A (이력 정정):** 마이그레이션 실패 시 `npx prisma migrate resolve`를 사용하여 실패 이력을 정정하거나 처리 상태 롤백.
   - **Case B (스냅샷 복구):** 데이터 유실 등 치명적 오류 시 Supabase 스냅샷으로 특정 시점(T-minus 5 min)으로 되돌리는 절차 실행.

3. **커뮤니케이션 (보고 및 전파)**
   - 사고 발생 즉시 사내 채널에 '서비스 일시 중단' 공지
   - 복구 진행 상황을 투명하게 공유하는 보고 절차 수행

---

## 6. 초보 개발자를 위한 5대 핵심 구현 가이드 및 SOP

초보 개발자가 개발 단계에서 런타임 에러를 겪지 않고, 운영 환경에서 즉각 대응할 수 있도록 반드시 지켜야 할 핵심 가이드입니다.

### 6.1. Supabase Auth ↔ Prisma USER 동기화 규칙
#### 1. 목적
Supabase의 자체 인증 테이블(`auth.users`)과 애플리케이션의 사용자 테이블(`public.User`) 간의 1:1 관계를 완벽하게 일치시키고 동기화한다.
#### 2. 적용 범위
신규 회원가입 로직, 소셜 로그인 최초 접근 시 생성 로직
#### 3. 구현 규칙
*   **Trigger 방식 (권장):** Supabase `auth.users`에 새 레코드가 삽입될 때, PostgreSQL Database Trigger를 사용하여 `public.User` 테이블에 기본 권한(`role: 'USER'`)을 가진 레코드를 자동 생성한다.
    *   *이유:* API 서버가 죽어 있어도 DB 단에서 무결성이 100% 보장됨.
*   **연결 키:** Prisma `User` 모델의 `auth_id` 컬럼은 `@unique` 속성을 가져야 하며, Supabase `auth.users.id` (UUID)를 저장한다.
#### 4. 금지 사항
*   Next.js API 라우트 내에서 `supabase.auth.signUp()` 호출 직후 `prisma.user.create()`를 순차적으로 호출하는 방식 금지 (네트워크 단절 시 고아 데이터 발생).
#### 5. 예외 처리
*   동기화 실패(트리거 오류 등)로 `auth.users`에는 있으나 `User` 테이블에는 없는 계정이 로그인 시도 시, API에서 HTTP 403과 함께 "계정 동기화 오류입니다. 관리자에게 문의하세요" 반환.
#### 6. 완료 기준
*   Supabase Dashboard에서 회원가입 테스트 시, Prisma Studio의 `User` 테이블에 동일한 `auth_id`를 가진 로우가 1초 내에 자동 생성되어야 함.
#### 7. 테스트 기준
*   회원가입 후 발급받은 Access Token으로 즉시 `/api/users/me` 호출 시 본인 정보가 정상 조회되는지 확인.
#### 8. 초보자가 자주 틀리는 포인트
*   Prisma `User` 테이블의 Primary Key(`id`)와 Supabase의 `auth.users.id`를 같은 것으로 착각함. (Prisma의 `id`는 auto-increment 정수 또는 별도 UUID로 두고, `auth_id`라는 별도 String/UUID 컬럼으로 매핑해야 함).

### 6.2. 인증/인가 상세 구현 규칙
#### 1. 목적
모든 보호된 라우트(API 및 UI)에 대해 위변조 불가능한 인증 상태를 확인하고, 정확한 Role(역할) 기반 인가를 수행한다.
#### 2. 적용 범위
`src/middleware.ts`, 모든 `src/app/api/...` 라우트 핸들러
#### 3. 구현 규칙
*   **토큰 저장 위치:** `sb-access-token`과 `sb-refresh-token`은 반드시 `HttpOnly`, `Secure` 옵션이 켜진 쿠키(Cookie)에 저장한다. (Local Storage 절대 금지).
*   **Middleware 처리:** Next.js `middleware.ts`에서는 `@supabase/ssr` 패키지의 `createServerClient`를 사용하여 세션을 검증하고, 만료된 경우 쿠키를 자동 갱신(Refresh)한다.
*   **Role 매핑:** 토큰의 User ID로 Prisma `User` 테이블을 조회하여 `role` 값을 가져오고, 미들웨어에서 `req.headers.set('x-user-role', user.role)` 형태로 헤더에 주입하여 하위 API 라우트로 넘긴다.
#### 4. 금지 사항
*   JWT 페이로드에 있는 `role` 값을 100% 신뢰하여 인가에 사용하는 것 금지 (토큰 발급 후 DB에서 권한이 박탈되었을 수 있음. 반드시 DB 조회 혹은 캐시 검증 병행).
#### 5. 예외 처리
*   Access Token 만료 및 Refresh Token까지 만료/유효하지 않은 경우: HTTP 401 반환 및 쿠키 삭제 처리.
#### 6. 완료 기준
*   권한이 없는 유저가 관리자 API(`/api/admin/*`) 호출 시 정확히 HTTP 403 Forbidden을 반환해야 함.
#### 7. 테스트 기준
*   쿠키를 임의로 조작하거나 만료된 토큰을 보냈을 때 백엔드에서 401 Unauthorized를 내뱉고 로그인 페이지로 리다이렉트 시키는지 검증.
#### 8. 초보자가 자주 틀리는 포인트
*   Next.js App Router의 Server Component에서 Client SDK를 사용하여 세션을 가져오려고 시도하다가 하이드레이션(Hydration) 에러나 서버 에러를 냄 (서버에서는 반드시 SSR용 SDK 사용).

### 6.3. DTO/Zod 단일 진실원칙 규칙
#### 1. 목적
타입스크립트 타입 선언과 Zod 검증 스키마를 이중으로 작성하여 발생하는 유지보수 헬(Hell)과 런타임 에러를 원천 차단한다.
#### 2. 적용 범위
프론트엔드 API 호출 페이로드, 백엔드 API 라우트의 Request Body 및 Response Body
#### 3. 구현 규칙
*   **SSOT(Single Source of Truth):** 반드시 **Zod 스키마를 먼저 선언**한다.
*   **타입 도출:** TypeScript `interface`나 `type`을 직접 손으로 작성하지 않고, `z.infer<typeof 스키마명>`을 사용하여 Zod로부터 타입을 역산(Extract)하여 사용한다.
    ```typescript
    // ✅ 올바른 예시
    export const CreateUserSchema = z.object({ email: z.string().email(), age: z.number().min(18) });
    export type CreateUserDTO = z.infer<typeof CreateUserSchema>;
    ```
#### 4. 금지 사항
*   `interface CreateUserDTO { ... }`를 별도 파일에 만들고, Zod 스키마를 또 다른 곳에 정의하는 행위 절대 금지.
#### 5. 예외 처리
*   API 라우트에서 `CreateUserSchema.parse(body)` 실패 시 ZodError를 캐치하여, 어떤 필드가 왜 틀렸는지 HTTP 400 Bad Request와 함께 명확한 JSON 구조로 응답.
#### 6. 완료 기준
*   프로젝트 내 모든 API Request DTO가 `z.infer`를 통해 생성된 타입만을 사용하고 있어야 함.
#### 7. 테스트 기준
*   필수 값이 누락된 JSON을 API로 POST 전송했을 때, 서버가 죽지 않고 규격화된 검증 실패 메시지를 반환하는지 테스트.
#### 8. 초보자가 자주 틀리는 포인트
*   Next.js API Route 내에서 `try-catch` 없이 Zod의 `.parse()`를 썼다가 검증 실패 시 서버 크래시(500 에러)를 발생시킴. 반드시 `safeParse()`를 쓰거나 에러 핸들러 미들웨어를 구현해야 함.

### 6.4. Mock 로그인 → 실제 로그인 전환 규칙
#### 1. 목적
로컬 UI 개발 단계에서 실제 DB와 인증 서버 없이 작업할 수 있게 하되, 프로덕션 전환 시 단 한 줄의 코드(환경 변수) 변경만으로 실제 로직을 타게 만든다.
#### 2. 적용 범위
로그인 페이지, 인증 상태 조회 훅(`useAuth`), API 클라이언트 유틸리티
#### 3. 구현 규칙
*   **환경 변수 제어:** `.env.local`에 `NEXT_PUBLIC_AUTH_MODE=mock` (또는 `real`)을 선언한다.
*   **Adapter 패턴 적용:** `AuthService` 인터페이스를 만들고, `MockAuthService`와 `SupabaseAuthService` 두 개의 구현체를 만든다.
*   **DI (의존성 주입):** 런타임에 `NEXT_PUBLIC_AUTH_MODE` 값에 따라 팩토리 함수가 알맞은 구현체를 반환하게 한다.
    *   *Mock 토큰 포맷:* `mock-token-admin`, `mock-token-user`와 같이 하드코딩된 명시적 문자열 사용.
#### 4. 금지 사항
*   React 컴포넌트(UI) 내부 코드에 `if (mode === 'mock')` 분기문을 덕지덕지 넣는 것 절대 금지. (분기는 오직 Service Factory 내에서만 1회 수행).
#### 5. 예외 처리
*   프로덕션 빌드 시점(`NODE_ENV=production`)에 `NEXT_PUBLIC_AUTH_MODE=mock`으로 설정되어 있으면 빌드를 강제로 실패(Fail) 시킴.
#### 6. 완료 기준
*   UI 코드나 API 호출 코드를 전혀 수정하지 않고, `.env.local` 수정 및 서버 재시작만으로 임시 로그인과 실제 로그인이 전환되어야 함.
#### 7. 테스트 기준
*   Mock 모드에서 admin 계정 로그인 후 보이는 UI와, Real 모드에서 실제 admin 계정으로 로그인 후 보이는 UI가 100% 동일한지 확인.
#### 8. 초보자가 자주 틀리는 포인트
*   Mock 모드일 때 발급한 가짜 토큰이 브라우저 쿠키에 남아있어, Real 모드로 전환 후 진짜 서버로 가짜 토큰을 보내 CORS나 500 에러를 겪음. 전환 시 반드시 쿠키 클리어 로직 필요.

### 6.5. 계정 잠금 / 자동 로그아웃 / 복구 운영 SOP
#### 1. 목적
무차별 대입 공격(Brute Force)을 막고 공용 PC 등에서의 세션 탈취를 방지하며, 문제 발생 시 운영자가 즉시 해결할 수 있는 가이드를 제공한다.
#### 2. 적용 범위
로그인 API 핸들러, 미들웨어 세션 체크, 어드민 백오피스 사용자 관리 페이지
#### 3. 구현 규칙
*   **자동 로그아웃:** Supabase 세션 만료 시간(기본 1시간 등)과 별개로, 프론트엔드에 사용자 상호작용(마우스, 키보드) 이벤트 리스너를 달아 30분 이상 Inactive 시 로컬 로그아웃(토큰 삭제) 트리거.
*   **5회 실패 잠금 로직:** Prisma `User` 테이블에 `failed_login_attempts` (Int), `locked_until` (DateTime) 컬럼 추가. 비밀번호 오류 시 카운트 증가. 5회 도달 시 `locked_until`을 현재 시간 + 30분으로 세팅.
*   **운영자 복구 (SOP):**
    1. 고객 "비밀번호 5회 틀려서 잠겼습니다." 문의 접수.
    2. 운영자가 백오피스 어드민 페이지에서 해당 유저 검색.
    3. [잠금 해제] 버튼 클릭 (내부적으로 대상 유저의 `failed_login_attempts = 0`, `locked_until = null` 로 UPDATE 수행).
#### 4. 금지 사항
*   고객의 비밀번호를 운영자가 임의의 값(예: `1234`)으로 변경하여 알려주는 행위 절대 금지. (비밀번호 초기화는 반드시 이메일 초기화 링크를 통해서만 수행).
#### 5. 예외 처리
*   계정이 잠긴 상태에서 올바른 비밀번호를 입력하더라도 로그인 거부 및 "계정이 30분간 잠겼습니다. (남은 시간: XX분)" 명시적 에러 반환.
#### 6. 완료 기준
*   고의로 비밀번호 5회 오입력 시 로그인이 완벽히 차단되며, 어드민 화면에서 클릭 한 번으로 즉시 차단 해제 후 로그인이 가능해져야 함.
#### 7. 테스트 기준
*   잠금 기간(30분)이 지나면 수동 해제 없이도 자동으로 로그인 가능 상태로 복구되는지 시간 조작을 통해 테스트.
#### 8. 초보자가 자주 틀리는 포인트
*   실패 카운트를 쿠키나 로컬스토리지에 저장하여, 시크릿 모드를 열거나 캐시를 지우면 카운트가 초기화되어 무한 시도가 가능해지는 보안 구멍을 만듦 (반드시 DB에 저장할 것).

---

## 7. 구현용 표준 아키텍처 제안

초보자가 위 규칙들을 구현할 때 파일 위치와 흐름을 헷갈리지 않도록 명확한 기준을 제시합니다.

### 7.1. 인증 흐름 순서도 (Standard Flow)

```text
[클라이언트(브라우저)]
  │
  ├── 1. 로그인 요청 (이메일/비번) ───────▶ [Next.js API Route (/api/auth/login)]
  │                                        │
  │                                        ├── 2. Zod 스키마 검증 (형식 체크)
  │                                        │   └── (실패시) 400 Bad Request 리턴
  │                                        │
  │                                        ├── 3. Supabase 로그인 시도 (signInWithPassword)
  │                                        │   ├── (비번 오입력) DB 카운트 증가 -> 401 리턴
  │                                        │   └── (5회 초과) 계정 잠금 -> 403 리턴
  │                                        │
  │                                        ├── 4. Prisma USER 테이블 조회 (auth_id로 매핑)
  │                                        │   └── (데이터 없음) 500 에러 리턴 및 로깅
  │                                        │
  │                                        └── 5. 성공: 쿠키에 Token 저장, 200 OK 리턴
  │
  ├── 6. 보호된 페이지/API 접근 ─────────▶ [Next.js Middleware (middleware.ts)]
  │                                        │
  │                                        ├── 7. Request 헤더/쿠키 검증 (Supabase SSR)
  │                                        │   └── (만료시) Token Refresh 시도 또는 401 리턴
  │                                        │
  │                                        ├── 8. UUID 추출 및 Redis/DB에서 Role(역할) 판별
  │                                        │   └── (권한 없음) 403 Forbidden 리턴
  │                                        │
  │                                        └── 9. request_id, x-user-role 헤더 주입 후 통과
  │
  ▼
[목적지 API / Server Component 실행 및 결과 반환]
```

### 7.2. 권장 폴더 구조 (Next.js App Router 기준)

```text
src/
  app/
    (auth)/               # 로그인, 회원가입 라우트 그룹
    api/                  # 통합 API 엔드포인트
      auth/               # 로그인 로직 라우트
      users/              # 유저 관리 라우트
  middleware.ts           # [핵심] 토큰 검증, Role 판별, request_id 추적
  lib/
    db/
      prisma.ts           # Prisma 클라이언트 싱글톤 인스턴스
    auth/
      supabase-server.ts  # 서버용 Supabase 클라이언트
      supabase-client.ts  # 클라이언트용 Supabase 클라이언트
    utils/
      errorHandler.ts     # ZodError 및 Prisma 예외 공통 포맷 처리
  services/
    authService.ts        # Mock/Real 전환 로직이 포함된 서비스 레이어
  repositories/
    userRepository.ts     # DB(Prisma) 직접 조회/수정 로직 캡슐화
  schemas/
    auth.schema.ts        # [단일 진실 원칙] Zod 스키마 (로그인, 회원가입 등)
    user.schema.ts        # 유저 응답용 Zod 스키마
  types/
    dto.ts                # z.infer를 통해 추출된 타입들 export
  constants/
    roles.ts              # 권한 체계 상수 정의 (ADMIN, USER 등)
    errorCodes.ts         # 표준화된 에러 코드 맵핑표
```

---

## 8. 상세 Task List 및 완료 기준

### Epic 0: 환경 설정 (ENV-SETUP)

**ENV-001: Next.js + TypeScript 초기화**
- **1. 입력:** `npx create-next-app@latest my-app --ts`
- **2. 처리:** 패키지 다운로드 및 보일러플레이트 생성
- **3. 출력:** Next.js 로고 화면 (`localhost:3000`)
- **4. 예외:** 포트 충돌 시 3001 포트 사용 확인
- **5. 테스트:** 브라우저 접속 및 `curl localhost:3000`

**ENV-002: Supabase 프로젝트 연동**
- **1. 입력:** Supabase Dashboard에서 발급받은 URL 및 ANON_KEY
- **2. 처리:** `.env.local` 파일에 환경 변수 맵핑
- **3. 출력:** 애플리케이션 내 Supabase Client 초기화 완료
- **4. 예외:** 키 값 오타 시 클라이언트 초기화 에러 발생
- **5. 테스트:** `supabase.auth.getSession()` 호출하여 에러 없이 빈 세션 반환 확인

**ENV-003: Prisma 초기화**
- **1. 입력:** `npx prisma init` 및 `DATABASE_URL` 설정
- **2. 처리:** `schema.prisma` 파일 생성 및 커넥션 풀 연결
- **3. 출력:** `npx prisma db push` 실행 성공
- **4. 예외:** DB URL 인증 실패 시 연동 에러
- **5. 테스트:** `npx prisma studio` 실행 후 스튜디오 진입 확인

**ENV-004: Postman 환경 구성**
- **1. 입력:** API Base URL (`http://localhost:3000/api/v1`)
- **2. 처리:** Postman Collection 및 Environment 생성
- **3. 출력:** `GET /api/v1/health` 엔드포인트 세팅 완료
- **4. 예외:** 로컬 서버 미구동 시 Connection Refused 에러
- **5. 테스트:** Health 엔드포인트 호출 시 HTTP 200 응답 확인

**ENV-005: Safe-Guard 스크립트 구현 (안전 배포 설정)**
- **1. 입력:** `package.json` 스크립트 환경 설정 (또는 셸 스크립트)
- **2. 처리:** `NODE_ENV=production`일 경우 위험한 명령어(`db push` 등) 실행을 중단(`exit 1`)시키는 로직 포함
- **3. 출력:** 배포 파이프라인(GitHub Actions 등)에서 오직 `migrate deploy` 명령만 허용하도록 강제
- **4. 예외:** 스크립트 우회 시 배포 단계에서 무조건 블록 처리
- **5. 테스트:** 로컬에서 `NODE_ENV=production npx prisma db push` 실행 시 강제 종료(exit 1)되는지 검증

---

### Epic 0.5: SQL 검증 체크리스트 (Sprint 0)

초보 개발자가 운영 DB 배포 전 반드시 수행해야 하는 '3단계 SQL 검증 체크리스트'입니다. 이 절차를 통과해야만 배포를 승인합니다.

**CHK-001: 3단계 SQL 검증 수행**
- **1. 입력:** 자동 생성된 `migration.sql` 파일
- **2. 처리:** 
  - **1단계 (DROP 구문 검색):** 파일 내에 `DROP TABLE`이나 `DROP COLUMN`이 있는지 확인 (데이터 삭제 위험)
  - **2단계 (컬럼 속성 확인):** `NOT NULL` 제약 조건이 데이터가 이미 있는 테이블에 추가되는지 확인 (배포 실패 위험)
  - **3단계 (데이터 타입 변경):** 기존 컬럼의 타입 변경(예: String -> Int) 시 데이터 유실 여부 판단
- **3. 출력:** 검증 통과(Pass) 시 배포 파이프라인 승인
- **4. 예외:** 위반 사항 발견 시 마이그레이션 롤백 및 보완
- **5. 테스트:** 의도적으로 `DROP` 구문이 포함된 SQL로 검증 시 반려(Fail)되는지 확인

---

### Epic 1: DB 스키마 (Sprint 0)

**DB-001 ~ DB-011: 11개 핵심 테이블 생성**
- **1. 입력:** `schema.prisma` 파일 내 11개 모델 정의서
- **2. 처리:** `npx prisma db push` 마이그레이션 실행
- **3. 출력:** 실제 PostgreSQL 테이블 11개 생성 완료
- **4. 예외:** FK 타입 불일치 (예: String vs UUID) 시 생성 실패
- **5. 테스트:** Prisma Studio에서 11개 테이블 목록 노출 확인

---

### Epic 1.5: 아키텍처 기반 (Sprint 1a)

**ARCH-001: 패스스루 미들웨어 슬롯**
- **1. 입력:** HTTP Request 객체
- **2. 처리:** 빈 미들웨어 체인(`next()`)을 통과시키는 베이스 라우터 래퍼 실행
- **3. 출력:** 로직 변형 없이 원본 Response 반환
- **4. 예외:** 미들웨어 내부 예외 발생 시 500 에러 캐치
- **5. 테스트:** 미들웨어를 거친 API와 안 거친 API의 응답 완전 일치 확인

**ARCH-002: ErrorCode Enum 정의**
- **1. 입력:** 도메인별 예상 에러 목록 문자열 (예: `NC_NOT_FOUND`)
- **2. 처리:** `src/constants/error-codes.ts`에 TypeScript Enum 선언
- **3. 출력:** 전역에서 사용 가능한 `ErrorCode` 객체
- **4. 예외:** 중복된 문자열 할당 시 컴파일러 경고 발생
- **5. 테스트:** 임의의 API에서 import하여 JSON 응답 구조에 정상 적용되는지 확인

---

### Epic 2: API DTO + 에러 코드 (Sprint 1a)

**API-007s: 공통 에러/성공 응답 래퍼**
- **1. 입력:** 비즈니스 데이터 객체 또는 Error 객체
- **2. 처리:** `status`, `request_id` 필드를 주입하여 표준 규격 래핑
- **3. 출력:** 성공/실패 표준화된 JSON Response 반환
- **4. 예외:** 직렬화 불가능한 객체 전달 시 빈 JSON 반환 방어
- **5. 테스트:** 성공/실패 데이터를 넣고 반환된 JSON 규격 일치 여부 검증

**API-R01 ~ R05: 도메인별 DTO 인터페이스**
- **1. 입력:** Zod 스키마 또는 TypeScript Interface 정의
- **2. 처리:** Request/Response의 데이터 구조 및 타입 강제
- **3. 출력:** TS 타입 (예: `CreateNCRequest`, `AuditReportResponse`)
- **4. 예외:** 타입 에러 시 빌드 단계에서 실패
- **5. 테스트:** DTO 변수에 잘못된 타입 할당 시 컴파일 에러 발생 확인

---

### Epic 3: Mock 데이터 (Sprint 1b)

**MOCK-001 ~ MOCK-005: Seed 데이터 적재**
- **1. 입력:** 정적 JSON 배열 데이터 (SITE, USER 등)
- **2. 처리:** `prisma.site.createMany()` 스크립트 기반 데이터 삽입
- **3. 출력:** 실제 DB 내부에 기초 데이터 20건 적재 완료
- **4. 예외:** 부모 테이블(FK)보다 자식 테이블 먼저 생성 시 제약 에러
- **5. 테스트:** `npx prisma db seed` 실행 후 Studio에서 건수 일치 확인

---

### Epic 4: API 라우트 (Sprint 1b)

*(CRUD-001 ~ CRUD-013 공통 완료 기준)*
- **1. 입력:** HTTP Body (JSON) 또는 Query Parameter
- **2. 처리:** Prisma Client 단일 호출 (`findMany`, `create`, `update`)
- **3. 출력:** API-007s 규격을 감싼 JSON (데이터 포함) 반환
- **4. 예외:** DB 쿼리 실패 시 `INTERNAL_ERROR` 혹은 `NOT_FOUND`
- **5. 테스트:** Postman으로 13개 엔드포인트 호출 후 HTTP 200 반환 확인

---

### Epic 5: UI 최소 기능 (Sprint 1c)

**UI-001a: Mock 로그인 화면**
- **1. 입력:** 이메일 및 비밀번호 (Form)
- **2. 처리:** 하드코딩된 인증 성공 로직 후 JWT 목업 토큰 저장
- **3. 출력:** `/dashboard` 페이지로 클라이언트 사이드 라우팅
- **4. 예외:** 빈 이메일 제출 시 유효성 검사 에러 라벨 노출
- **5. 테스트:** 브라우저에서 버튼 클릭만으로 대시보드 진입되는지 확인

**UI-002s ~ UI-005s: 기본 목록/상세 화면**
- **1. 입력:** 마운트 시 API GET 호출 응답 데이터
- **2. 처리:** JSON 데이터를 기반으로 React Table 컴포넌트에 맵핑
- **3. 출력:** 데이터가 채워진 UI 목록 및 상세 뷰 렌더링
- **4. 예외:** 데이터 패칭 실패 시 Error 바운더리 또는 Fallback UI 노출
- **5. 테스트:** 화면 진입 시 DB의 Mock 데이터가 화면에 일치하게 뜨는지 눈으로 확인

---

### Epic 6: 인증·인가 (Sprint 2)

**UI-001b & AUTH-001s: Supabase Auth 실제 연동**
- **1. 입력:** 사용자 이메일, 패스워드
- **2. 처리:** `supabase.auth.signInWithPassword()` 호출
- **3. 출력:** 진짜 인증된 Session 객체 쿠키/스토리지 저장
- **4. 예외:** 인증 실패 시 Supabase AuthApiError 400 에러 처리
- **5. 테스트:** DB의 `auth.users`와 대조하여 로그인 성공 여부 검증

**AUTH-002s: 역할 구분 (RBAC)**
- **1. 입력:** API 헤더 내 JWT 토큰
- **2. 처리:** 토큰 디코드 후 `USER` 테이블의 `role` 검사
- **3. 출력:** 일치 시 `next()`, 불일치 시 HTTP 403 반환
- **4. 예외:** 토큰 만료/변조 시 401 `UNAUTHORIZED` 반환
- **5. 테스트:** User 권한으로 Admin API 호출 시 403 에러 리턴 확인

**AUTH-003s: 로그인 실패 잠금**
- **1. 입력:** 반복된 로그인 실패 이벤트
- **2. 처리:** 실패 횟수 5회 이상 카운트 시 추가 요청 차단
- **3. 출력:** "계정이 잠겼습니다" 에러 메시지 노출
- **4. 예외:** 캐시/상태 초기화 버그로 잠금 안 풀림 현상
- **5. 테스트:** 의도적으로 5번 틀린 뒤 6번째에 정상 비밀번호 입력해도 막히는지 확인

**AUTH-004s: 자동 로그아웃**
- **1. 입력:** 사용자 액션 유무 타이머 (30분)
- **2. 처리:** 30분 초과 시 로컬 세션 삭제 로직 실행
- **3. 출력:** 로그인 화면으로 강제 리다이렉트
- **4. 예외:** 토큰 갱신 로직 누락 시 사용 도중 튕김 현상
- **5. 테스트:** 유휴 시간 임의 단축(예: 10초) 후 자동 로그아웃 되는지 확인

---

### Epic 7: 기본 Validation (Sprint 2)

**VAL-001 ~ VAL-004: Zod 스키마 기반 검증**
- **1. 입력:** HTTP Request Body 전체
- **2. 처리:** 미들웨어 층에서 `zod.parse()` 실행
- **3. 출력:** 규격 일치 시 컨트롤러로 패스, 불일치 시 400 반환
- **4. 예외:** JSON 문법 자체가 깨진 경우 파서 에러
- **5. 테스트:** 빈 바디 전송 시 `VALIDATION_FAILED` 에러와 빠진 필드 목록 응답 확인

---

### Epic 8: 단순 로그 시스템 (Sprint 2)

**LOG-001 ~ LOG-003: 시스템 감사 로그**
- **1. 입력:** C/U/D API 호출 시점의 리소스 및 액션 문자열
- **2. 처리:** `AUDIT_LOG` 테이블에 비동기 Insert
- **3. 출력:** DB에 `{action: 'CREATE', entity: 'NC_CASE'}` 기록 생성
- **4. 예외:** 로그 저장 실패 시 메인 프로세스 롤백 없이 계속 진행 (Best Effort)
- **5. 테스트:** 데이터 조작 후 `GET /api/v1/logs`에서 직전 이력 확인

---

### Epic 9: Smart Audit 규칙 기반 (Sprint 3a)

**SA-001s ~ SA-004s: 하드코딩 매핑 및 리포트 생성**
- **1. 입력:** `RAW_DATA` 페이로드 JSON
- **2. 처리:** `switch-case` 하드코딩 기반으로 ISO Section 도출
- **3. 출력:** 분류 완료된 `AUDIT_REPORT` 레코드
- **4. 예외:** 룰셋에 없는 미분류 데이터는 'unknown' 탭으로 할당
- **5. 테스트:** 3종의 샘플 페이로드를 넣고 Report JSON 배열이 알맞게 떨어지는지 확인

---

### Epic 10: NC 시정 조치 (Sprint 3a)

**NC-001s ~ NC-003s: 기한 설정 및 시정 흐름**
- **1. 입력:** NC 생성 시점의 `severity` (HIGH/LOW)
- **2. 처리:** HIGH면 오늘+7일, LOW면 오늘+30일로 `due_date` 할당
- **3. 출력:** `due_date` 필드가 채워진 레코드 반환
- **4. 예외:** 잘못된 날짜 포맷 계산으로 DB 에러 발생
- **5. 테스트:** NC 생성 직후 DB의 날짜값이 정확히 7일/30일 뒤인지 검증

---

### Epic 11: Edge 데이터 수집 (Sprint 3b)

**EDGE-001s ~ EDGE-003s: Raw 데이터 적재**
- **1. 입력:** IoT 단말에서 쏘는 Raw JSON 문자열
- **2. 처리:** 큐잉 없이 즉시 DB `payload` 컬럼에 Insert
- **3. 출력:** 성공 HTTP 201 응답 및 저장된 레코드 ID
- **4. 예외:** 용량 한도(5MB) 초과 시 HTTP 413 반환
- **5. 테스트:** 1MB 크기 JSON 전송 시 1초 내에 201 응답 수신 확인

---

### Epic 12: Lean 진단 (Sprint 3b)

**LEAN-000s ~ LEAN-003s: 재무적 낭비 금액 도출**
- **1. 입력:** 현장 조사 데이터 (수량, 단가, 리드타임 등)
- **2. 처리:** 4대 낭비 계산 수식(수량 × 단가)을 통한 금액 연산
- **3. 출력:** 종합 낭비 금액이 기재된 `LEAN_DIAGNOSIS` 결과 반환
- **4. 예외:** 수량에 문자열 포함 시 Validation 에러 처리
- **5. 테스트:** (수량 10 × 단가 500) 입력 시 결과 금액 "5000" 계산 확인

---

### Epic 13: 벌크 업로드 (Sprint 3b)

**BULK-001s ~ BULK-003s: CSV 기반 다건 등록**
- **1. 입력:** CSV 파일 데이터 (multipart/form-data)
- **2. 처리:** 서버 파싱 후 데이터 Validation 검증 거쳐 DB `createMany`
- **3. 출력:** 완료 건수 및 실패 열의 Error 리포트
- **4. 예외:** 잘못된 헤더 컬럼명 제출 시 전체 파싱 거부 (400)
- **5. 테스트:** 10건 중 2건이 오류인 파일 업로드 시 8건 성공, 2건 에러 번호 응답 확인

---

### Epic 14~16: AI Optional Layer (Sprint 4)

**AI 연동 (MAP / STR / STT / VIS / FALL)**
- **1. 입력:** 원본 텍스트/음성/이미지 페이로드 및 Feature Flag (ON 상태)
- **2. 처리:** Gemini API 호출 및 응답값을 비즈니스 JSON으로 파싱
- **3. 출력:** AI 기반으로 완성된 객체 반환
- **4. 예외:** API 장애/타임아웃 시 **즉시 기존 Phase 3 룰베이스 로직으로 Fallback**
- **5. 테스트:** 환경 변수에서 억지로 Gemini Key를 깨뜨린 후에도 시스템이 안 멈추고 동작하는지 확인

---

**— END OF TASK LIST v5 (Beginner Edition, 완벽 가이드) —**
