# PRO ILI SMART — 프로젝트 컨텍스트 (AI 에이전트용)

> ⚠️ **ISSUE-05 반영:** 모든 TASK_SPEC에서 이 문서를 참조합니다.
> AI 에이전트가 독립적으로 태스크를 수행할 때 필요한 환경 정보를 제공합니다.

---

## 1. 디렉토리 구조

```
src/
├── app/
│   ├── (auth)/                    # 인증 관련 페이지 (로그인, 회원가입)
│   │   ├── login/page.tsx
│   │   └── signup/page.tsx
│   ├── (dashboard)/               # 인증 필요 페이지 그룹
│   │   ├── layout.tsx
│   │   ├── page.tsx               # 대시보드 메인
│   │   ├── audit/                 # Smart Audit 관련 UI
│   │   ├── nc/                    # NC 시정 관련 UI
│   │   ├── lean/                  # Lean 진단 관련 UI
│   │   ├── admin/                 # 관리자 전용 UI
│   │   └── settings/              # 설정
│   ├── api/
│   │   └── v1/                    # REST API Route Handlers
│   │       ├── auth/
│   │       ├── audit/
│   │       ├── nc/
│   │       ├── lean/
│   │       ├── edge/
│   │       └── templates/
│   ├── actions/                   # Server Actions
│   │   ├── auth/
│   │   │   ├── signUp.ts
│   │   │   ├── signIn.ts
│   │   │   └── signOut.ts
│   │   ├── audit/
│   │   │   └── generateReport.ts
│   │   ├── nc/
│   │   │   ├── createNCCase.ts
│   │   │   ├── parseAndGenerateDraft.ts
│   │   │   └── escalateNCCase.ts
│   │   ├── lean/
│   │   ├── edge/
│   │   │   └── transcribeVoice.ts
│   │   └── bulk/
│   ├── layout.tsx
│   └── page.tsx
├── components/
│   ├── ui/                        # shadcn/ui 기반 공통 컴포넌트
│   ├── audit/                     # Audit 도메인 컴포넌트
│   ├── nc/                        # NC 도메인 컴포넌트
│   ├── lean/                      # Lean 도메인 컴포넌트
│   └── edge/                      # Edge/Zero-UI 컴포넌트
├── lib/
│   ├── ai/                        # AI 엔진 모듈
│   │   ├── nc-parser.ts
│   │   ├── corrective-draft-generator.ts
│   │   └── audit-mapper.ts
│   ├── audit/                     # 감사 기록 모듈
│   ├── integrity/                 # IntegrityManager 모듈
│   │   └── integrity-manager.ts
│   ├── events/                    # 이벤트 발행/구독 모듈
│   ├── prompts/                   # AI 프롬프트 템플릿
│   ├── supabase/                  # Supabase 클라이언트 설정
│   │   ├── client.ts              # 브라우저용
│   │   ├── server.ts              # 서버용
│   │   └── admin.ts               # 관리자용
│   ├── validations/               # Zod 스키마 정의
│   └── utils/                     # 공통 유틸리티
├── types/                         # TypeScript 타입 정의
│   ├── dto/                       # API DTO 타입
│   │   ├── auth.ts
│   │   ├── audit.ts
│   │   ├── nc.ts
│   │   ├── lean.ts
│   │   ├── edge.ts
│   │   └── template.ts
│   ├── database.ts                # Prisma 확장 타입
│   └── common.ts                  # 공통 에러 코드 등
├── middleware.ts                   # Next.js 인증 미들웨어
└── styles/

prisma/
├── schema.prisma                  # Prisma 스키마 정의
├── migrations/                    # 마이그레이션 파일
└── seed.ts                        # Seed 데이터

tests/
├── unit/                          # 단위 테스트
├── integration/                   # 통합 테스트
└── fixtures/                      # 테스트 픽스처

mock/                              # MSW 또는 Mock API
├── handlers/
└── data/
```

---

## 2. 환경 변수 목록

