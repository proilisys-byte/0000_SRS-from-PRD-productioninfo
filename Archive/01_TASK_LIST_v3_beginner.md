# PRO ILI SMART — 개발 태스크 목록 v3 (초보 설계자용)

| 항목 | 내용 |
|:---|:---|
| **기준 문서** | SRS_v1.md, TASK_LIST_v2.md |
| **작성일** | 2026-04-23 |
| **개정 이력** | v1 → v2(시니어용) → **v3(초보 설계자용)** |
| **목적** | 3개월차 설계자가 **혼자서 끝까지 구현 가능한** 태스크 구조 |

---

## 🚨 v3 핵심 변경 (v2 대비)

| 원칙 | v2 (시니어) | v3 (초보자) |
|---|---|---|
| RULE-01 | Sprint당 개념 무제한 | **Sprint당 핵심 개념 최대 2개** |
| RULE-02 | AI가 Core Flow 내장 | **AI는 Optional Layer (Phase 4)** |
| RULE-03 | Queue/DLQ/비동기 | **모든 처리 동기(Synchronous)** |
| RULE-04 | Merkle/SHA-256 체인 | **단순 로그 테이블 (created_at + log)** |
| RULE-05 | Fallback/Circuit Breaker | **실패 = 에러 반환 (단순화)** |

### 위험 태스크 다운그레이드 결과

| 원래 태스크 | v2 방식 | v3 방식 |
|---|---|---|
| SA-C-002b (AI 매핑) | LLM + Output Contract | **하드코딩 규칙 기반 매핑** |
| ZUI-C-003c (AI 구조화) | 비동기 LLM 처리 | **JSON 그대로 저장** |
| INT-C-001 (Integrity) | SHA-256 + Merkle | **created_at + log만 유지** |
| AI-C-001~008 | AI Governance 전체 | **⛔ 전체 제거 → Phase 5 이후** |
| RES-001~003 | DLQ + 지수 백오프 | **Retry 1회, DLQ 제거** |

### 디버깅 필수 필드 (모든 API에 적용)

모든 Command/Mutation 요청에 아래 4개 필드를 **반드시** 포함:

```typescript
interface DebugContext {
  request_id: string;    // UUID — 요청 추적용
  step_name: string;     // 예: "create_nc_case"
  status: 'start' | 'success' | 'fail';
  error_message?: string; // 실패 시 원인
}
```

### Hidden Coupling 방지 규칙

- ❌ AI는 DB 직접 접근 금지
- ✅ API → Service → DB 단방향만 허용
- ❌ 이벤트 기반 구조 금지 (Phase 4까지)

### 성공 기준

- [ ] 한 명이 하루 안에 전체 흐름 설명 가능
- [ ] 에러 발생 시 5분 내 원인 파악 가능
- [ ] Postman으로 전체 기능 테스트 가능

---

## Phase 1: 무조건 성공 단계 (Sprint 0 + Sprint 1, 3주)

> 🎯 **목표**: "데이터 생성 → 조회"만 되면 성공
> 📌 **개념**: DB CRUD + API CRUD + UI 최소 기능

### Epic 1: DB 스키마 (핵심 테이블만)

> [!TIP]
> Prisma 마이그레이션 1개씩 실행 → `npx prisma db push` → 성공 확인 → 다음 진행

| # | Task ID | 기능 | 복잡도 | 구현 시간 | 검증 질문 |
|---|---|---|---|---|---|
| 1 | DB-001 | SITE 테이블 (id, name, address, created_at) | L | 1h | 없으면 동작 안 함 ✅ |
| 2 | DB-002 | USER 테이블 (id, email, role, site_id FK) | L | 1h | 없으면 동작 안 함 ✅ |
| 3 | DB-003 | RAW_DATA 테이블 (id, source_type, payload JSONB, created_at) | L | 1h | 없으면 동작 안 함 ✅ |
| 4 | DB-004 | TEMPLATE 테이블 (id, name, schema JSONB, version) | L | 1h | 없으면 동작 안 함 ✅ |
| 5 | DB-005 | AUDIT_SESSION 테이블 (id, status, template_id FK, site_id FK) | L | 1h | 없으면 동작 안 함 ✅ |
| 6 | DB-006 | AUDIT_REPORT 테이블 (id, session_id FK, content JSONB, version) | L | 1h | 없으면 동작 안 함 ✅ |
| 7 | DB-007 | NC_CASE 테이블 (id, severity, status, description) | L | 1h | 없으면 동작 안 함 ✅ |
| 8 | DB-008 | CORRECTIVE_ACTION 테이블 (id, nc_id FK, progress, action_text) | L | 1h | 없으면 동작 안 함 ✅ |
| 9 | DB-009 | LEAN_DIAGNOSIS 테이블 (id, site_id FK, waste_data JSONB) | L | 1h | 없으면 동작 안 함 ✅ |
| 10 | DB-010s | **AUDIT_LOG 테이블 (단순화)** — id, action, entity, entity_id, created_at, user_id | L | 1h | 로그 기록용 ✅ |
| 11 | DB-011 | SUBMISSION_LOG 테이블 (id, entity, payload, created_at) | L | 1h | 로그 기록용 ✅ |

