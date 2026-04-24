---
name: Feature Task
about: SRS 기반의 구체적인 개발 태스크 명세
title: "[Feature] ZUI-C-003a: Raw 데이터 저장 (무조건 성공)"
labels: 'feature, backend, edge, priority:critical, sprint:1'
assignees: ''
---

## 🎯 Summary
- 기능명: [ZUI-C-003a] Edge→Cloud 동기화 — Raw 데이터 저장 (무조건 성공)
- 목적: Edge 디바이스에서 전송된 음성/이미지 Raw 데이터를 가공 없이 즉시 RAW_DATA 테이블에 저장한다. AI 구조화나 무결성 검증 이전에 데이터 유실을 원천 차단하는 "무조건 성공" 계층이다.

> ⚠️ **v2 리팩토링:** 기존 ZUI-C-003에서 분리. "데이터는 일단 저장, 처리는 나중에" 원칙.

## 🔗 References (Spec & Context)
- SRS 문서: [`05_SRS_v1.md#REQ-FUNC-013`] — Edge→Cloud 실시간 동기화
- SRS 문서: [`05_SRS_v1.md#REQ-NF-010`] — 전송 유실률 0%
- 프로젝트 컨텍스트: [`PROJECT_CONTEXT.md`]

## 📁 Implementation Paths
- `src/app/actions/edge/saveRawData.ts` — Raw 저장 Server Action
- `src/lib/edge/raw-data-saver.ts` — Raw 저장 모듈

## ✅ Task Breakdown (실행 계획)
- [ ] Raw 데이터 수신 Server Action 구현
  - 입력 검증: 최소한의 메타데이터(source_type, device_id, timestamp)
  - RAW_DATA 테이블 즉시 Insert (status: "PENDING")
- [ ] 저장 실패 시 로컬 큐 적재 (Dead Letter 대비)
- [ ] 저장 완료 응답 즉시 반환 (후속 처리와 분리)
- [ ] 단위 테스트: Insert 성공률 100% 검증

## 🧪 Acceptance Criteria (BDD/GWT)

**Scenario 1: 정상 Raw 저장**
- Given: Edge에서 음성 데이터(WAV, 2MB)가 전송됨
- When: `saveRawData` Server Action 호출
- Then: RAW_DATA 테이블에 status="PENDING"으로 즉시 저장, 200 응답 반환

**Scenario 2: 대용량 파일**
- Given: 4MB 이미지 파일 전송
- When: Raw 저장 요청
- Then: 정상 저장 (4.5MB 미만이므로 직접 저장)

**Scenario 3: 메타데이터 최소 검증**
- Given: device_id 누락
- When: Raw 저장 요청
- Then: 400 반환, 데이터 미저장

## ⚙️ Technical & Non-Functional Constraints
- 유실률: 0% (REQ-NF-010)
- 성능: p99 ≤ 500ms (즉시 저장)
- AI 미사용: 이 태스크에서 AI 처리 없음

## 🏁 Definition of Done (DoD)
- [ ] 모든 Acceptance Criteria 충족
- [ ] 저장 성공률 100% 테스트 통과
- [ ] ESLint 경고 0건

## 🚧 Dependencies & Blockers
- **Depends on:** DB-003 (RAW_DATA 테이블), API-003 (Edge Sync DTO)
- **Blocks:** ZUI-C-003b (Sync queue), ZUI-C-003c (AI 구조화)
