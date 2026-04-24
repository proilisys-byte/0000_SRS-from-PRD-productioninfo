---
name: Feature Task
about: SRS 기반의 구체적인 개발 태스크 명세
title: "[Feature] ZUI-C-003d: Integrity 적용"
labels: 'feature, backend, edge, integrity, priority:high, sprint:4'
assignees: ''
---

## 🎯 Summary
- 기능명: [ZUI-C-003d] Edge→Cloud 동기화 — Integrity 적용
- 목적: ZUI-C-003c에서 구조화된 최종 데이터에 SHA-256 해시 및 타임스탬프를 부여하고, Edge 전송 무결성을 검증한다. IntegrityManager(INT-C-001)를 활용한다.

> ⚠️ **v2 리팩토링:** 기존 ZUI-C-003에서 분리. Integrity Layer 전담.

## 🔗 References (Spec & Context)
- SRS 문서: [`05_SRS_v1.md#REQ-NF-011`] — 무결성 검증 통과율 100%
- SRS 문서: [`05_SRS_v1.md#REQ-FUNC-024`] — SHA-256 무결성
- 프로젝트 컨텍스트: [`PROJECT_CONTEXT.md`]

## 📁 Implementation Paths
- `src/lib/edge/edge-integrity.ts` — Edge 무결성 검증 모듈

## ✅ Task Breakdown (실행 계획)
- [ ] Edge 전송 데이터 SHA-256 검증 (IntegrityManager 활용)
  - `verifyTransmissionIntegrity(payload, expectedHash)`
- [ ] 구조화 완료 데이터에 IntegrityEnvelope 적용
- [ ] AUDIT_LOG에 Edge 동기화 이벤트 기록
- [ ] 무결성 검증 실패 시 재전송 요청
- [ ] 단위 테스트

## 🧪 Acceptance Criteria (BDD/GWT)

**Scenario 1: 전송 무결성 검증 성공**
- Given: Edge에서 SHA-256 해시와 함께 데이터 수신
- When: 무결성 검증 실행
- Then: 해시 일치, 검증 통과율 100%

**Scenario 2: 위변조 감지**
- Given: 전송 중 데이터 변조 발생
- When: 무결성 검증 실행
- Then: 해시 불일치 감지, 재전송 요청 + 경고 로그

## ⚙️ Technical & Non-Functional Constraints
- 무결성: 검증 통과율 100% (REQ-NF-011)
- 모듈: IntegrityManager 활용 필수 (인라인 구현 금지)

## 🏁 Definition of Done (DoD)
- [ ] 모든 Acceptance Criteria 충족
- [ ] 무결성 검증 100% 테스트 통과
- [ ] ESLint 경고 0건

## 🚧 Dependencies & Blockers
- **Depends on:** ZUI-C-003c (AI 구조화 완료), INT-C-001 (IntegrityManager)
- **Blocks:** TEST-ZUI-005 (SHA-256 무결성 테스트)
