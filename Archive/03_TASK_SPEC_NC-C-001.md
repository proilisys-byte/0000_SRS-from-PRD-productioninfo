---
name: Feature Task
about: SRS 기반의 구체적인 개발 태스크 명세
title: "[Feature] NC-C-001: NC 케이스 등록 + AI 사유 파싱 + 시정 초안 자동 생성"
labels: 'feature, backend, ai, nc, priority:critical, sprint:2'
assignees: ''
---

## ⚠️ DEPRECATED — 이 태스크는 3개로 분리되었습니다 (ISSUE-02)

> **이 파일은 더 이상 사용하지 마세요.** 아래 3개 SPEC으로 대체되었습니다:
> - [`07_TASK_SPEC_NC-C-001a.md`](./07_TASK_SPEC_NC-C-001a.md) — NC 케이스 등록 (순수 CRUD)
> - [`07_TASK_SPEC_NC-C-001b.md`](./07_TASK_SPEC_NC-C-001b.md) — AI 사유 파싱 + 시정 초안 생성
> - [`07_TASK_SPEC_NC-C-001c.md`](./07_TASK_SPEC_NC-C-001c.md) — 에스컬레이션 이벤트 + AUDIT_LOG

---

## 🎯 Summary (원본 — 참고용)
- 기능명: [NC-C-001] NC 케이스 등록 + AI 사유 파싱 + 시정 초안 자동 생성 (POST /api/v1/nc/cases)
- 목적: 원청으로부터 통보된 부적합(NC) 사유 텍스트를 접수하고, Gemini AI를 활용하여 사유를 자동 파싱한 후, 유사 NC 이력을 참조하여 시정 조치 계획서 초안을 5분 이내에 자동 생성한다. 이를 통해 품질팀장(박품질)의 긴급 NC 대응 소요 시간을 획기적으로 단축한다.

## 🔗 References (Spec & Context)
> 💡 AI Agent & Dev Note: 작업 시작 전 아래 문서를 반드시 먼저 Read/Evaluate 할 것.
- SRS 문서: [`05_SRS_v1.md#REQ-FUNC-006`] — NC 통보 사유 파싱 엔진 (커버리지 ≥ 95%)
- SRS 문서: [`05_SRS_v1.md#REQ-FUNC-010`] — 유사 NC 사례 검색 (동일 nc_code 최대 5건)
- SRS 문서: [`05_SRS_v1.md#§4.3.2`] — 긴급 NC 시정 패키지 시퀀스 다이어그램
- SRS 문서: [`05_SRS_v1.md#§4.1.2`] — NC 시정 패키지 기능 요구사항 테이블
- SRS 문서: [`05_SRS_v1.md#§3.5.2`] — NC 통보 → 긴급 시정 대응 핵심 흐름
- 데이터 모델: [`05_SRS_v1.md#§3.4.3`] — ERD (NC_CASE, CORRECTIVE_ACTION 엔티티)
- API 명세: API-004 (NC Response API DTO)
- 관련 태스크: NC-Q-002 (유사 NC 사례 검색 — 이 태스크의 선행 의존)
- 태스크 목록: [`TASKS/06_TASK_LIST_v1.md#Epic6`] — 긴급 NC 시정 패키지 태스크 목록

## ✅ Task Breakdown (실행 계획)
- [ ] NC 케이스 등록 Server Action 구현 (`createNCCase.ts`)
  - Zod 입력 검증: nc_code, severity(Critical/Major/Minor), description, site_id
  - NC_CASE 레코드 생성 (status: "OPEN", created_at 자동 부여)
- [ ] AI 사유 파싱 엔진 구현
  - Gemini 1.5 Flash API 연동 (`@ai-sdk/google`, `generateText()`)
  - 사유 텍스트 → 구조화된 부적합 항목 분해 (nc_category, affected_items, root_cause_hint)
  - 프롬프트 엔지니어링: ISO 9001/14001/45001 NC 분류 체계 기반 파싱 룰셋
  - 파싱 결과 Zod 검증 (환각 필터링)
- [ ] 유사 NC 이력 조회 연동 (NC-Q-002 호출)
  - 동일 nc_code 기반 과거 사례 최대 5건 조회
  - 유사 사례의 시정 조치 성공률 포함
- [ ] 시정 초안 자동 생성
  - Gemini AI 기반 시정 계획서 초안 생성 (유사 사례 참조 포함)
  - CORRECTIVE_ACTION 레코드 자동 생성 (status: "DRAFT", progress: 0)
  - 시정 항목별 우선순위 자동 부여
- [ ] Critical 심각도 감지 시 에스컬레이션 트리거 준비
  - severity === "Critical" 시 NC-C-004 에스컬레이션 이벤트 발행 (후속 태스크에서 소비)
