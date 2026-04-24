---
name: Feature Task
about: SRS 기반의 구체적인 개발 태스크 명세
title: "[Feature] NC-C-001b: AI 사유 파싱 + 시정 초안 자동 생성 엔진"
labels: 'feature, backend, ai, nc, priority:critical, sprint:2'
assignees: ''
requires_human_review: true
---

## 🎯 Summary
- 기능명: [NC-C-001b] AI 사유 파싱 + 시정 초안 자동 생성 엔진 (Gemini AI 연동)
- 목적: NC-C-001a에서 등록된 NC 케이스의 사유 텍스트를 Gemini AI로 자동 파싱하고, 유사 NC 이력을 참조하여 시정 조치 계획서 초안을 5분 이내에 자동 생성한다.

> ⚠️ **ISSUE-02 반영:** 기존 NC-C-001에서 분리. AI 파싱 + 초안 생성에만 집중합니다.
> 🔍 **requires_human_review: true** — H 복잡도. AI 초안 생성 후 시니어 리뷰 필수.

## 🔗 References (Spec & Context)
> 💡 AI Agent & Dev Note: 작업 시작 전 아래 문서를 반드시 먼저 Read/Evaluate 할 것.
- SRS 문서: [`05_SRS_v1.md#REQ-FUNC-006`] — NC 통보 사유 파싱 엔진 (커버리지 ≥ 95%)
- SRS 문서: [`05_SRS_v1.md#REQ-FUNC-010`] — 유사 NC 사례 검색
- 관련 태스크: NC-C-001a (NC 등록 — 선행), NC-Q-002 (유사 NC 검색 — 선행)
- 프로젝트 컨텍스트: [`PROJECT_CONTEXT.md`]

## 📁 Implementation Paths
- `src/lib/ai/nc-parser.ts` — AI 사유 파싱 엔진
- `src/lib/ai/corrective-draft-generator.ts` — 시정 초안 생성기
- `src/app/actions/nc/parseAndGenerateDraft.ts` — Server Action
- `src/lib/prompts/nc-parsing.ts` — 프롬프트 템플릿

## ✅ Task Breakdown (실행 계획)
- [ ] AI 사유 파싱 엔진 구현
  - Gemini 1.5 Flash API 연동 (`@ai-sdk/google`, `generateText()`)
  - 사유 텍스트 → 구조화된 부적합 항목 분해 (nc_category, affected_items, root_cause_hint)
  - 프롬프트 엔지니어링: ISO 9001/14001/45001 NC 분류 체계 기반
  - 파싱 결과 Zod 검증 (환각 필터링)
- [ ] 유사 NC 이력 조회 연동 (NC-Q-002 호출)
  - 동일 nc_code 기반 과거 사례 최대 5건 조회
  - 유사 사례의 시정 조치 성공률 포함
- [ ] 시정 초안 자동 생성
  - Gemini AI 기반 시정 계획서 초안 생성 (유사 사례 참조 포함)
  - CORRECTIVE_ACTION 레코드 업데이트 (status: "DRAFT")
  - 시정 항목별 우선순위 자동 부여
- [ ] AI 파싱 실패 시 Fallback: 빈 초안 반환 + 수동 작성 안내
- [ ] 단위 테스트 + 통합 테스트

## 🧪 Acceptance Criteria (BDD/GWT)

**Scenario 1: 정상 파싱 및 초안 생성**
- Given: NC-C-001a로 등록된 NC_CASE가 존재하고, description이 주어짐
- When: AI 파싱 + 초안 생성을 요청함
- Then: 파싱 커버리지 ≥ 95%, 시정 계획 초안(CORRECTIVE_ACTION 1건 이상)이 5분 이내 생성.

**Scenario 2: 유사 NC 사례 참조**
- Given: 동일 nc_code로 과거 3건의 NC 존재
- When: 유사 사례 검색 수행
- Then: 최대 5건의 유사 사례가 초안 참고 데이터에 포함.

**Scenario 3: AI 파싱 실패 시 Fallback**
- Given: Gemini API가 5xx 에러 반환
- When: AI 파싱 실패
- Then: 빈 템플릿 초안 반환, "수동 작성이 필요합니다" 안내 포함.

## ⚙️ Technical & Non-Functional Constraints
- 성능: 초안 생성 응답시간 ≤ 5분 (REQ-FUNC-006 AC)
- AI 정확도: 파싱 커버리지 ≥ 95% (Appendix A.4 기준). temperature=0
- 비용: Gemini Free Tier 한도 내 (REQ-NF-018)
- AI 제어: Gemini 출력 Zod 검증 필수 (REQ-NF-029)
- 환경 변수: `GOOGLE_AI_API_KEY` 필수 (PROJECT_CONTEXT.md 참조)

## 🏁 Definition of Done (DoD)
- [ ] 모든 Acceptance Criteria 충족
- [ ] AI 파싱 커버리지 ≥ 95% 테스트 통과
- [ ] AI 실패 Fallback 정상 동작
- [ ] 단위/통합 테스트 통과
- [ ] ESLint 경고 0건
- [ ] 🔍 시니어 개발자 코드 리뷰 완료

## 🚧 Dependencies & Blockers
- **Depends on:** NC-C-001a (NC 등록 완료), NC-Q-002 (유사 NC 검색)
- **Blocks:** TEST-NC-001 (파싱 커버리지 테스트), UI-006 (시정 초안 표시)