> [!IMPORTANT]
> **v2의 DB-010(Merkle 해시 체인) → v3의 DB-010s(단순 로그)로 다운그레이드**
> SHA-256, hash_chain, Merkle 필드 전부 제거. `created_at` + `action` + `entity`만 유지.

> [!WARNING]
> v2의 DB-012~017 (AI 레지스트리, Golden Dataset, JSON 호환 미들웨어, Insert-only Policy)은 **Phase 4 이후로 연기**.

**Phase 1 DB 체크포인트:**
```
✅ Prisma Studio에서 11개 테이블 모두 보임
✅ 각 테이블에 seed 데이터 1건 이상 Insert 성공
✅ FK 관계 정상 동작 확인
```

---

### Epic 2: API DTO + 에러 코드 (최소한만)

| # | Task ID | 기능 | 복잡도 | 구현 시간 | 검증 질문 |
|---|---|---|---|---|---|
| 1 | API-007s | **공통 에러 응답 형식** (status, message, request_id) | L | 2h | Postman에서 에러 형식 통일 ✅ |
| 2 | API-R01 | Audit Report CRUD DTO (create/read만) | L | 2h | 리포트 생성·조회 ✅ |
| 3 | API-R02 | NC Case CRUD DTO (create/read/update) | L | 2h | NC 등록·조회·수정 ✅ |
| 4 | API-R03 | Template CRUD DTO (read만) | L | 1h | 템플릿 조회 ✅ |
| 5 | API-R04 | Lean Diagnosis DTO (create/read) | L | 2h | 진단 실행·조회 ✅ |
| 6 | API-R05 | Edge Sync DTO (create/read) | L | 2h | 데이터 수집·조회 ✅ |

> [!NOTE]
> v2의 API-001~006 (도메인별 분리 DTO)을 **단일 공통 형식**으로 통합.
> v2의 API-008 (OpenAPI Validation)은 Phase 2로 연기.

**공통 응답 형식:**
```typescript
// 성공
{ status: 'success', data: {...}, request_id: 'uuid' }

// 실패
{ status: 'error', message: '설명', error_code: 'NC_NOT_FOUND', request_id: 'uuid' }
```

---

### Epic 3: Mock 데이터

| # | Task ID | 기능 | 복잡도 | 구현 시간 |
|---|---|---|---|---|
| 1 | MOCK-001 | SITE 2개 + USER 4명 Seed 데이터 | L | 1h |
| 2 | MOCK-002 | TEMPLATE 3종 (ISO 9001/14001/45001) Seed | L | 1h |
| 3 | MOCK-003 | RAW_DATA 샘플 10건 | L | 1h |
| 4 | MOCK-004 | NC_CASE 3건 + CORRECTIVE_ACTION 3건 | L | 1h |
| 5 | MOCK-005 | LEAN_DIAGNOSIS 샘플 3건 | L | 30m |

> [!WARNING]
> v2의 MOCK-006 (MSW Mocking API)은 **Phase 2로 연기**. 지금은 실제 API를 직접 호출.

---

### Epic 4: API 라우트 (CRUD만)

> 🎯 모든 API는 `API → Service → DB` 단방향. 비즈니스 로직 없이 순수 CRUD만.

