---
name: Feature Task
about: PRD&SRS 기반의 구체적인 개발 태스크 명세
title: "[MOCK] MOCK-003: Audit List Mock Endpoint"
labels: 'mock, api, spec, priority:medium'
assignees: ''
---

## 🎯 Summary
- 기능명: [MOCK-003] Audit List Mock Endpoint
- 목적: 프론트엔드 개발자 및 QA가 `UI-011` (Audit Session List) 화면을 구축하고 테스트할 수 있도록, 지연/에러/페이징이 반영된 가짜 데이터를 제공하는 Mock 서버(API)를 정의한다.

## 🔗 References (Spec & Context)
> 작업 시작 전 아래 문서를 반드시 먼저 읽고 정합성을 유지할 것.
- 관련 UI: `UI-011_audit_session_list.md`
- 데이터 계약: `API-003_audit_report_dto.md`
- 조회 로직 명세: `F1-Q-001_audit_report_query.md`

## 🧭 Scope
### In Scope
- `GET /api/mock/v1/audit-sessions` 엔드포인트 응답 정의
- 페이지네이션(Page, Size, TotalCount) 구조 포함
- 10개 이상의 다양한 상태(Draft, Submitted 등)를 지닌 Mock Array 데이터
- 에러 트리거링 로직(`X-Mock-Scenario` 헤더 지원)

### Out of Scope
- 영구 저장(DB 연동) 로직 없음 (Stateless 응답)
- 세션 생성/수정 Mock (별도 문서 처리)

## 🧱 Preconditions
- 선행 완료 문서:
  - `API-003_audit_report_dto.md`
- 시작 가능 조건:
  - 리스트 조회 DTO와 화면 요구사항이 픽스된 상태

## ✅ Task Breakdown (실행 계획)
- [ ] Mock 데이터 팩토리 작성 (UUID, 임의 텍스트, 랜덤 상태값 생성)
- [ ] Request Query Parameter (`cursor`, `limit`, `status`, `tenant_id`) 파싱 로직 작성
- [ ] 필터 조건에 따른 Array 필터링 로직 구현
- [ ] Response Payload (Items 배열 + Cursor Pagination 메타데이터) 구성
- [ ] `X-Mock-Scenario` 헤더에 따른 400, 403, 500 에러 및 타임아웃 반환 분기 작성
- [ ] **Empty(0건), Large(100건+) 시나리오 테스트 케이스 추가**

### Mock 데이터 팩토리 필드 명세

| 필드 | 타입 | 생성 규칙 | 예시 |
|:---|:---|:---|:---|
| `id` | UUID v4 | `crypto.randomUUID()` | `"a1b2c3d4-..."` |
| `title` | string | `"Audit Session #{index}"` | `"Audit Session #1"` |
| `status` | enum | `Draft`(40%), `In_Progress`(30%), `Submitted`(20%), `Finalized`(10%) | `"Draft"` |
| `created_by` | string | 테스트 사용자명 3명 랜덤 | `"김철수"` |
| `created_at` | ISO 8601 | 최근 30일 내 랜덤 | `"2026-04-20T09:30:00Z"` |
| `updated_at` | ISO 8601 | `created_at` 이후 랜덤 | `"2026-04-22T14:15:00Z"` |
| `tenant_id` | UUID v4 | 고정 2개 테넌트 중 랜덤 | `"t-001-..."` |
| `entry_count` | number | 0~100 랜덤 | `42` |

### 응답 JSON 페이로드 구조 (Cursor Pagination)
```json
{
  "items": [
    {
      "id": "a1b2c3d4-e5f6-7890-abcd-ef1234567890",
      "title": "Audit Session #1",
      "status": "Draft",
      "created_by": "김철수",
      "created_at": "2026-04-20T09:30:00Z",
      "updated_at": "2026-04-22T14:15:00Z",
      "tenant_id": "t-001-xxxx",
      "entry_count": 42
    }
  ],
  "pagination": {
    "next_cursor": "eyJpZCI6ImExYjJjM2Q0Li4uIn0=",
    "has_next": true,
    "total_count": 156,
    "limit": 20
  }
}
```

## 🧪 Acceptance Criteria (BDD / GWT)
Scenario 1: 정상 리스트 반환
- Given: 정상적인 조회 요청을 보냈다.
- When: 엔드포인트를 호출한다.
- Then: 상태 코드 200과 함께 정의된 개수(`limit`)만큼의 아이템이 배열 형태로 반환된다.

Scenario 2: 상태 필터 동작 (Mock 레벨)
- Given: `?status=Draft` 쿼 파라미터를 추가하여
- When: 엔드포인트를 호출한다.
- Then: 반환된 아이템들의 `status` 속성이 모두 `Draft`이다.

Scenario 3: 타임아웃 테스트 (에러 트리거)
- Given: `X-Mock-Scenario: TIMEOUT` 헤더를 포함하여
- When: 엔드포인트를 호출한다.
- Then: 응답이 3초간 지연된 후, 504 Gateway Timeout (또는 임의의 지연 효과) 오류가 발생한다.

Scenario 4: 빈 결과 반환 (Edge Case)
- Given: `X-Mock-Scenario: EMPTY` 헤더를 포함하여
- When: 엔드포인트를 호출한다.
- Then: `items` 배열이 빈 배열(`[]`)이고 `total_count: 0`, `has_next: false`로 반환된다.

Scenario 5: 대량 결과 반환 (Edge Case)
- Given: `X-Mock-Scenario: LARGE` 헤더를 포함하고 `limit=100`으로 요청
- When: 엔드포인트를 호출한다.
- Then: 100건의 아이템이 반환되고 `has_next: true`, 유효한 `next_cursor`가 포함된다.

## 🔐 Technical / Domain Constraints
- 공통 에러 스키마 연계:
  - 발생시키는 가짜 에러 또한 `API-001`의 스키마 구조를 완벽히 준수해야 함
- 성능 / 운영 제약:
  - 로컬 MSW(Mock Service Worker) 또는 개발 서버에서만 활성화되도록 환경변수 제어

## 📦 Deliverables
- 산출 문서/코드/테스트/Mock/연계 결과:
  - Mock API Handler 코드 (예: MSW handlers.ts 항목 추가)
  - 연동 가이드라인

## 🏁 Definition of Done (DoD)
- [ ] PRD/SRS 및 관련 기존 TASK 문서와 충돌하지 않는가?
- [ ] Acceptance Criteria를 모두 반영했는가?
- [ ] 상태/권한/tenant-site/에러/감사로그 기준이 누락되지 않았는가?
- [ ] 관련 후속 문서 또는 구현 단계로 바로 넘길 수 있는가?
- [ ] 범위가 다른 카테고리 문서와 섞이지 않았는가?

## 🚧 Dependencies & Blockers
- Depends on:
  - `API-003_audit_report_dto.md`
- Blocks:
  - `UI-011_audit_session_list.md` 구현 완료

## 📝 Notes for Dev / AI Agent
- 이 TASK의 핵심 초점:
  - 프론트엔드가 백엔드 개발 완료를 기다리지 않고 바로 Pagination과 Error UX를 개발할 수 있도록 돕는 데 있다.
