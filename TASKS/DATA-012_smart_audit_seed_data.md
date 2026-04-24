---
name: Feature Task
about: PRD&SRS 기반의 구체적인 개발 태스크 명세
title: "[DATA] DATA-012: Smart Audit Seed Data"
labels: 'data, smart-audit, spec, priority:medium'
assignees: ''
---

## 🎯 Summary
- 기능명: [DATA-012] Smart Audit Seed Data
- 목적: 통합 테스트(TEST-F1-xxx) 및 로컬 개발 환경 셋업, 경영진 데모(Sprint 1) 시나리오를 충족하기 위해 필요한 기초 기준 정보 및 가상의 Audit Session 이력 데이터를 구성한다.

## 🔗 References (Spec & Context)
> 작업 시작 전 아래 문서를 반드시 먼저 읽고 정합성을 유지할 것.
- 데이터 스키마: `DATA-SCHEMA_v1.md`
- 테스트 시나리오: `TEST-F1-001`, `TEST-F1-002`
- 데모 및 결정 로그: `08_DECISION_LOG_v1.md` Section 6

## 🧭 Scope
### In Scope
- 테스트 및 데모용 Tenant 및 Site 기초 데이터 (Seed) 구성 ("미래정밀" 등)
- 검증용 권한 별 Mock Users (Admin, User)
- 기준정보 마스터 데이터 (제품, 공정) 삽입
- 상태별(Draft, Submitted, Generated 등) Audit Session 더미 데이터 및 관련 음성/텍스트 데이터
- 연관된 Audit Logs 초기 데이터 
- Sprint 1 데모 검증 데이터 셋업 (음성 파일, CSV 샘플 등)

### Out of Scope
- 프로덕션 DB에 직접 데이터 삽입 금지 (운영/Production 환경에서는 최소 기초 권한만 주입)

## 🧱 Preconditions
- 선행 완료 문서:
  - `DATA-SCHEMA_v1.md`
- 시작 가능 조건:
  - DB 스키마 픽스 및 마이그레이션(Prisma Migrate 등) 인프라 확보

## 📄 Detailed Seed Data Specification

### 1. Seed Data 구성 항목
Sprint 1 통합 테스트 및 경영진 데모 시나리오를 충족하기 위한 최소 데이터셋 규격입니다.

* **테넌트(고객사)**: 1개 (이름: "미래정밀")
* **계정 설정**:
  * Admin: 1개 (admin@mirae.com / role: admin)
  * User: 2개 (worker1@mirae.com, worker2@mirae.com / role: user)
* **기준정보 마스터 데이터**:
  * 제품 5종: P-001(기어), P-002(샤프트), P-003(베어링), P-004(실린더), P-005(밸브)
  * 공정 10개: 주조, 가공, 연마, 조립, 도색, 검사 등
* **Bulk Import용 샘플 CSV 파일 (10행 예시)**:
  ```csv
  제품코드,제품명,표준단가,공정명
  P-001,기어,15000,가공
  P-001,기어,15000,조립
  P-002,샤프트,20000,가공
  ... (총 10행)
  ```
* **음성 입력 시나리오용 샘플 발화문 (제조 도메인)**:
  1. "기어 조립 공정, 작업 수량 백 개, 스크래치 불량 두 개 발생"
  2. "샤프트 연마 끝났고 정상 수량 오십 개"
  3. "베어링 가공 중에 공구 파손으로 세 개 불량, 코드는 D-004"
* **Audit 세션**: 2개
  * 세션 1: 어제 일자, 상태 `COMPLETED`
  * 세션 2: 금일 일자, 상태 `IN_PROGRESS` (데모 시 이 세션에 STT 데이터를 적재함)

### 2. Seed 스크립 구조
로컬 환경 초기화 및 CI/CD 파이프라인을 위한 Prisma `seed.ts` 구조입니다.

```typescript
// prisma/seed.ts
import { PrismaClient } from '@prisma/client';

const prisma = new PrismaClient();

async function main() {
  const isLocal = process.env.NODE_ENV === 'development';

  if (isLocal) {
    console.log("🌱 로컬 개발 환경 Seed 데이터 삽입 시작...");
    
    // 1. 테넌트 생성
    const tenant = await prisma.tenant.create({
      data: { name: '미래정밀' }
    });

    // 2. 권한 및 사용자 생성 (비밀번호는 auth 로직에 의해 생성, 여기서는 프로필만)
    // 3. 기준정보(제품, 공정) 삽입
    // 4. 세션 더미 데이터 삽입
    
    console.log("✅ Seed 완료");
  } else {
    console.log("⚠️ 운영/스테이징 환경에서는 Seed 스크립트가 제한적으로 동작합니다.");
    // 운영 환경용 기초 권한 세팅 등 최소 데이터만 유지
  }
}

main()
  .catch((e) => {
    console.error(e);
    process.exit(1);
  })
  .finally(async () => {
    await prisma.$disconnect();
  });
```