| # | Task ID | 기능 | HTTP | 복잡도 | 구현 시간 |
|---|---|---|---|---|---|
| 1 | CRUD-001 | 템플릿 목록 조회 | GET /api/v1/templates | L | 1h |
| 2 | CRUD-002 | 템플릿 상세 조회 | GET /api/v1/templates/:id | L | 1h |
| 3 | CRUD-003 | 감사 세션 생성 | POST /api/v1/audit/sessions | L | 2h |
| 4 | CRUD-004 | 감사 리포트 생성 | POST /api/v1/audit/reports | L | 2h |
| 5 | CRUD-005 | 감사 리포트 조회 | GET /api/v1/audit/reports/:id | L | 1h |
| 6 | CRUD-006 | NC 케이스 등록 | POST /api/v1/nc/cases | L | 2h |
| 7 | CRUD-007 | NC 케이스 조회 | GET /api/v1/nc/cases/:id | L | 1h |
| 8 | CRUD-008 | 시정 조치 등록 | POST /api/v1/nc/actions | L | 2h |
| 9 | CRUD-009 | 시정 조치 상태 수정 | PATCH /api/v1/nc/actions/:id | L | 1h |
| 10 | CRUD-010 | Raw 데이터 저장 | POST /api/v1/edge/sync | L | 2h |
| 11 | CRUD-011 | Raw 데이터 조회 | GET /api/v1/edge/data | L | 1h |
| 12 | CRUD-012 | Lean 진단 실행 | POST /api/v1/lean/diagnose | M | 3h |
| 13 | CRUD-013 | Lean 진단 결과 조회 | GET /api/v1/lean/diagnose/:id | L | 1h |

**Phase 1 API 체크포인트:**
```
✅ Postman Collection으로 13개 엔드포인트 전부 호출 성공
✅ 정상 요청 → 200 + data 반환
✅ 잘못된 요청 → 400 + error message 반환
✅ 없는 ID 조회 → 404 반환
```

---

### Epic 5: UI 최소 기능

| # | Task ID | 기능 | 복잡도 | 구현 시간 |
|---|---|---|---|---|
| 1 | UI-001s | 로그인 화면 (이메일+비밀번호, Supabase Auth 연동) | M | 4h |
| 2 | UI-002s | 대시보드 홈 (메뉴 네비게이션만) | L | 2h |
| 3 | UI-003s | 템플릿 목록 페이지 | L | 2h |
| 4 | UI-004s | NC 케이스 목록 + 등록 폼 | M | 4h |
| 5 | UI-005s | Raw 데이터 목록 페이지 | L | 2h |

> [!NOTE]
> v2의 UI-001~014 (14개 대시보드 화면)에서 **핵심 5개만** 추출. 나머지는 Phase 3~4.

**Phase 1 완료 기준:**
```
✅ 로그인 → 대시보드 → 목록 조회 → 데이터 생성 흐름 동작
✅ 전체 흐름을 Postman + 브라우저로 10분 내 시연 가능
✅ 에러 발생 시 console.log로 request_id 추적 가능
```

---

## Phase 2: 정상 흐름 안정화 (Sprint 2, 2주)

> 🎯 **목표**: 인증 + 입력 검증 + 로그 기록
> 📌 **개념**: Auth 완성 + Validation

### Epic 6: 인증·인가

| # | Task ID | 기능 | 복잡도 | 구현 시간 | 검증 질문 |
|---|---|---|---|---|---|
| 1 | AUTH-001s | Supabase Auth 회원가입·로그인 구현 | M | 4h | 없으면 동작 안 함 ✅ |
| 2 | AUTH-002s | 역할 구분 (Admin/User) 미들웨어 | M | 3h | 디버깅 가능 ✅ |
| 3 | AUTH-003s | 로그인 실패 5회 → 잠금 (단순 카운터) | L | 2h | 하루 구현 가능 ✅ |
| 4 | AUTH-004s | 세션 30분 유휴 → 자동 로그아웃 | L | 2h | 하루 구현 가능 ✅ |

> [!NOTE]
> v2의 AUTH-C-002 (12자 3종 패스워드 정책)과 AUTH-C-005 (RBAC 감사로그 연동)는 **Phase 3로 연기**.
> 지금은 "로그인 되면 성공, Admin/User 구분만 동작"이 목표.

### Epic 7: 기본 Validation

| # | Task ID | 기능 | 복잡도 | 구현 시간 |
|---|---|---|---|---|
| 1 | VAL-001 | Zod 스키마 기반 입력 검증 (공통 미들웨어) | M | 3h |
| 2 | VAL-002 | NC 등록 필수값 검증 (severity, description 필수) | L | 1h |
| 3 | VAL-003 | Audit Report 생성 필수값 검증 | L | 1h |
| 4 | VAL-004 | Edge Sync 데이터 형식 검증 (source_type enum 체크) | L | 1h |

