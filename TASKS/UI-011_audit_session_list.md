---
name: Feature Task
about: PRD&SRS 기반의 구체적인 개발 태스크 명세
title: "[UI] UI-011: Audit Session List Page"
labels: 'ui, smart-audit, spec, priority:medium'
assignees: ''
---

## 🎯 Summary
- 기능명: [UI-011] Audit Session List Page
- 목적: Smart Audit F1 라인의 세션들을 한눈에 조회, 검색, 필터링하여 운영자 및 심사자가 원하는 작업 공간(`UI-010`)으로 쉽게 진입할 수 있도록 돕는다.

## 🔗 References (Spec & Context)
> 작업 시작 전 아래 문서를 반드시 먼저 읽고 정합성을 유지할 것.
- Query API 명세: `F1-Q-001_audit_report_query.md`
- 상세 화면 명세: `UI-010_audit_workspace_page.md`
- Mock 데이터: `MOCK-003_audit_list_mock_endpoint.md`
- **Pagination 정본**: `F1-C-001_audit_session_management.md` Appendix A-4 (Cursor 기반)

## 🧭 Scope
### In Scope
- 세션 목록 데이터 표출 (Table 또는 List UI)
- 검색 기능 (세션명, 작성자 등)
- 필터링 기능 (상태별: Draft, Submitted 등, 날짜별)
- **Cursor 기반 페이지네이션** 로직 연동 (`F1-C-001` Appendix A-4 준수)
- 개별 항목 클릭 시 Workspace 진입 연동

### Out of Scope
- 세션 일괄(Bulk) 상태 변경 (Sprint 1 대상 아님)
- 리포트 직접 다운로드 (목록이 아닌 상세화면 기능)

## 🧱 Preconditions
- 선행 완료 문서:
  - `F1-Q-001_audit_report_query.md`
  - `UI-010_audit_workspace_page.md`
- 시작 가능 조건:
  - 리스트를 그리기 위한 백엔드 조회 쿼리 파라미터가 정의되어 있음

## ✅ Task Breakdown (실행 계획)
- [ ] 데이터 테이블 레이아웃 및 컬럼 구성 (ID, 제목, 상태, 작성자, 수정일)
- [ ] 필터 컴포넌트(Select Box, Date Picker 등) 구현
- [ ] **Cursor 기반** Pagination UI 및 데이터 패칭 훅 작성 (Appendix A-4 Zod 스키마 적용)
- [ ] 상태별(Badge) 색상 및 아이콘 UI 적용
- [ ] API 연동 및 빈 상태(Empty State) 예외 처리
- [ ] **대량 데이터(10,000건+) 환경 가상 스크롤 또는 점진적 로딩 적용**

## 🧪 Acceptance Criteria (BDD / GWT)
Scenario 1: 리스트 정상 조회
- Given: 테넌트 내에 접근 가능한 세션 데이터가 여러 개 존재한다.
- When: Session List 화면에 진입한다.
- Then: 첫 페이지의 세션 목록이 최신순으로 표시되며 올바른 상태 뱃지가 노출된다.

Scenario 2: 조건 필터링 동작
- Given: 'Draft' 상태의 세션만 보고 싶다.
- When: 상태 필터에서 'Draft'를 선택한다.
- Then: 리스트가 리프레시되어 Draft 상태인 세션만 표시된다.

Scenario 3: 권한 기반 데이터 제한
- Given: A 테넌트의 사용자가 화면에 접근했다.
- When: 전체 목록 조회를 요청한다.
- Then: B 테넌트의 세션은 절대 리스트에 나타나지 않는다 (API 레벨 필터링을 UI에서 정상 표출).

Scenario 4: 성능 기준 충족
- Given: 테넌트 내에 1,000건 이상의 세션 데이터가 존재한다.
- When: Session List 화면에 최초 진입한다.
- Then: LCP(Largest Contentful Paint) **≤ 2.5초** 이내에 첫 페이지 목록이 렌더링된다.

Scenario 5: 빈 결과 처리
- Given: 테넌트 내에 세션 데이터가 0건이거나 필터 조건에 맞는 결과가 없다.
- When: 조회를 시도한다.
- Then: "등록된 세션이 없습니다. 새 세션을 시작해보세요." Empty State UI가 표시된다.

## 🔐 Technical / Domain Constraints
- 권한/RBAC 규칙:
  - Tenant Admin은 모든 세션, 일반 User는 본인 관련 세션만 보이게 될 수 있음(API 응답 결과에 종속)
- 공통 에러 스키마 연계:
  - 리스트 조회 실패(500) 시 에러 바운더리 또는 Fallback UI 제공
- 성능 / 운영 제약:
  - 렌더링 최적화를 위해 불필요한 리렌더링 억제 및 디바운스(검색창, 300ms) 적용
  - **LCP 목표: ≤ 2.5초** (1,000건 기준)
  - **데이터 볼륨 상한: 테넌트당 최대 10,000건** 지원 (이후 아카이빙 정책 적용)
  - **Pagination**: Cursor 기반 (Offset 방지) — `F1-C-001` Appendix A-4 SSOT 준수

## 📦 Deliverables
- 산출 문서/코드/테스트/Mock/연계 결과:
  - `AuditSessionList.tsx` 컴포넌트 코드
  - 리스트 필터링/페이징 기능 테스트

## 🏁 Definition of Done (DoD)
- [ ] PRD/SRS 및 관련 기존 TASK 문서와 충돌하지 않는가?
- [ ] Acceptance Criteria를 모두 반영했는가?
- [ ] 상태/권한/tenant-site/에러/감사로그 기준이 누락되지 않았는가?
- [ ] 관련 후속 문서 또는 구현 단계로 바로 넘길 수 있는가?
- [ ] 범위가 다른 카테고리 문서와 섞이지 않았는가?

## 🚧 Dependencies & Blockers
- Depends on:
  - `F1-Q-001_audit_report_query.md`
  - `UI-000_app_shell_layout.md`
- Blocks:
  - 없음

## 📝 Notes for Dev / AI Agent
- 관련 문서와의 경계:
  - 이 화면은 철저히 조회용이다. 세션의 생성 버튼은 존재할 수 있으나 생성 동작 이후 상세 화면(`UI-010`)으로 처리를 위임한다.
