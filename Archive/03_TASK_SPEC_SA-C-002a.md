---
name: Feature Task
about: SRS 기반의 구체적인 개발 태스크 명세
title: "[Feature] SA-C-002a: 입력 데이터 정규화 (Rule-based)"
labels: 'feature, backend, audit, priority:high, sprint:1'
assignees: ''
---

## 🎯 Summary
- 기능명: [SA-C-002a] Audit 리포트 생성 — 입력 데이터 정규화 (Rule-based)
- 목적: RAW_DATA와 TEMPLATE 데이터를 AI 매핑 엔진에 전달하기 전에 rule-based로 정규화한다. enum 매핑은 rule-based 우선 처리하여 AI 의존도를 낮추고 결정론적 처리를 보장한다.

> ⚠️ **v2 리팩토링:** 기존 SA-C-002에서 분리. Deterministic Layer로서 AI 없이 동작하는 전처리 파이프라인.

## 🔗 References (Spec & Context)
> 💡 AI Agent & Dev Note: 작업 시작 전 아래 문서를 반드시 먼저 Read/Evaluate 할 것.
- SRS 문서: [`05_SRS_v1.md#REQ-FUNC-001, REQ-FUNC-002`] — Audit 리포트 PDF 생성 + 자동 매핑
- 데이터 모델: [`05_SRS_v1.md#§6.2.2, §6.2.8`] — RAW_DATA, TEMPLATE
- 프로젝트 컨텍스트: [`PROJECT_CONTEXT.md`]

## 📁 Implementation Paths
- `src/lib/audit/data-normalizer.ts` — 입력 데이터 정규화 모듈
- `src/lib/audit/enum-mapper.ts` — Rule-based enum 매핑 유틸리티
- `src/lib/validations/audit-input.ts` — 입력 검증 Zod 스키마
- `src/types/dto/audit.ts` — Audit 정규화 DTO 타입

## ✅ Task Breakdown (실행 계획)
- [ ] RAW_DATA 50건+ 존재 확인 및 데이터 추출 파이프라인
- [ ] TEMPLATE 스키마 필드 추출 및 정규화
- [ ] Rule-based enum 매핑 구현 (결정론적 처리 우선)
  - source_type → 표준 분류 코드 매핑
  - severity → ISO 등급 매핑
  - 기타 정적 enum 필드 자동 매핑
- [ ] 데이터 정규화 함수 구현
  - 날짜 포맷 통일 (ISO 8601)
  - 문자열 트리밍/정규화
  - 누락 필드 기본값 주입
- [ ] 정규화 결과 DTO 정의 및 Zod 검증
- [ ] 단위 테스트: 정규화 입출력 일관성 검증

## 🧪 Acceptance Criteria (BDD/GWT)

**Scenario 1: 정상 데이터 정규화**
- Given: 50건+ RAW_DATA와 ISO9001 TEMPLATE이 존재
- When: `normalizeAuditInput()` 호출
- Then: 모든 enum 필드가 rule-based로 매핑되고, 정규화된 DTO가 반환된다.

**Scenario 2: enum 매핑 결정론적 처리**
- Given: 동일한 입력 데이터
- When: 2회 호출
- Then: 100% 동일한 출력 (결정론적 보장)

**Scenario 3: 데이터 부족**
- Given: RAW_DATA 50건 미만
- When: 정규화 요청
- Then: 400 + `INSUFFICIENT_DATA` 에러 코드 반환

## ⚙️ Technical & Non-Functional Constraints
- AI 미사용: 이 태스크는 순수 rule-based. AI 연동 없음
- 성능: 1,000건 데이터 정규화 ≤ 2초
- 결정론적: 동일 입력 → 동일 출력 100% 보장
- 프레임워크: Next.js (C-TEC-001)

## 🏁 Definition of Done (DoD)
- [ ] 모든 Acceptance Criteria 충족
- [ ] enum 매핑 규칙이 문서화 (`docs/enum-mapping-rules.md`)
- [ ] 단위 테스트 통과
- [ ] ESLint 경고 0건

## 🚧 Dependencies & Blockers
- **Depends on:** DB-003 (RAW_DATA), DB-004 (TEMPLATE), DB-005 (AUDIT_SESSION), API-002 (Audit DTO)
- **Blocks:** SA-C-002b (AI 매핑 — 정규화된 데이터 입력으로 사용)