### Epic 8: 단순 로그 시스템

| # | Task ID | 기능 | 복잡도 | 구현 시간 |
|---|---|---|---|---|
| 1 | LOG-001 | 공통 로그 함수 — AUDIT_LOG 테이블에 Insert | L | 2h |
| 2 | LOG-002 | 모든 CRUD API에 로그 호출 추가 | L | 3h |
| 3 | LOG-003 | 로그 조회 API (GET /api/v1/logs, Admin 전용) | L | 2h |

> [!IMPORTANT]
> **v2의 INT-C-001(SHA-256 해시체인) → LOG-001(단순 Insert)**
> `{ action: 'CREATE', entity: 'NC_CASE', entity_id: '123', user_id: 'abc', created_at: now() }`
> 이것만으로 "누가 언제 뭘 했는지" 충분히 추적 가능.

**Phase 2 완료 기준:**
```
✅ 비로그인 상태에서 API 호출 → 401 반환
✅ User 역할로 Admin 전용 API 호출 → 403 반환
✅ 필수값 누락 요청 → 400 + 명확한 에러 메시지
✅ 모든 CRUD 동작이 AUDIT_LOG에 기록됨
✅ GET /api/v1/logs로 전체 로그 조회 가능
```

---

## Phase 3: 확장 — 비즈니스 로직 (Sprint 3, 2주)

> 🎯 **목표**: 핵심 비즈니스 기능 추가
> 📌 **개념**: 규칙 기반 매핑 + 분석 기능

### Epic 9: Smart Audit (규칙 기반)

| # | Task ID | 기능 | 복잡도 | 구현 시간 |
|---|---|---|---|---|
| 1 | SA-001s | 입력 데이터 정규화 (rule-based, trim/대소문자 통일) | M | 3h |
| 2 | SA-002s | **하드코딩 규칙 기반 매핑** (ISO 항목 → 보고서 필드) | M | 4h |
| 3 | SA-003s | 결과 검증 (JSON Schema 검증, 필수 필드 체크) | M | 3h |
| 4 | SA-004s | 리포트 버전 이력 저장 (version 컬럼 increment) | L | 2h |

> [!IMPORTANT]
> **v2의 SA-C-002b(LLM AI 매핑) → SA-002s(하드코딩 if-else 매핑)**
> ```typescript
> // v3: 단순 규칙 매핑 예시
> function mapToReport(rawData: RawData): ReportField {
>   switch(rawData.source_type) {
>     case 'inspection': return { section: '4.2', field: 'inspection_result' };
>     case 'measurement': return { section: '7.1', field: 'measurement_value' };
>     default: return { section: 'unknown', field: 'manual_review' };
>   }
> }
> ```

### Epic 10: NC 시정 (핵심만)

| # | Task ID | 기능 | 복잡도 | 구현 시간 |
|---|---|---|---|---|
| 1 | NC-001s | NC 등록 시 severity에 따른 기한 자동 설정 | L | 2h |
| 2 | NC-002s | 시정 조치 진행률 계산 (완료/전체 × 100) | L | 2h |
| 3 | NC-003s | 기한 초과 NC 목록 조회 API | L | 2h |

> [!NOTE]
> v2의 NC-C-001b(AI 사유 파싱), NC-C-001c(에스컬레이션 이벤트), NC-C-003(SHA-256 비교 보고서)는 **Phase 4로 연기**.

### Epic 11: Edge 데이터 수집 (단순화)

| # | Task ID | 기능 | 복잡도 | 구현 시간 |
|---|---|---|---|---|
| 1 | EDGE-001s | Raw 데이터 **JSON 그대로 저장** (구조화 없이) | L | 1h |
| 2 | EDGE-002s | 대용량 파일 Supabase Storage 업로드 | M | 3h |
| 3 | EDGE-003s | 네트워크 오류 시 에러 반환 (Retry 1회만) | L | 2h |

> [!IMPORTANT]
> **v2의 ZUI-C-003c(비동기 LLM 구조화) → EDGE-001s(JSON 그대로 저장)**
> Queue, DLQ, 비동기 처리 전부 제거. 받은 데이터를 그대로 DB에 넣는다.

