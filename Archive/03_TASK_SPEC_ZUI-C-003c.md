---
name: Feature Task
about: SRS 기반의 구체적인 개발 태스크 명세
title: "[Feature] ZUI-C-003c: AI 구조화 (비동기)"
labels: 'feature, backend, ai, edge, priority:high, sprint:3'
assignees: ''
requires_human_review: true
---

## 🎯 Summary
- 기능명: [ZUI-C-003c] Edge→Cloud 동기화 — AI 구조화 (비동기)
- 목적: Sync Queue(ZUI-C-003b)에서 전달된 Raw 데이터를 Gemini LLM으로 구조화한다. 비동기 처리로 Edge 응답 지연을 방지하고, AI 실패 시 수동 입력 Fallback으로 전환한다.

> ⚠️ **v2 리팩토링:** 기존 ZUI-C-003에서 분리. AI 처리만 전담 (비동기).
> 🔍 **requires_human_review: true** — H 복잡도.

## 🔗 References (Spec & Context)
- SRS 문서: [`05_SRS_v1.md#REQ-FUNC-013`] — LLM 구조화
- SRS 문서: [`05_SRS_v1.md#§4.3.3`] — Zero-UI 시퀀스
- 프로젝트 컨텍스트: [`PROJECT_CONTEXT.md`]

## 📁 Implementation Paths
- `src/lib/edge/ai-structurizer.ts` — AI 구조화 모듈
- `src/lib/prompts/edge-structuring.ts` — 프롬프트 템플릿

## ✅ Task Breakdown (실행 계획)
- [ ] Raw 데이터 → 구조화 데이터 변환 (Gemini API)
  - 음성 전사 텍스트 → 불량 접수 구조화 데이터
  - 이미지 분석 결과 → 외관 판정 구조화 데이터
- [ ] 비동기 처리 (Edge 응답과 분리)
- [ ] AI Output Contract 강제 (Zod 검증)
- [ ] 구조화 결과 RAW_DATA 업데이트 (structured_data 필드)
- [ ] AI 실패 시 status="NEEDS_MANUAL" 마킹
- [ ] 단위 테스트

## 🧪 Acceptance Criteria (BDD/GWT)

**Scenario 1: 정상 AI 구조화**
- Given: Raw 음성 전사 텍스트가 큐에서 전달됨
- When: AI 구조화 실행
- Then: structured_data 필드가 채워지고 status="COMPLETED"

**Scenario 2: AI 실패 → 수동 입력 마킹**
- Given: Gemini API 장애
- When: 3회 재시도 후 실패
- Then: status="NEEDS_MANUAL", 수동 입력 UI 전환 이벤트 발행

## ⚙️ Technical & Non-Functional Constraints
- AI 제어: temperature=0, Zod 검증 (REQ-NF-029)
- 비용: Gemini Free Tier 한도 내 (REQ-NF-018)
- 비동기: Edge 응답 블로킹 금지

## 🏁 Definition of Done (DoD)
- [ ] 모든 Acceptance Criteria 충족
- [ ] AI 실패 시 Fallback 동작 확인
- [ ] 🔍 시니어 리뷰 완료

## 🚧 Dependencies & Blockers
- **Depends on:** ZUI-C-003b (Sync Queue), ZUI-C-001 (STT), ZUI-C-002 (Vision)
- **Blocks:** ZUI-C-003d (Integrity 적용)
