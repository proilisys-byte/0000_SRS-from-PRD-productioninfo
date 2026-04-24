---
name: Feature Task
about: SRS 기반의 구체적인 개발 태스크 명세
title: "[Feature] ZUI-C-003b: Sync Queue 처리"
labels: 'feature, backend, edge, priority:high, sprint:3'
assignees: ''
---

## 🎯 Summary
- 기능명: [ZUI-C-003b] Edge→Cloud 동기화 — Sync Queue 처리
- 목적: ZUI-C-003a에서 저장된 Raw 데이터를 순차적으로 처리하는 Sync Queue를 구현한다. 처리 순서 보장, 재시도, Dead Letter Queue를 포함한다.

> ⚠️ **v2 리팩토링:** 기존 ZUI-C-003에서 분리. Processing Layer 전담.

## 🔗 References (Spec & Context)
- SRS 문서: [`05_SRS_v1.md#§4.3.3`] — Zero-UI 수집기 시퀀스
- SRS 문서: [`05_SRS_v1.md#REQ-NF-027`] — 재시도 정책
- 프로젝트 컨텍스트: [`PROJECT_CONTEXT.md`]

## 📁 Implementation Paths
- `src/lib/edge/sync-queue.ts` — Sync Queue 관리 모듈
- `src/lib/edge/dead-letter-queue.ts` — DLQ 처리

## ✅ Task Breakdown (실행 계획)
- [ ] Sync Queue 구현 (FIFO 순서 보장)
  - RAW_DATA status="PENDING" → "PROCESSING" → "COMPLETED" / "FAILED"
- [ ] 재시도 정책 (지수 백오프 + Jitter, 최대 3회)
- [ ] Dead Letter Queue 구현
  - 3회 재시도 실패 → DLQ로 이동
  - DLQ 모니터링 알림
- [ ] 배치 처리 옵션 (동시 처리 수 제한)
- [ ] 큐 상태 조회 API (ZUI-Q-001 확장)
- [ ] 단위 테스트

## 🧪 Acceptance Criteria (BDD/GWT)

**Scenario 1: 정상 큐 처리**
- Given: PENDING 상태 Raw 데이터 10건
- When: Sync Queue 처리 실행
- Then: 순차적으로 PROCESSING → COMPLETED로 상태 전이

**Scenario 2: 처리 실패 → 재시도**
- Given: AI 구조화(ZUI-C-003c) 실패
- When: 재시도 정책 적용
- Then: 지수 백오프 3회 재시도 후 최종 실패 시 DLQ 이동

**Scenario 3: DLQ 모니터링**
- Given: DLQ에 5건 이상 적재
- When: 임계치 초과 감지
- Then: 관리자 알림 발송

## ⚙️ Technical & Non-Functional Constraints
- 순서: FIFO 보장
- 재시도: 지수 백오프 + Jitter (REQ-NF-027)
- 동시성: 최대 3개 동시 처리

## 🏁 Definition of Done (DoD)
- [ ] 모든 Acceptance Criteria 충족
- [ ] DLQ 처리 로직 동작 확인
- [ ] 단위 테스트 통과
- [ ] ESLint 경고 0건

## 🚧 Dependencies & Blockers
- **Depends on:** ZUI-C-003a (Raw 저장), RES-001 (DLQ 인프라), RES-002 (Retry 중앙화)
- **Blocks:** ZUI-C-003c (AI 구조화 — 큐에서 꺼내서 처리)