### Epic 12: Lean 진단 (최소)

| # | Task ID | 기능 | 복잡도 | 구현 시간 |
|---|---|---|---|---|
| 1 | LEAN-001s | 데이터 유효성 검증 (7일 미만 경고) | L | 2h |
| 2 | LEAN-002s | 4대 낭비 금액 환산 (단순 수식 적용) | M | 4h |
| 3 | LEAN-003s | 결과 조회 API | L | 1h |

### Epic 13: 벌크 업로드 (기본)

| # | Task ID | 기능 | 복잡도 | 구현 시간 |
|---|---|---|---|---|
| 1 | BULK-001s | CSV 파일 업로드 + 파싱 | M | 3h |
| 2 | BULK-002s | 필수값 검증 + 에러 리포트 반환 | M | 3h |
| 3 | BULK-003s | 검증 통과 데이터 DB 일괄 Insert | M | 2h |

**Phase 3 완료 기준:**
```
✅ 규칙 기반 매핑으로 리포트 자동 생성 가능
✅ NC 등록 → 시정 조치 → 진행률 확인 흐름 동작
✅ Edge에서 데이터 전송 → DB 저장 확인
✅ Lean 진단 실행 → 결과 조회 가능
✅ CSV 업로드 → 검증 → DB 적재 가능
```

---

## Phase 4: AI 도입 — Optional Layer (Sprint 4, 2주)

> 🎯 **목표**: 기존 기능을 건드리지 않고 AI 추가
> ⚠️ **원칙**: AI가 실패해도 Phase 3까지의 기능은 100% 정상 동작

### Epic 14: AI 매핑 (Optional)

| # | Task ID | 기능 | 복잡도 | 구현 시간 |
|---|---|---|---|---|
| 1 | AI-MAP-001 | Feature Flag: AI 매핑 ON/OFF 토글 | L | 2h |
| 2 | AI-MAP-002 | Gemini API 연동 — 매핑 결과 생성 | H | 6h |
| 3 | AI-MAP-003 | AI 실패 시 → 규칙 기반(SA-002s)으로 자동 전환 | M | 3h |

### Epic 15: AI 데이터 구조화 (Optional)

| # | Task ID | 기능 | 복잡도 | 구현 시간 |
|---|---|---|---|---|
| 1 | AI-STR-001 | Feature Flag: AI 구조화 ON/OFF 토글 | L | 2h |
| 2 | AI-STR-002 | Gemini API로 Raw JSON → 구조화 데이터 변환 | H | 6h |
| 3 | AI-STR-003 | AI 실패 시 → 원본 JSON 유지 (EDGE-001s) | L | 1h |

### Epic 16: 음성/이미지 AI (Optional)

| # | Task ID | 기능 | 복잡도 | 구현 시간 |
|---|---|---|---|---|
| 1 | AI-STT-001 | 음성 STT — Gemini Multimodal 연동 | H | 6h |
| 2 | AI-VIS-001 | Vision AI — 부품 판별 연동 | H | 6h |
| 3 | AI-FALL-001 | 3회 실패 → 수동 입력 UI 전환 | M | 3h |

> [!WARNING]
> **AI-C-001~008 (AI Governance)는 Phase 5 이후로 완전 제거.**
> Model Card, Drift 감지, IAA 계수, HitL 프로세스 등은 시스템이 안정화된 후 도입.

**Phase 4 완료 기준:**
```
✅ Feature Flag OFF → Phase 3 기능 100% 정상
✅ Feature Flag ON → AI 기능 동작
✅ AI 실패 → 자동으로 규칙 기반으로 전환 (서비스 중단 없음)
✅ AI 호출 로그가 AUDIT_LOG에 기록됨
```

---

## Sprint 배치 요약

| Sprint | 기간 | Phase | 핵심 작업 | 태스크 수 |
|---|---|---|---|---|
| **Sprint 0** | 1주 | Phase 1 | DB 11개 테이블 + Seed 데이터 | 16 |
| **Sprint 1** | 2주 | Phase 1 | API 6개 + CRUD 13개 + UI 5개 | 24 |
| **Sprint 2** | 2주 | Phase 2 | Auth 4개 + Validation 4개 + Log 3개 | 11 |
| **Sprint 3** | 2주 | Phase 3 | 비즈니스 로직 (매핑/NC/Edge/Lean/Bulk) | 16 |
| **Sprint 4** | 2주 | Phase 4 | AI Optional Layer | 9 |
| **합계** | **9주** | | | **76** |

