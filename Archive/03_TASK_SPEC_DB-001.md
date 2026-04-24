---
name: Feature Task
about: SRS 기반의 구체적인 개발 태스크 명세
title: "[Feature] DB-001: SITE 테이블 스키마 및 Prisma 마이그레이션 작성"
labels: 'feature, backend, database, priority:critical, sprint:0'
assignees: ''
---

## 🎯 Summary
- 기능명: [DB-001] SITE 테이블 스키마 및 Prisma 마이그레이션 작성
- 목적: 전체 시스템의 데이터 계층 기반이 되는 SITE(사업장) 테이블을 Prisma ORM 기반으로 정의하고 마이그레이션을 생성한다. 모든 업무 엔티티(RAW_DATA, AUDIT_SESSION, NC_CASE, LEAN_DIAGNOSIS 등)가 SITE를 FK로 참조하므로, 이 태스크는 전체 크리티컬 패스의 최초 시작점이다.

## 🔗 References (Spec & Context)
> 💡 AI Agent & Dev Note: 작업 시작 전 아래 문서를 반드시 먼저 Read/Evaluate 할 것.
- SRS 문서: [`05_SRS_v1.md#§3.4.3`] — ERD (Entity-Relationship Diagram)
- SRS 문서: [`05_SRS_v1.md#§4.2.11`] — 멀티테넌트 격리 정책 (tenant_id 포함 설계)
- SRS 문서: [`05_SRS_v1.md#§1.2.3`] — 기술 제약사항 (C-TEC-003: Prisma + SQLite/Supabase)
- SRS 문서: [`05_SRS_v1.md#§3.6.1`] — Prisma Client Extensions SQLite↔PostgreSQL 호환
- 태스크 목록: [`TASKS/06_TASK_LIST_v1.md#Epic1`] — DB Schema 태스크 목록
- 프로젝트 컨텍스트: [`PROJECT_CONTEXT.md`] — 디렉토리 구조, 환경 변수, 코딩 컨벤션

## 📁 Implementation Paths
- `prisma/schema.prisma` — Prisma 스키마 정의 (메인)
- `prisma/migrations/` — 마이그레이션 파일 생성 위치
- `prisma/seed.ts` — Seed 데이터 (후속 MOCK-001에서 확장)
- `src/types/database.ts` — Prisma 확장 타입

## ✅ Task Breakdown (실행 계획)
- [ ] Prisma 프로젝트 초기화 (`prisma init`) 및 `schema.prisma` 기본 설정
- [ ] `datasource` 블록 구성 — SQLite(dev) / PostgreSQL(prod) 전환 가능 설정
- [ ] `TENANT` 모델 정의 (tenant_id UUID PK, name, industry_type, status enum, timestamps)
- [ ] `SITE` 모델 정의 (site_id UUID PK, tenant_id FK, company_name, address, contact_info, timestamps)
- [ ] TENANT ↔ SITE 관계 설정 (1:N, onDelete CASCADE 정책)
- [ ] 인덱스 설정 (tenant_id 기반 조회 최적화, company_name 검색 인덱스)
- [ ] `prisma migrate dev` 실행 및 마이그레이션 파일 생성 검증
- [ ] SQLite 환경에서 마이그레이션 정상 적용 확인
- [ ] 단위 테스트: SITE CRUD 정상 동작 확인 (Prisma Client 활용)

## 🧪 Acceptance Criteria (BDD/GWT)

**Scenario 1: SITE 레코드 정상 생성**
- Given: 유효한 TENANT 레코드가 DB에 존재함
- When: tenant_id, company_name, address를 포함한 SITE 생성 요청을 수행함
- Then: SITE 레코드가 DB에 정상 생성되고, auto-generated UUID site_id를 반환한다.

**Scenario 2: TENANT 삭제 시 CASCADE 동작**
- Given: TENANT에 연결된 SITE 레코드가 3건 존재함
- When: 해당 TENANT를 삭제함
- Then: 연결된 모든 SITE 레코드가 함께 삭제되며, 고아 레코드가 0건이다.

**Scenario 3: 중복 사업장명 허용**
- Given: 동일 tenant_id 하에 company_name이 "A공장"인 SITE가 이미 존재함
- When: 동일 tenant_id로 company_name "A공장"인 SITE 생성을 시도함
- Then: 별도의 site_id로 정상 생성된다. (사업장명은 Unique 제약 없음, 동명 사업장 허용)

**Scenario 4: SQLite ↔ PostgreSQL 호환성**
- Given: SQLite 환경에서 작성된 마이그레이션 파일이 존재함
- When: DATABASE_URL을 PostgreSQL로 전환하고 `prisma migrate deploy` 실행함
- Then: 마이그레이션이 에러 없이 적용되고, 동일한 스키마가 PostgreSQL에 생성된다.

## ⚙️ Technical & Non-Functional Constraints
- 프레임워크: Prisma ORM + Next.js App Router 환경 (C-TEC-001, C-TEC-003)
- DB 전환: SQLite(개발) ↔ PostgreSQL/Supabase(운영) 무중단 전환 지원 (§3.6.1)
- 멀티테넌시: MVP 단계는 Single Tenant이나, tenant_id 필드 필수 포함 (§4.2.11)
- UUID: 모든 PK는 UUID v4 사용 (`@default(uuid())`)
- 타임스탬프: created_at, updated_at 자동 관리 (`@default(now())`, `@updatedAt`)
- 명명 규칙: 테이블명 PascalCase(Prisma 모델), 컬럼명 snake_case (`@@map` 활용)

## 🏁 Definition of Done (DoD)
- [ ] 모든 Acceptance Criteria를 충족하는가?
- [ ] Prisma 마이그레이션 파일이 생성되고 SQLite에서 정상 적용되는가?
- [ ] TENANT ↔ SITE 관계가 ERD(§3.4.3)와 일치하는가?
- [ ] 단위 테스트(Prisma Client CRUD)가 추가되었고 통과하는가?
- [ ] ESLint / Prisma validate 경고가 없는가?
- [ ] schema.prisma 파일이 코드 리뷰 가능한 상태인가?

## 🚧 Dependencies & Blockers
- **Depends on:** None (전체 크리티컬 패스의 최초 시작점)
- **Blocks:** DB-002, DB-003, DB-004, DB-005, DB-007, DB-009, DB-010, DB-011, DB-012, DB-014, DB-015, DB-016 (모든 후속 DB 스키마 태스크)
