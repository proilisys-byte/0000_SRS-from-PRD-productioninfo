---
name: Feature Task
about: SRS 기반의 구체적인 개발 태스크 명세
title: "[Feature] NC-C-001c: Critical 에스컬레이션 이벤트 발행 + AUDIT_LOG 기록"
labels: 'feature, backend, nc, priority:high, sprint:2'
assignees: ''
---

## 🎯 Summary
- 기능명: [NC-C-001c] Critical 에스컬레이션 이벤트 발행 + AUDIT_LOG 기록
- 목적: NC 케이스 등록 시 severity=Critical 감지 → 에스컬레이션 이벤트를 발행하고, 모든 NC 등록 이벤트를 IntegrityManager를 통해 AUDIT_LOG에 기록한다.

> ⚠️ **ISSUE-02 반영:** 기존 NC-C-001에서 분리. 에스컬레이션 + 감사 기록에 집중합니다.
> NC-C-004(에스컬레이션 알림 발송)와 통합 가능한 후보입니다.

## 🔗 References (Spec & Context)
> 💡 AI Agent & Dev Note: 작업 시작 전 아래 문서를 반드시 먼저 Read/Evaluate 할 것.
- SRS 문서: [`05_SRS_v1.md#REQ-FUNC-009`] — Critical 에스컬레이션
- SRS 문서: [`05_SRS_v1.md#REQ-FUNC-024`] — AUDIT_LOG 기록
- 관련 태스크: INT-C-001 (IntegrityManager — 선행)
- 프로젝트 컨텍스트: [`PROJECT_CONTEXT.md`]

## 📁 Implementation Paths
- `src/app/actions/nc/escalateNCCase.ts` — 에스컬레이션 Server Action
- `src/lib/events/nc-escalation.ts` — 에스컬레이션 이벤트 발행 모듈
- `src/lib/audit/nc-audit-logger.ts` — NC 감사 기록 모듈

## ✅ Task Breakdown (실행 계획)
- [ ] NC 등록 이벤트 AUDIT_LOG 기록 (IntegrityManager 활용)
  - event_type: "NC_CASE_CREATED"
  - before_state: null, after_state: NC_CASE 레코드 스냅샷
- [ ] Critical 심각도 감지 시 에스컬레이션 이벤트 발행
  - severity === "Critical" 시 NC-C-004 에스컬레이션 이벤트 발행
  - 이벤트 페이로드: nc_id, severity, created_at, escalation_deadline(+24h)
- [ ] 단위 테스트

## 🧪 Acceptance Criteria (BDD/GWT)

**Scenario 1: Critical 에스컬레이션 트리거**
- Given: severity="Critical"인 NC_CASE가 NC-C-001a로 등록됨
- When: NC-C-001c가 실행됨
- Then: 에스컬레이션 이벤트가 발행되어 NC-C-004에서 24시간 카운트다운이 시작.

**Scenario 2: AUDIT_LOG 기록**
- Given: 임의 severity의 NC_CASE가 등록됨
- When: NC 등록 이벤트 감사 기록이 실행됨
- Then: AUDIT_LOG에 event_type="NC_CASE_CREATED" 레코드가 SHA-256 해시와 함께 기록.

**Scenario 3: Non-Critical은 에스컬레이션 미발행**
- Given: severity="Minor"인 NC_CASE가 등록됨
- When: NC-C-001c가 실행됨
- Then: AUDIT_LOG 기록만 수행되고, 에스컬레이션 이벤트는 발행되지 않음.

## ⚙️ Technical & Non-Functional Constraints
- 무결성: IntegrityManager 모듈 활용 필수 (인라인 SHA-256 구현 금지)
- 프레임워크: Next.js Server Action (C-TEC-001)

## 🏁 Definition of Done (DoD)
- [ ] 모든 Acceptance Criteria 충족
- [ ] AUDIT_LOG에 NC 등록 이벤트가 기록되는가?
- [ ] Critical 에스컬레이션 이벤트가 정상 발행되는가?
- [ ] 단위 테스트 통과
- [ ] ESLint 경고 0건

## 🚧 Dependencies & Blockers
- **Depends on:** NC-C-001a (NC 등록), INT-C-001 (IntegrityManager)
- **Blocks:** NC-C-004 (에스컬레이션 알림 발송 — 이벤트 소비)