| 변수명 | 용도 | 사용 태스크 | 필수 |
|---|---|---|---|
| `NEXT_PUBLIC_SUPABASE_URL` | Supabase 프로젝트 URL | AUTH-*, DB-* | ✅ |
| `NEXT_PUBLIC_SUPABASE_ANON_KEY` | Supabase 공개 키 (클라이언트) | AUTH-*, DB-* | ✅ |
| `SUPABASE_SERVICE_ROLE_KEY` | Supabase 서비스 역할 키 (서버) | DB-017, AUTH-C-005 | ✅ |
| `GOOGLE_AI_API_KEY` | Gemini AI API 키 | SA-C-002, NC-C-001b, ZUI-C-001~002 | ✅ |
| `DATABASE_URL` | Prisma DB 연결 문자열 | DB-001~017 | ✅ |
| `DIRECT_URL` | Prisma Direct 연결 (Supabase 풀링 우회) | DB-* | 선택 |
| `NEXT_PUBLIC_USE_MOCK` | Mock API 모드 토글 (true/false) | MOCK-006 | 개발용 |
| `NEXT_PUBLIC_APP_URL` | 애플리케이션 URL | AUTH-C-001 | ✅ |

### .env.local 템플릿
```env
# Supabase
NEXT_PUBLIC_SUPABASE_URL=https://your-project.supabase.co
NEXT_PUBLIC_SUPABASE_ANON_KEY=eyJ...
SUPABASE_SERVICE_ROLE_KEY=eyJ...

# Database (Prisma)
DATABASE_URL=file:./dev.db          # 개발: SQLite
# DATABASE_URL=postgresql://...     # 운영: PostgreSQL

# Google AI (Gemini)
GOOGLE_AI_API_KEY=AIza...

# Application
NEXT_PUBLIC_APP_URL=http://localhost:3000
NEXT_PUBLIC_USE_MOCK=true           # 개발 시 Mock 모드
```

---

## 3. 코딩 컨벤션

### 네이밍 규칙
- **파일명:** kebab-case (`integrity-manager.ts`, `create-nc-case.ts`)
- **컴포넌트:** PascalCase (`AuditReportCard.tsx`)
- **함수/변수:** camelCase (`generateReport`, `ncCaseData`)
- **상수:** UPPER_SNAKE_CASE (`MAX_RETRY_COUNT`, `GENESIS_HASH`)
- **타입/인터페이스:** PascalCase (`IntegrityEnvelope`, `STTResult`)
- **Prisma 모델:** PascalCase (`AuditLog`, `NcCase`)
- **DB 컬럼:** snake_case (`hash_sha256`, `prev_hash`)

### Server Action 규칙
- 파일 최상단에 `"use server"` 선언
- Zod 입력 검증 필수
- 반환 타입: `{ success: true, data: T } | { success: false, error: ErrorResponse }`
- 에러 코드는 `src/types/common.ts`의 공통 에러 체계 사용

### Import 규칙
- 절대 경로 사용: `@/lib/...`, `@/types/...`, `@/components/...`
- 외부 패키지 → 내부 모듈 → 타입 순서

### AI 관련 규칙
- Gemini 출력은 반드시 Zod 검증 후 사용
- temperature=0 결정론적 출력 기본
- API 실패 시 지수 백오프 3회 재시도

---

## 4. 기술 스택 요약

| 영역 | 기술 | 버전/비고 |
|---|---|---|
| 프레임워크 | Next.js (App Router) | 14.x |
| UI | Tailwind CSS + shadcn/ui | |
| ORM | Prisma | SQLite(dev) / PostgreSQL(prod) |
| 인증 | Supabase Auth | @supabase/ssr |
| AI | Vercel AI SDK + Gemini | @ai-sdk/google |
| 테스트 | Vitest + React Testing Library | |
| Mock | MSW 2.x | 개발 환경 전용 |
| 배포 | Vercel | Hobby Plan (60s timeout) |

---

**— END OF PROJECT CONTEXT —**
