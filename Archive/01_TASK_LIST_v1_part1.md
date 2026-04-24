# PRO ILI SMART — 개발 태스크 목록 명세서 (Task Breakdown)

| 항목 | 내용 |
|:---|:---|
| **기준 문서** | SRS_v1.md |
| **작성일** | 2026-04-19 |
| **작성자** | Technical PM / System Architect |
| **목적** | SRS 기반 실행 가능한 개발 태스크(Epic/Feature) 도출 |

---

## Step 1. 계약·데이터 명세 태스크 (Contract & Data Layer)

### Epic 1: 데이터베이스 스키마 및 마이그레이션

| Task ID | Epic | Feature (기능명) | 관련 SRS 섹션 | 선행 태스크 | 복잡도 |
|---|---|---|---|---|---|
| DB-001 | DB Schema | SITE 테이블 스키마 및 Prisma 마이그레이션 작성 | §6.2.1 | None | L |
| DB-002 | DB Schema | USER 테이블 스키마 및 Prisma 마이그레이션 작성 (role enum: Admin/User) | §6.2.1, §4.2.4.2 | DB-001 | L |
| DB-003 | DB Schema | RAW_DATA 테이블 스키마 작성 (source_type enum, SHA-256 필드 포함) | §6.2.2 | DB-001 | M |
| DB-004 | DB Schema | TEMPLATE 테이블 스키마 작성 (JSONB schema_definition, 버전 관리) | §6.2.8 | DB-001 | L |
| DB-005 | DB Schema | AUDIT_SESSION 테이블 스키마 작성 (status enum, template FK) | §6.2.3 | DB-001, DB-004 | L |
| DB-006 | DB Schema | AUDIT_REPORT 테이블 스키마 작성 (version, hash_sha256 포함) | §6.2.4 | DB-005 | L |
| DB-007 | DB Schema | NC_CASE 테이블 스키마 작성 (severity/status enum) | §6.2.5 | DB-001 | L |
| DB-008 | DB Schema | CORRECTIVE_ACTION 테이블 스키마 작성 (progress, evidence_url) | §6.2.6 | DB-007 | L |
| DB-009 | DB Schema | LEAN_DIAGNOSIS 테이블 스키마 작성 (waste_breakdown JSONB, data_quality_flag) | §6.2.7 | DB-001 | L |
| DB-010 | DB Schema | AUDIT_LOG Insert-only 테이블 스키마 작성 (Merkle 해시 체인, storage_type 고정) | §6.2.9 | DB-001 | H |
| DB-011 | DB Schema | SUBMISSION_LOG 테이블 스키마 작성 (Insert-only) | §6.2.10 | DB-001 | L |
| DB-012 | DB Schema | AI_MODEL_REGISTRY 테이블 스키마 작성 | §6.2.11 | DB-001 | M |
| DB-013 | DB Schema | AI_INFERENCE_LOG 테이블 스키마 작성 | §6.2.12 | DB-012 | M |
| DB-014 | DB Schema | GOLDEN_DATASET 테이블 스키마 작성 (IAA score CHECK ≥ 0.8) | §6.2.13, §B.6 | DB-001 | L |
| DB-015 | DB Schema | REPORT_MAPPING 테이블 스키마 작성 (report↔data 매핑 추적) | §6.9 | DB-003, DB-006 | L |
| DB-016 | DB Schema | Prisma Client Extensions — SQLite↔PostgreSQL JSON 호환 미들웨어 구현 | §3.6.1 | DB-001~015 | H |
| DB-017 | DB Schema | Supabase AUDIT_LOG 테이블 Insert-only DB Policy (DELETE/UPDATE 차단) 적용 | §REQ-FUNC-024, §4.2.3 | DB-010 | H |

### Epic 2: API 및 통신 계약 (DTO / Error Code 정의)

