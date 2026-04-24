---
name: Feature Task
about: SRS 기반의 구체적인 개발 태스크 명세
title: "[Feature] SA-C-002b: AI 매핑 엔진 (LLM)"
labels: 'feature, backend, ai, audit, priority:critical, sprint:2'
assignees: ''
requires_human_review: true
---

## 🎯 Summary
- 기능명: [SA-C-002b] Audit 리포트 생성 — AI 매핑 (LLM)
- 목적: SA-C-002a에서 정규화된 데이터를 Gemini AI 매핑 엔진으로 ISO 규격 Audit 리포트 필드에 매핑한다. Vercel AI SDK `streamText`로 60초 타임아웃을 우회한다. AI Output Contract를 강제하여 JSON Schema 100% validate (API-008 연동).

> ⚠️ **v2 리팩토링:** 기존 SA-C-002에서 분리. LLM 전용 — rule-based에서 처리 불가한 필드만 AI 매핑.
> 🔍 **requires_human_review: true** — H 복잡도. AI 초안 생성 후 시니어 리뷰 필수.

## 🔗 References (Spec & Context)
> 💡 AI Agent & Dev Note: 작업 시작 전 아래 문서를 반드시 먼저 Read/Evaluate 할 것.
- SRS 문서: [`05_SRS_v1.md#REQ-FUNC-001, REQ-FUNC-002`]
- 시퀀스: [`05_SRS_v1.md#§4.3.1`]
- 기술 제약: [`05_SRS_v1.md#§3.6.2`] — Edge Runtime & Streaming
- 프로젝트 컨텍스트: [`PROJECT_CONTEXT.md`]

## 📁 Implementation Paths
- `src/lib/ai/audit-mapper.ts` — Gemini AI 매핑 엔진
- `src/lib/ai/audit-output-contract.ts` — AI Output Contract (JSON Schema 강제)
- `src/lib/prompts/audit-mapping.ts` — 프롬프트 템플릿

## ✅ Task Breakdown (실행 계획)
- [ ] Gemini AI 매핑 엔진 연동 (`@ai-sdk/google`, `streamText()`, temperature=0)
- [ ] AI Output Contract 강제
  - JSON Schema 100% validate (API-008과 연결)
  - 출력 구조 Zod 스키마 정의 + 검증
- [ ] Deterministic Layer 추가
  - enum 매핑은 SA-C-002a rule-based 결과 우선 사용
  - AI는 rule-based에서 미매핑된 필드만 처리
- [ ] Vercel AI SDK streamText 스트리밍 구현
- [ ] 과거 3개월 Trend 분석 섹션 데이터 연동 (REQ-FUNC-003)
- [ ] Golden Dataset 기반 정확도 검증 (train/validation 분리)
  - train용: AI 프롬프트 튜닝에 사용
  - validation용: 최종 정확도 검증에만 사용 (데이터 오염 방지)
- [ ] 단위 테스트

## 🧪 Acceptance Criteria (BDD/GWT)

**Scenario 1: 정상 AI 매핑**
- Given: SA-C-002a에서 정규화된 데이터 + ISO9001 TEMPLATE
- When: AI 매핑 엔진 호출
- Then: 필수 필드 100% 매핑, JSON Schema 100% valid. 실패율 < 0.5%

**Scenario 2: 타임아웃 우회**
- Given: 500건+ 대량 데이터
- When: `streamText()` 호출
- Then: 60초 타임아웃 없이 스트리밍 유지

**Scenario 3: AI Output Contract 위반**
- Given: AI가 JSON Schema에 맞지 않는 출력을 반환
- When: Output Contract 검증 실행
- Then: SA-C-002d Fallback으로 전환

## ⚙️ Technical & Non-Functional Constraints
- 정확도: 필수 필드 누락률 0%, 생성 실패율 < 0.5% (REQ-NF-012)
- AI 제어: temperature=0 결정론적 출력 (REQ-NF-029)
- 비용: Gemini Free Tier (15 RPM, 1,500 RPD) (REQ-NF-018)
- 보안: OpenAPI 3.0 Input Validation (REQ-NF-SEC-API-004)
- 성능: Vercel Hobby Timeout ≤ 60s → streamText 필수

## 🏁 Definition of Done (DoD)
- [ ] 모든 Acceptance Criteria 충족
- [ ] Golden Dataset validation용 정확도 ≥ 99%
- [ ] AI Output Contract JSON Schema 100% validate
- [ ] 단위/통합 테스트 통과
- [ ] ESLint 경고 0건
- [ ] 🔍 시니어 개발자 코드 리뷰 완료

## 🚧 Dependencies & Blockers
- **Depends on:** SA-C-002a (정규화된 입력), DB-006 (AUDIT_REPORT), API-008 (JSON Schema), INFRA-001 (Streaming)
- **Blocks:** SA-C-002c (결과 검증), SA-C-002d (Fallback), SA-C-003 (해시 기록), UI-003
