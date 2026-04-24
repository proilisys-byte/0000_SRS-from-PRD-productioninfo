---
name: Feature Task
about: SRS 기반의 구체적인 개발 태스크 명세
title: "[Feature] SA-C-002: Audit 리포트 생성 — Gemini AI 매핑 엔진 연동"
labels: 'feature, backend, ai, priority:critical, sprint:1'
assignees: ''
requires_human_review: true
---

## ⚠️ DEPRECATED — 이 태스크는 4개로 분리되었습니다 (v2 리팩토링)

> **이 파일은 더 이상 사용하지 마세요.** 아래 4개 SPEC으로 대체되었습니다:
> - [`07_TASK_SPEC_SA-C-002a.md`](./07_TASK_SPEC_SA-C-002a.md) — 입력 데이터 정규화 (rule-based)
> - [`07_TASK_SPEC_SA-C-002b.md`](./07_TASK_SPEC_SA-C-002b.md) — AI 매핑 (LLM)
> - [`07_TASK_SPEC_SA-C-002c.md`](./07_TASK_SPEC_SA-C-002c.md) — 결과 검증 (rule + schema)
> - [`07_TASK_SPEC_SA-C-002d.md`](./07_TASK_SPEC_SA-C-002d.md) — Fallback (AI 실패 시)

---

## :dart: Summary (원본 — 참고용)
- 기능명: [SA-C-002] Audit 리포트 생성 Server Action — Gemini AI 매핑 엔진 연동
- 목적: 현장 RAW_DATA와 원청 TEMPLATE을 기반으로 Gemini AI 매핑 엔진을 활용하여 ISO 규격 Audit 증빙 리포트를 자동 생성한다. Vercel AI SDK `streamText`로 60초 타임아웃을 우회하고 클라이언트 측 PDF 렌더링으로 최종 결과물을 제공한다.

## :link: References (Spec & Context)
> :bulb: AI Agent & Dev Note: 작업 시작 전 아래 문서를 반드시 먼저 Read/Evaluate 할 것.
- SRS 문서: [`05_SRS_v1.md#REQ-FUNC-001, REQ-FUNC-002`] — Audit 리포트 PDF 생성 + 자동 매핑
- 시퀀스 다이어그램: [`05_SRS_v1.md#§4.3.1`] — Smart Audit 엔진 시퀀스
- 데이터 모델: [`05_SRS_v1.md#§6.2.3~6.2.4, §6.2.8`] — AUDIT_SESSION, AUDIT_REPORT, TEMPLATE
- API 명세: [`05_SRS_v1.md#§6.1 #1~2`] — POST/GET /api/v1/audit/reports
- 기술 제약: [`05_SRS_v1.md#§3.6.2`] — Edge Runtime & Streaming 아키텍처
- 프로젝트 컨텍스트: [`PROJECT_CONTEXT.md`] — 디렉토리 구조, 환경 변수, 코딩 컨벤션

> 🔍 **requires_human_review: true** — H 복잡도. AI 초안 생성 후 시니어 리뷰 필수.

## 📁 Implementation Paths
- `src/app/actions/audit/generateReport.ts` — Server Action 엔트리포인트
- `src/lib/ai/audit-mapper.ts` — Gemini AI 매핑 엔진
- `src/lib/validations/audit-report.ts` — Zod 입력 검증 스키마
- `src/types/dto/audit.ts` — Audit DTO 타입 정의

## :white_check_mark: Task Breakdown (실행 계획)
- [ ] Server Action 엔트리포인트 구현 (`generateReport.ts`, Zod 입력 검증)
- [ ] RAW_DATA 50건+ 존재 확인 및 TEMPLATE 스키마 추출 파이프라인
- [ ] Gemini AI 매핑 엔진 연동 (`@ai-sdk/google`, `streamText()`, temperature=0)
- [ ] AI 출력 후처리: Zod 검증, 누락 필드 카운팅, 환각 필터
- [ ] AUDIT_REPORT 레코드 저장 (version 자동 증가, SHA-256 해시)
- [ ] 클라이언트 스트리밍 응답 구현 (html2pdf.js 연동 데이터 구조)
- [ ] 과거 3개월 Trend 분석 섹션 생성 (REQ-FUNC-003)
- [ ] 단위 테스트 + Golden Dataset 기반 정확도 검증 테스트

## :test_tube: Acceptance Criteria (BDD/GWT)

**Scenario 1: 정상 리포트 생성**
- Given: 50건+ RAW_DATA + ISO9001 템플릿이 있는 AUDIT_SESSION
- When: `generateReport` Server Action 호출
- Then: 필수 필드 100% 매핑, AUDIT_REPORT DB 저장, 스트리밍 전달. 실패율 < 0.5%

**Scenario 2: 타임아웃 우회**
- Given: 500건+ 대량 데이터
- When: Vercel AI SDK `streamText()` 호출
- Then: 60초 타임아웃 없이 스트리밍 연결 유지, 점진적 결과 전달

**Scenario 3: 데이터 부족**
- Given: RAW_DATA 50건 미만
- When: 리포트 생성 요청
- Then: 400 + `INSUFFICIENT_DATA` 에러 코드 반환

**Scenario 4: Gemini API 장애**
- Given: Gemini API 5xx 또는 무응답
- When: `streamText()` 실패
- Then: 지수 백오프 3회 재시도 → 서킷 브레이커 → 503 반환

## :gear: Technical & Non-Functional Constraints
- 성능: Vercel Hobby Timeout ≤ 60s — `streamText` 스트리밍 필수 (REQ-NF-001)
- 정확도: 필수 필드 누락률 = 0%, 생성 실패율 < 0.5% (REQ-NF-012)
- 보안: OpenAPI 3.0 Input Validation 필수 (REQ-NF-SEC-API-004)
- AI 제어: temperature=0 결정론적 출력 (REQ-NF-029 VMP)
- 비용: Gemini Free Tier (15 RPM, 1,500 RPD) 한도 내 (REQ-NF-018)
- 무결성: SHA-256 + 타임스탬프 필수 (REQ-FUNC-024)
- 프레임워크: Next.js App Router Server Action (C-TEC-001/002/005/006)

## :checkered_flag: Definition of Done (DoD)
- [ ] 모든 Acceptance Criteria 충족
- [ ] Golden Dataset 기반 매핑 정확도 ≥ 99% 테스트 통과
- [ ] 단위/통합 테스트 추가 및 통과
- [ ] ESLint / 정적 분석 경고 0건
- [ ] Vercel Hobby 환경 스트리밍 동작 확인
- [ ] API 명세서(OpenAPI) 최신화

## :construction: Dependencies & Blockers
- **Depends on:** DB-003, DB-005, DB-006, SA-Q-001, API-002, AUTH-C-001, INT-C-001, INFRA-001
- **Blocks:** SA-C-003, UI-003, TEST-SA-001, TEST-SA-002, INFRA-002