| Task ID | Epic | Feature (기능명) | 관련 SRS 섹션 | 선행 태스크 | 복잡도 |
|---|---|---|---|---|---|
| API-001 | API Spec | Auth 도메인 Request/Response DTO 정의 (로그인, 세션, 역할) | §4.2.4.1~4.2.4.2 | DB-002 | M |
| API-002 | API Spec | Audit Report API DTO 정의 (POST /api/v1/audit/reports, GET .../{report_id}) | §6.1 #1~2 | DB-005, DB-006 | M |
| API-003 | API Spec | Zero-UI Edge Sync API DTO 정의 (POST /api/v1/edge/sync, GET .../status) | §6.1 #3~4 | DB-003 | M |
| API-004 | API Spec | NC Response API DTO 정의 (POST, GET, PATCH) | §6.1 #5~7 | DB-007, DB-008 | M |
| API-005 | API Spec | Lean Diagnosis API DTO 정의 (POST /diagnose, GET /export) | §6.1 #8~9 | DB-009 | M |
| API-006 | API Spec | Template Registry API DTO 정의 (GET 목록/상세, POST 등록) | §6.1 #10~12 | DB-004 | L |
| API-007 | API Spec | 공통 에러 코드 체계 정의 (400/401/403/404/409/429/500) | §4.2.4.4 | None | M |
| API-008 | API Spec | OpenAPI 3.0 스키마 기반 Input Validation 명세 작성 | REQ-NF-SEC-API-004 | API-001~006 | H |

### Epic 3: Mock 데이터 및 Fixture

| Task ID | Epic | Feature (기능명) | 관련 SRS 섹션 | 선행 태스크 | 복잡도 |
|---|---|---|---|---|---|
| MOCK-001 | Mock Data | SITE·USER Seed 데이터 및 Fixture 생성 | §6.2.1 | DB-001, DB-002 | L |
| MOCK-002 | Mock Data | TEMPLATE Seed 데이터 (ISO9001/14001/45001 샘플 양식 3종) | §6.2.8 | DB-004 | L |
| MOCK-003 | Mock Data | RAW_DATA 샘플 데이터 (voice/vision/manual 각 10건) | §6.2.2 | DB-003 | L |
| MOCK-004 | Mock Data | NC_CASE + CORRECTIVE_ACTION 샘플 데이터 (critical/major/minor 각 3건) | §6.2.5~6 | DB-007, DB-008 | L |
| MOCK-005 | Mock Data | LEAN_DIAGNOSIS 샘플 데이터 (7일+ 생산 데이터 포함) | §6.2.7 | DB-009 | L |
| MOCK-006 | Mock Data | 프론트엔드 개발용 Mocking API 엔드포인트 세팅 (MSW 또는 Next.js Route Handler) | §3.3 | API-001~006 | M |

---

## Step 2. 로직·상태 변경 태스크 (Logic & Mutation — CQRS 분리)

### Epic 4: 인증·인가 (Auth & RBAC)

| Task ID | Epic | Feature (기능명) | 관련 SRS 섹션 | 선행 태스크 | 복잡도 |
|---|---|---|---|---|---|
| AUTH-Q-001 | Auth/Query | Supabase Auth 기반 현재 사용자 세션 조회 로직 구현 | §4.2.4.1 | DB-002, API-001 | M |
| AUTH-Q-002 | Auth/Query | 사용자 역할(Admin/User) 권한 매트릭스 조회 로직 구현 | REQ-NF-SEC-AUTHZ-001 | AUTH-Q-001 | L |
| AUTH-C-001 | Auth/Command | Supabase Auth 기반 회원가입·로그인 Server Action 구현 | §4.2.4.1 | DB-002, API-001 | M |
| AUTH-C-002 | Auth/Command | 패스워드 정책 검증 로직 구현 (12자, 3종 이상, 재사용 5개 금지, 90일 만료) | REQ-NF-SEC-AUTH-003 | AUTH-C-001 | M |
| AUTH-C-003 | Auth/Command | 계정 잠금 로직 구현 (연속 5회 실패 → 15분 잠금) | REQ-NF-SEC-AUTH-005 | AUTH-C-001 | M |
| AUTH-C-004 | Auth/Command | 세션 관리 구현 (유휴 30분 로그아웃, JWT TTL ≤ 1h, Refresh 7d) | REQ-NF-SEC-AUTH-004 | AUTH-C-001 | H |
| AUTH-C-005 | Auth/Command | RBAC 권한 변경 Server Action 구현 (Admin만 가능, 감사 로그 연동) | REQ-FUNC-026 | AUTH-Q-002, DB-010 | M |

### Epic 5: Smart Audit 엔진