### 3. Sprint 1 데모 시나리오 검증 데이터 준비
`08_DECISION_LOG_v1.md` Section 6 기준을 만족하기 위해 다음 데이터를 데모 데이터베이스에 준비합니다.
1. **STT 기반 현장 입력 증명용**: 사전에 준비된 명확한 음성 파일(.m4a) 3개
2. **Bulk Import 증명용**: 오류가 포함된 CSV(실패 케이스 검증용) 1개, 정상 CSV 1개
3. **Smart Audit 증명용**: 매핑에 충분한 분량의 (20개 이상의) STT 텍스트 엔트리가 담긴 `IN_PROGRESS` 상태의 세션 1개
4. **RBAC 분리 증명용**: 브라우저 창 2개(Admin 계정 세션, User 계정 세션)
5. **Insert-only 로깅 증명용**: Admin 토큰 또는 Service Key가 적용된 DB 콘솔 연결 상태

## ✅ Task Breakdown (실행 계획)
- [ ] `seed.ts` (또는 SQL 스크립트) 파일 생성 및 초기화 로직 셋업
- [ ] 의존성 순서에 따른 삽입 로직 구성: Tenants("미래정밀") -> Users(Admin/User) -> 기준정보 마스터 데이터(제품/공정)
- [ ] Smart Audit 테스트를 위한 세션 더미 데이터 및 STT 샘플 발화문 생성 (최소 20건 텍스트 매핑용 세션 포함)
- [ ] Bulk Import 검증용 CSV 샘플 파일 및 음성 파일(.m4a) 에셋 준비
- [ ] 멱등성 보장 로직 (이미 시드 데이터가 있으면 스킵 또는 리셋 후 재삽입) 적용
- [ ] 테스트 실행 전 Seed 주입을 위한 npm/yarn 명령어 구성 (`npm run db:seed`)

## 🧪 Acceptance Criteria (BDD / GWT)
Scenario 1: 시드 데이터 정상 주입
- Given: 초기화된 빈 로컬 데이터베이스가 있다.
- When: 시드 주입 명령(e.g., `npm run db:seed`)을 실행한다.
- Then: 에러 없이 데이터가 삽입되며, "미래정밀" 테넌트 및 관련 테스트 유저, 기준정보 데이터가 DB에 조회된다.

Scenario 2: 멱등성 유지 (재실행 보호)
- Given: 이미 시드 데이터가 주입된 데이터베이스가 있다.
- When: 시드 주입 명령을 다시 실행한다.
- Then: 충돌 에러(Unique Constraint)가 발생하지 않고, 안전하게 덮어쓰거나 무시된다.

Scenario 3: 데모 시나리오 조건 충족
- Given: 로컬 환경에서 시드 주입이 완료되었다.
- When: 데모 시나리오에 따라 Admin과 User 계정으로 로그인한다.
- Then: 2개 이상의 가상 Audit Session(하나는 완료, 하나는 진행중)이 조회되며, Bulk Import 테스트를 위한 CSV 파일 구조와 STT 검증용 발화문 데이터가 준비되어 있다.

## 🔐 Technical / Domain Constraints
- 권한/RBAC 규칙:
  - 생성되는 User 데이터는 반드시 지정된 Role(admin, user 등)을 보유해야 함
- tenant/site 범위 규칙:
  - 세션 데이터는 반드시 유효한 `tenant_id` 외래키를 참조해야 함
- 환경 분리:
  - `process.env.NODE_ENV === 'development'` 일 때만 대량의 더미 데이터를 주입하고, 운영 환경에서는 동작을 제한할 것

## 📦 Deliverables
- 산출 문서/코드/테스트/Mock/연계 결과:
  - `prisma/seed.ts` 스크립트 또는 SQL Init 스크립트
  - 샘플 자산 파일들 (CSV, m4a 오디오 등)

## 🏁 Definition of Done (DoD)
- [ ] PRD/SRS 및 관련 기존 TASK 문서와 충돌하지 않는가?
- [ ] Acceptance Criteria를 모두 반영했는가?
- [ ] 경영진 데모 시나리오에 필요한 모든 데이터 구성을 포함했는가?
- [ ] 관련 후속 문서 또는 구현 단계로 바로 넘길 수 있는가?

## 🚧 Dependencies & Blockers
- Depends on:
  - `DATA-SCHEMA_v1.md`
- Blocks:
  - E2E Test 및 백엔드 통합 검증 단계 전반
  - Sprint 1 데모 시연

## 📝 Notes for Dev / AI Agent
- 이 TASK의 핵심 초점:
  - 테스트와 시연의 일관성을 위해 멱등성(Idempotency)을 갖춘 스크립트를 작성하는 것이다.
  - 제공된 Seed 데이터 규격은 MVP 데모 및 QA 통과를 위한 최소 필수 요건이다.