---

## Dependency Graph (단순화)

```
Phase 1: DB → API DTO → Mock → CRUD API → UI
                                    ↓
Phase 2:                     Auth → Validation → Log
                                    ↓
Phase 3:              규칙 매핑 → NC 시정 → Edge → Lean → Bulk
                                    ↓
Phase 4:              Feature Flag → AI 연동 → Fallback(규칙 기반)
```

> [!TIP]
> 모든 화살표는 **단방향**. 역방향 의존성이 발생하면 설계 오류.

---

## 태스크 통계 요약 (v3 vs v2)

| 항목 | v2 (시니어) | v3 (초보자) | 변화 |
|---|---|---|---|
| 총 태스크 수 | 182 | **76** | **-58% 감소** |
| Epic 수 | 27 | **16** | -41% 감소 |
| H 복잡도 태스크 | 30+ | **6** (Phase 4에 집중) | -80% 감소 |
| AI 관련 태스크 | 20+ | **9** (전부 Optional) | Phase 4로 격리 |
| Sprint 수 | 6 (Sprint 0A~4) | **5** (Sprint 0~4) | 간소화 |
| 예상 기간 | 10주+ | **9주** | 안정적 |

---

## v2 → v3 태스크 매핑표

> 각 v2 태스크가 v3에서 어디로 갔는지 추적용

| v2 Task ID | v2 기능 | v3 상태 | v3 Task ID / 사유 |
|---|---|---|---|
| DB-001~009 | 핵심 테이블 | ✅ 유지 | DB-001~009 |
| DB-010 | Merkle 해시 체인 로그 | ⬇️ 다운그레이드 | DB-010s (단순 로그) |
| DB-011 | SUBMISSION_LOG | ✅ 유지 | DB-011 |
| DB-012~017 | AI 레지스트리/Golden/미들웨어 | ⏩ Phase 5+ 연기 | — |
| API-001~006 | 도메인별 DTO | ⬇️ 통합 | API-R01~R05 |
| API-007 | 에러 코드 체계 | ⬇️ 단순화 | API-007s |
| API-008 | OpenAPI Validation | ⏩ Phase 3 연기 | VAL-001 (Zod) |
| AUTH-C-001~005 | 인증 전체 | ⬇️ 핵심만 | AUTH-001s~004s |
| SA-C-002a | 정규화 | ✅ 유지 | SA-001s |
| SA-C-002b | AI 매핑 | ⬇️ 규칙 기반 | SA-002s (Phase 3) + AI-MAP (Phase 4) |
| SA-C-002c | 결과 검증 | ✅ 유지 | SA-003s |
| SA-C-002d | Fallback | ⛔ 제거 | Phase 4의 AI-MAP-003으로 대체 |
| INT-C-001~005 | 무결성 전체 | ⬇️ 대폭 축소 | LOG-001~003 |
| AI-C-001~008 | AI Governance | ⛔ 전체 제거 | Phase 5+ |
| RES-001~003 | DLQ/Retry/Feature Flag | ⬇️ 축소 | EDGE-003s (Retry 1회) |
| ZUI-C-003a | Raw 저장 | ✅ 유지 | EDGE-001s |
| ZUI-C-003b | Sync Queue | ⛔ 제거 | — |
| ZUI-C-003c | AI 구조화 | ⬇️ JSON 저장 | EDGE-001s + AI-STR (Phase 4) |
| ZUI-C-003d | Integrity | ⛔ 제거 | — |

---

## 🚨 설계 검증 체크리스트

모든 태스크에 대해 아래 4가지를 확인:

| 질문 | YES면 | NO면 |
|---|---|---|
| 이 기능 없어도 시스템 동작하는가? | Optional 후보 | 필수 유지 |
| 디버깅 가능한가? | 통과 | **request_id + step_name 추가** |
| 로그만으로 문제를 찾을 수 있는가? | 통과 | **LOG 호출 추가** |
| 하루 안에 구현 가능한가? | 통과 | **⚠️ 태스크 분할 필요** |

> [!CAUTION]
> 하나라도 NO인 태스크는 **설계 과도** → 즉시 단순화하라.

---

**— END OF TASK LIST v3 (Beginner Edition) —**

