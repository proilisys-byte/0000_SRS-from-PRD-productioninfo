---
name: Feature Task
about: SRS 기반의 구체적인 개발 태스크 명세
title: "[Feature] SA-C-002d: Fallback (AI 실패 시)"
labels: 'feature, backend, audit, priority:high, sprint:2'
assignees: ''
---

## 🎯 Summary
- 기능명: [SA-C-002d] Audit 리포트 생성 — Fallback (AI 실패 시)
- 목적: Gemini AI 매핑(SA-C-002b)이 실패하거나 결과 검증(SA-C-002c)에서 기준치 미달 시 대안 처리를 수행한다. rule-based 매핑 결과만으로 부분 리포트를 생성하거나, 수동 매핑 UI로 전환한다.

> ⚠️ **v2 리팩토링:** 기존 SA-C-002에서 분리. AI 실패 시 시스템 중단 방지를 위한 필수 안전망.

## 🔗 References (Spec & Context)
- SRS 문서: [`05_SRS_v1.md#REQ-NF-027`] — 외부 서비스 실패 재시도 정책
- SRS 문서: [`05_SRS_v1.md#REQ-NF-027-B`] — 서킷 브레이커 + Fallback
- 프로젝트 컨텍스트: [`PROJECT_CONTEXT.md`]

## 📁 Implementation Paths
- `src/lib/audit/audit-fallback.ts` — Fallback 처리 모듈
- `src/lib/audit/partial-report-generator.ts` — 부분 리포트 생성기

## ✅ Task Breakdown (실행 계획)
- [ ] AI 실패 감지 로직 (Gemini API 5xx, 타임아웃, Output Contract 위반)
- [ ] 지수 백오프 3회 재시도 → 서킷 브레이커 → Fallback 전환
- [ ] Fallback 전략 구현
  - Strategy 1: SA-C-002a rule-based 결과만으로 부분 리포트 생성
  - Strategy 2: 수동 매핑 UI 전환 이벤트 발행
- [ ] 부분 리포트 표시 (매핑률 표기, 미매핑 필드 하이라이트)
- [ ] Fallback 이벤트 AUDIT_LOG 기록
- [ ] 단위 테스트

## 🧪 Acceptance Criteria (BDD/GWT)

**Scenario 1: Gemini API 장애 시 Fallback**
- Given: Gemini API 5xx 또는 무응답
- When: 지수 백오프 3회 재시도 후 모두 실패
- Then: 서킷 브레이커 → 503 반환 대신 rule-based 부분 리포트 생성

**Scenario 2: Output Contract 위반 시 Fallback**
- Given: SA-C-002c 검증에서 기준치 미달
- When: Fallback 전환 트리거
- Then: 부분 리포트 생성 + 미매핑 필드 목록 반환

**Scenario 3: Feature Flag OFF 시 AI 우회**
- Given: Feature Flag `AI_AUDIT_MAPPING=false`
- When: 리포트 생성 요청
- Then: SA-C-002b를 건너뛰고 SA-C-002a 결과만으로 부분 리포트 생성

## ⚙️ Technical & Non-Functional Constraints
- 재시도: 지수 백오프 + Jitter, 최대 3회 (REQ-NF-027)
- 서킷 브레이커: 5회 연속 실패 시 Open (INFRA-014 연동)
- Feature Flag: RES-003 연동

## 🏁 Definition of Done (DoD)
- [ ] 모든 Acceptance Criteria 충족
- [ ] AI 실패 시 시스템 중단 0건
- [ ] Feature Flag OFF 시 정상 동작
- [ ] 단위 테스트 통과
- [ ] ESLint 경고 0건

## 🚧 Dependencies & Blockers
- **Depends on:** SA-C-002a (rule-based 결과), SA-C-002b (AI 매핑), SA-C-002c (검증), INFRA-013 (재시도), INFRA-014 (서킷 브레이커)
- **Blocks:** UI-003 (리포트 생성 화면 — Fallback UI 포함)