| Task ID | Epic | Feature (기능명) | 관련 SRS 섹션 | 선행 태스크 | 복잡도 |
|---|---|---|---|---|---|
| SA-Q-001 | SmartAudit/Query | 템플릿 목록 조회 (GET /api/v1/templates) | REQ-FUNC-004, §6.1 #10 | DB-004, API-006 | L |
| SA-Q-002 | SmartAudit/Query | 템플릿 상세 조회 (GET /api/v1/templates/{id}) | REQ-FUNC-004, §6.1 #11 | SA-Q-001 | L |
| SA-Q-003 | SmartAudit/Query | 기생성 Audit 리포트 조회 (GET /api/v1/audit/reports/{id}) | §6.1 #2 | DB-006, API-002 | L |
| SA-Q-004 | SmartAudit/Query | 리포트 버전 이력 비교 diff 조회 로직 구현 | REQ-FUNC-005 | SA-Q-003 | M |
| SA-Q-005 | SmartAudit/Query | 품질 이상 탐지 Trend 분석 데이터 조회 (과거 3개월) | REQ-FUNC-003 | DB-003 | M |
| SA-C-001 | SmartAudit/Command | 커스텀 템플릿 등록 Server Action (POST /api/v1/templates) | §6.1 #12 | DB-004, AUTH-C-005 | M |
| SA-C-002 | SmartAudit/Command | Audit 리포트 생성 Server Action — Gemini AI 매핑 엔진 연동 (Vercel AI SDK streamText) 🔍 | REQ-FUNC-001, REQ-FUNC-002 | DB-003, DB-005, DB-006, SA-Q-001, INT-C-001 | H |
| SA-C-003 | SmartAudit/Command | 리포트 생성 시 AUDIT_LOG SHA-256 해시 체인 기록 로직 | REQ-FUNC-001, REQ-FUNC-024 | DB-010, SA-C-002 | M |

### Epic 6: 긴급 NC 시정 패키지

| Task ID | Epic | Feature (기능명) | 관련 SRS 섹션 | 선행 태스크 | 복잡도 |
|---|---|---|---|---|---|
| NC-Q-001 | NC/Query | NC 케이스 상세 및 시정 진행 상태 조회 (GET /api/v1/nc/cases/{nc_id}) | §6.1 #6 | DB-007, DB-008, API-004 | L |
| NC-Q-002 | NC/Query | 유사 NC 사례 검색 (동일 nc_code 기반 최대 5건) | REQ-FUNC-010 | DB-007 | M |
| NC-C-001a | NC/Command | NC 케이스 등록 Server Action (순수 CRUD — Zod 검증, NC_CASE 생성) | REQ-FUNC-006 | DB-007, DB-008, API-004 | M |
| NC-C-001b | NC/Command | AI 사유 파싱 + 시정 초안 자동 생성 엔진 (Gemini AI 연동) 🔍 | REQ-FUNC-006 | NC-C-001a, NC-Q-002 | H |
| NC-C-001c | NC/Command | Critical 에스컬레이션 이벤트 발행 + AUDIT_LOG 기록 | REQ-FUNC-006, REQ-FUNC-009 | NC-C-001a, INT-C-001 | M |
| NC-C-002 | NC/Command | 시정 조치 상태·진행률 갱신 (PATCH .../actions/{action_id}) | REQ-FUNC-007 | DB-008, API-004 | M |
| NC-C-003 | NC/Command | 시정 전후 무결성 비교 보고서 생성 (SHA-256 메타데이터 포함) 🔍 | REQ-FUNC-008 | NC-C-002, DB-011, INT-C-001 | H |
| NC-C-004 | NC/Command | Critical 심각도 에스컬레이션 알림 발송 로직 (24h 카운트다운, 4h 간격) | REQ-FUNC-009 | NC-C-001c | M |

### Epic 7: Zero-UI 수집기