- [ ] AUDIT_LOG 기록 — NC 케이스 생성 이벤트 (IntegrityManager 활용)
- [ ] 에러 처리: AI 파싱 실패 시 수동 입력 Fallback + 빈 초안 반환
- [ ] 단위 테스트 + 통합 테스트 (NC 등록 → 파싱 → 유사 검색 → 초안 생성 E2E)

## 🧪 Acceptance Criteria (BDD/GWT)

**Scenario 1: 정상 NC 등록 및 시정 초안 생성**
- Given: severity="Major", nc_code="NC-MAT-001", description="원자재 입고 검사 미실시로 인한 불량 자재 투입"이 주어짐
- When: `POST /api/v1/nc/cases` Server Action을 호출함
- Then: NC_CASE 레코드가 생성되고, AI가 파싱한 부적합 항목과 시정 계획 초안(CORRECTIVE_ACTION 1건 이상)이 5분 이내에 반환된다. 파싱 커버리지 ≥ 95%.

**Scenario 2: 유사 NC 사례 참조**
- Given: 과거에 동일 nc_code "NC-MAT-001"로 등록된 NC 사례가 3건 존재함
- When: NC 케이스 등록 시 유사 사례 검색이 수행됨
- Then: 최대 5건의 유사 사례가 반환되고, 각 사례의 시정 조치 내용과 성공률이 초안 참고 데이터로 포함된다.

**Scenario 3: Critical 심각도 에스컬레이션 트리거**
- Given: severity="Critical"인 NC 케이스가 등록됨
- When: NC_CASE 생성이 완료됨
- Then: 에스컬레이션 이벤트가 발행되어 NC-C-004에서 24시간 카운트다운이 시작된다.

**Scenario 4: AI 파싱 실패 시 Fallback**
- Given: Gemini API가 5xx 에러를 반환하거나 타임아웃이 발생함
- When: AI 사유 파싱이 실패함
- Then: NC_CASE는 정상 생성되되, 시정 초안은 빈 템플릿으로 반환되며, 사용자에게 "수동 작성이 필요합니다" 안내 메시지가 포함된다. 에러가 로깅된다.

**Scenario 5: 필수 필드 누락**
- Given: nc_code가 누락된 요청이 수신됨
- When: Zod 검증이 실행됨
- Then: 400 Bad Request와 `code: "VALIDATION_001"` 에러가 반환되며, NC_CASE가 생성되지 않는다.

## ⚙️ Technical & Non-Functional Constraints
- 성능: 초안 생성 응답시간 ≤ 5분 (REQ-FUNC-006 AC). Gemini Flash 모델 사용으로 속도 최적화
- AI 정확도: 사유 파싱 커버리지 ≥ 95% (Appendix A.4 기준). temperature=0 결정론적 출력
- 비용: Gemini Free Tier (15 RPM, 1,500 RPD) 한도 내 (REQ-NF-018)
- 보안: NC 사유 텍스트 로깅 시 민감정보 마스킹 (REQ-NF-SEC-LOG-003)
- 무결성: NC 등록 이벤트 AUDIT_LOG 기록 (REQ-FUNC-024)
- AI 제어: Gemini 출력 Zod 검증 필수, 환각 필터링 (REQ-NF-029)
- 프레임워크: Next.js Server Action + Vercel AI SDK (C-TEC-001, C-TEC-005, C-TEC-006)

## 🏁 Definition of Done (DoD)
- [ ] 모든 Acceptance Criteria를 충족하는가?
- [ ] NC_CASE + CORRECTIVE_ACTION 레코드가 정상 생성되는가?
- [ ] AI 사유 파싱 커버리지 ≥ 95% 테스트를 통과하는가?
- [ ] 유사 NC 사례 검색이 최대 5건 반환되는가?
- [ ] AI 실패 시 Fallback이 정상 동작하는가?
- [ ] AUDIT_LOG에 NC 등록 이벤트가 기록되는가?
- [ ] 단위 테스트 및 통합 테스트가 추가되었고 통과하는가?
- [ ] ESLint / 정적 분석 경고가 없는가?
- [ ] API 명세서(OpenAPI)가 최신화되었는가?

## 🚧 Dependencies & Blockers
- **Depends on:** DB-007 (NC_CASE 테이블), DB-008 (CORRECTIVE_ACTION 테이블), NC-Q-002 (유사 NC 사례 검색), API-004 (NC Response DTO), AUTH-C-001 (인증 기반), INT-C-001 (IntegrityManager — AUDIT_LOG 기록)
- **Blocks:** NC-C-002 (시정 진행률 갱신), NC-C-003 (시정 전후 비교 보고서), NC-C-004 (Critical 에스컬레이션), UI-006 (NC 케이스 등록 화면), TEST-NC-001 (파싱 커버리지 테스트)
