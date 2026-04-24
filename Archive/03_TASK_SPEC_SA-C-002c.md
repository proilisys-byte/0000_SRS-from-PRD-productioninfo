---
name: Feature Task
about: SRS 기반의 구체적인 개발 태스크 명세
title: "[Feature] SA-C-002c: 결과 검증 (Rule + Schema)"
labels: 'feature, backend, audit, priority:high, sprint:2'
assignees: ''
---

## 🎯 Summary
- 기능명: [SA-C-002c] Audit 리포트 생성 — 결과 검증 (Rule + Schema)
- 목적: SA-C-002b의 AI 매핑 결과를 rule-based + JSON Schema 이중 검증한다. 환각 필터링, 누락 필드 카운팅, TEMPLATE 정합성 확인을 수행하여 최종 AUDIT_REPORT 품질을 보장한다.

> ⚠️ **v2 리팩토링:** 기존 SA-C-002에서 분리. AI 출력 후처리 전담.

## 🔗 References (Spec & Context)
- SRS 문서: [`05_SRS_v1.md#REQ-FUNC-002`] — 자동 매핑 정확도
- SRS 문서: [`05_SRS_v1.md#REQ-NF-012`] — 필수 필드 누락률 0%
- 프로젝트 컨텍스트: [`PROJECT_CONTEXT.md`]

## 📁 Implementation Paths
- `src/lib/audit/result-validator.ts` — 결과 검증 모듈
- `src/lib/audit/hallucination-filter.ts` — 환각 필터링
- `src/lib/validations/audit-report.ts` — Zod 출력 검증 스키마

## ✅ Task Breakdown (실행 계획)
- [ ] AI 출력 JSON Schema 검증 (API-008 연동)
- [ ] 환각 필터링 로직
  - TEMPLATE에 정의되지 않은 필드 감지
  - RAW_DATA에 근거하지 않는 값 감지
- [ ] 누락 필드 카운팅 + 누락률 계산
- [ ] TEMPLATE 정합성 검증 (필수/선택 필드 매칭)
- [ ] 검증 실패 시 SA-C-002d Fallback 트리거
- [ ] AUDIT_REPORT 레코드 저장 (version 자동 증가, SHA-256 해시)
- [ ] 검증 결과 리포트 DTO 정의
- [ ] 단위 테스트

## 🧪 Acceptance Criteria (BDD/GWT)

**Scenario 1: 검증 통과**
- Given: AI 매핑 결과가 JSON Schema 100% valid
- When: 결과 검증 실행
- Then: AUDIT_REPORT DB 저장, 누락률 0% 확인

**Scenario 2: 환각 감지**
- Given: AI 출력에 TEMPLATE에 없는 필드가 포함됨
- When: 환각 필터링 실행
- Then: 해당 필드가 제거되고 경고 로그 기록

**Scenario 3: 검증 실패 → Fallback**
- Given: 필수 필드 누락률 > 5%
- When: 검증 결과 기준치 미달
- Then: SA-C-002d Fallback 프로세스로 전환

## ⚙️ Technical & Non-Functional Constraints
- 정확도: 필수 필드 누락률 = 0% (REQ-NF-012)
- 무결성: SHA-256 + 타임스탬프 (REQ-FUNC-024)
- 검증: Zod + JSON Schema 이중 검증

## 🏁 Definition of Done (DoD)
- [ ] 모든 Acceptance Criteria 충족
- [ ] 환각 필터링 테스트 통과
- [ ] 단위 테스트 통과
- [ ] ESLint 경고 0건

## 🚧 Dependencies & Blockers
- **Depends on:** SA-C-002b (AI 매핑 결과), API-008 (JSON Schema)
- **Blocks:** SA-C-003 (해시 체인 기록), UI-003 (리포트 표시)