| Task ID | Epic | Feature (기능명) | 관련 SRS 섹션 | 선행 태스크 | 복잡도 |
|---|---|---|---|---|---|
| ZUI-Q-001 | ZeroUI/Query | Edge 디바이스 연결 상태 조회 (GET /api/v1/edge/devices/{id}/status) | §6.1 #4 | API-003 | L |
| ZUI-C-001 | ZeroUI/Command | 음성 STT 변환 — Gemini Multimodal API 연동 (Vercel AI SDK) | REQ-FUNC-011 | DB-003, API-003 | H |
| ZUI-C-002 | ZeroUI/Command | Vision AI 부품 외관 판별 — Gemini Multimodal API 연동 | REQ-FUNC-012 | DB-003 | H |
| ZUI-C-003 | ZeroUI/Command | Edge→Cloud 실시간 데이터 동기화 Server Action (SHA-256 검증 + LLM 구조화) | REQ-FUNC-013, §4.3.3 | ZUI-C-001, ZUI-C-002, DB-003 | H |
| ZUI-C-004 | ZeroUI/Command | 대용량 파일(>4.5MB) Supabase Storage Direct Upload 시퀀스 구현 | §3.6.2 | ZUI-C-003 | M |
| ZUI-C-005 | ZeroUI/Command | 네트워크 오류 알림 및 재시도 유도 로직 구현 | REQ-FUNC-013 | ZUI-C-003 | M |
| ZUI-C-006 | ZeroUI/Command | AI 인식 실패 시 수동 입력 Fallback 전환 로직 (3회 연속 실패 → UI 전환) | REQ-FUNC-015 | ZUI-C-001 | M |

### Epic 8: Lean 진단 도구

| Task ID | Epic | Feature (기능명) | 관련 SRS 섹션 | 선행 태스크 | 복잡도 |
|---|---|---|---|---|---|
| LEAN-Q-001 | Lean/Query | ROI 양수 전환 추이 데이터 조회 | REQ-FUNC-020 | DB-009 | M |
| LEAN-Q-002 | Lean/Query | 진단 파라미터 변경 감사 이력 조회 | REQ-FUNC-023 | DB-010 | L |
| LEAN-C-001 | Lean/Command | 데이터 유효성 검증 (7일 미만 / 이상치 >30% 경고) | REQ-FUNC-022 | DB-003, DB-009 | M |
| LEAN-C-002 | Lean/Command | COPQ 4대 낭비 금액 환산 엔진 구현 (POST /api/v1/lean/diagnose) | REQ-FUNC-019 | LEAN-C-001, API-005 | H |
| LEAN-C-003 | Lean/Command | 경영진 요약 PDF Export 데이터 생성 (GET .../export) | REQ-FUNC-021 | LEAN-C-002 | M |
| LEAN-C-004 | Lean/Command | 진단 파라미터 변경 시 AUDIT_LOG 감사 기록 적재 | REQ-FUNC-023 | DB-010, LEAN-C-002 | L |

### Epic 9: 데이터 무결성 시스템

| Task ID | Epic | Feature (기능명) | 관련 SRS 섹션 | 선행 태스크 | 복잡도 |
|---|---|---|---|---|---|
| INT-C-001 | Integrity/Command | IntegrityManager — SHA-256 해시 생성 + 타임스탬프 부여 공통 모듈 구현 | REQ-FUNC-024, §6.5 | DB-010 | H |
| INT-C-002 | Integrity/Command | Insert-only AUDIT_LOG 적재 로직 및 Merkle 해시 체인 구현 | REQ-FUNC-024 | INT-C-001 | H |
| INT-C-003 | Integrity/Command | AUDIT_LOG DELETE/UPDATE 시도 시 거부 응답 처리 로직 | REQ-FUNC-024, §4.3.6 | DB-017 | M |
| INT-C-004 | Integrity/Command | 트랜잭션 경계 보상(Saga) — WORM 적재 실패 시 롤백 로직 | REQ-NF-025 | INT-C-002 | H |
| INT-C-005 | Integrity/Command | 멱등성(Idempotency) 키 기반 중복 처리 방지 미들웨어 | REQ-NF-026 | API-007 | M |

### Epic 10: 기준정보 벌크 업로드

| Task ID | Epic | Feature (기능명) | 관련 SRS 섹션 | 선행 태스크 | 복잡도 |
|---|---|---|---|---|---|
| BULK-Q-001 | BulkImport/Query | 마스터 데이터 표준 Excel/CSV 템플릿 다운로드 기능 | REQ-FUNC-INT-001 | DB-001 | L |
| BULK-C-001 | BulkImport/Command | CSV/Excel 파일 파싱 및 시스템 마스터 DB 일괄 적재 Server Action | REQ-FUNC-INT-002 | DB-001, DB-003 | H |
| BULK-C-002 | BulkImport/Command | 업로드 데이터 정규화 및 사전 검증 (필수값·중복·스키마 불일치 오류 리포트) | REQ-FUNC-INT-003 | BULK-C-001 | H |
