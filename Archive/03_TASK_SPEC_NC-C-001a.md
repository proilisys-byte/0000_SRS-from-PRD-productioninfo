---
name: Feature Task
about: SRS 기반의 구체적인 개발 태스크 명세
title: "[Feature] NC-C-001a: NC 케이스 등록 Server Action (순수 CRUD)"
labels: 'feature, backend, nc, priority:critical, sprint:2'
assignees: ''
---

## 🎯 Summary
- 기능명: [NC-C-001a] NC 케이스 등록 Server Action (순수 CRUD — Zod 검증, NC_CASE 생성)
- 목적: 원청으로부터 통보된 부적합(NC) 사유를 접수하여 NC_CASE 레코드를 생성하는 순수 CRUD Server Action을 구현한다. AI 파싱, 시정 초안 생성, 에스컬레이션은 별도 태스크(NC-C-001b, NC-C-001c)에서 처리한다.

> ⚠️ **ISSUE-02 반영:** 기존 NC-C-001에서 분리된 태스크입니다. 단일 책임 원칙에 따라 순수 CRUD만 담당합니다.

## 🔗 References (Spec & Context)
> 💡 AI Agent & Dev Note: 작업 시작 전 아래 문서를 반드시 먼저 Read/Evaluate 할 것.
- SRS 문서: [`05_SRS_v1.md#REQ-FUNC-006`] — NC 통보 사유 파싱 엔진
- SRS 문서: [`05_SRS_v1.md#§4.3.2`] — 긴급 NC 시정 패키지 시퀀스 다이어그램
- 데이터 모델: [`05_SRS_v1.md#§3.4.3`] — ERD (NC_CASE, CORRECTIVE_ACTION 엔티티)
- API 명세: API-004 (NC Response API DTO)
- 프로젝트 컨텍스트: [`PROJECT_CONTEXT.md`] — 디렉토리 구조, 환경 변수, 코딩 컨벤션
- 태스크 목록: [`TASKS/06_TASK_LIST_v1.md#Epic6`]

## 📁 Implementation Paths
- `src/app/actions/nc/createNCCase.ts` — Server Action 엔트리포인트
- `src/lib/validations/nc.ts` — Zod 입력 검증 스키마
- `src/types/nc.ts` — NC 관련 TypeScript 타입 정의
- `src/app/api/v1/nc/cases/route.ts` — API Route Handler (필요 시)

## ✅ Task Breakdown (실행 계획)
- [ ] NC 케이스 등록 Server Action 구현 (`createNCCase.ts`)
  - Zod 입력 검증: nc_code, severity(Critical/Major/Minor), description, site_id
  - NC_CASE 레코드 생성 (status: "OPEN", created_at 자동 부여)
  - CORRECTIVE_ACTION 빈 레코드 생성 (status: "PENDING", progress: 0)
- [ ] 에러 처리: API-007 공통 에러 코드 체계 연동
- [ ] 단위 테스트: NC_CASE CRUD 정상 동작 확인

## 🧪 Acceptance Criteria (BDD/GWT)

**Scenario 1: 정상 NC 등록**
- Given: severity="Major", nc_code="NC-MAT-001", description="원자재 입고 검사 미실시"가 주어짐
- When: `createNCCase` Server Action을 호출함
- Then: NC_CASE 레코드가 status="OPEN"으로 생성되고, nc_id가 반환된다.

**Scenario 2: 필수 필드 누락**
- Given: nc_code가 누락된 요청이 수신됨
- When: Zod 검증이 실행됨
- Then: 400 Bad Request와 `code: "VALIDATION_001"` 에러가 반환되며, NC_CASE가 생성되지 않는다.

**Scenario 3: 중복 NC 등록 허용**
- Given: 동일 nc_code로 NC_CASE가 이미 존재함
- When: 같은 nc_code로 새 NC 등록을 요청함
- Then: 별도의 nc_id로 정상 생성된다 (동일 코드 복수 발생 허용).

## ⚙️ Technical & Non-Functional Constraints
- 프레임워크: Next.js Server Action (C-TEC-001, C-TEC-002)
- 타입 안전: TypeScript strict 모드, Zod 스키마 검증
- 보안: NC 사유 텍스트 로깅 시 민감정보 마스킹 (REQ-NF-SEC-LOG-003)

## 🏁 Definition of Done (DoD)
- [ ] 모든 Acceptance Criteria를 충족하는가?
- [ ] NC_CASE 레코드가 정상 생성되는가?
- [ ] 단위 테스트가 추가되었고 통과하는가?
- [ ] ESLint / 정적 분석 경고가 없는가?

## 🚧 Dependencies & Blockers
- **Depends on:** DB-007 (NC_CASE 테이블), DB-008 (CORRECTIVE_ACTION 테이블), API-004 (NC Response DTO)
- **Blocks:** NC-C-001b (AI 파싱 엔진), NC-C-001c (에스컬레이션), NC-C-002 (시정 진행률 갱신), UI-006 (NC 등록 화면)
