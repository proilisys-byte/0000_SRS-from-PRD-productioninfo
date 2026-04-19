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
| SA-C-002 | SmartAudit/Command | Audit 리포트 생성 Server Action — Gemini AI 매핑 엔진 연동 (Vercel AI SDK streamText) | REQ-FUNC-001, REQ-FUNC-002 | DB-003, DB-005, DB-006, SA-Q-001 | H |
| SA-C-003 | SmartAudit/Command | 리포트 생성 시 AUDIT_LOG SHA-256 해시 체인 기록 로직 | REQ-FUNC-001, REQ-FUNC-024 | DB-010, SA-C-002 | M |

### Epic 6: 긴급 NC 시정 패키지

| Task ID | Epic | Feature (기능명) | 관련 SRS 섹션 | 선행 태스크 | 복잡도 |
|---|---|---|---|---|---|
| NC-Q-001 | NC/Query | NC 케이스 상세 및 시정 진행 상태 조회 (GET /api/v1/nc/cases/{nc_id}) | §6.1 #6 | DB-007, DB-008, API-004 | L |
| NC-Q-002 | NC/Query | 유사 NC 사례 검색 (동일 nc_code 기반 최대 5건) | REQ-FUNC-010 | DB-007 | M |
| NC-C-001 | NC/Command | NC 케이스 등록 + AI 사유 파싱 + 시정 초안 자동 생성 (POST /api/v1/nc/cases) | REQ-FUNC-006 | DB-007, DB-008, NC-Q-002 | H |
| NC-C-002 | NC/Command | 시정 조치 상태·진행률 갱신 (PATCH .../actions/{action_id}) | REQ-FUNC-007 | DB-008, API-004 | M |
| NC-C-003 | NC/Command | 시정 전후 무결성 비교 보고서 생성 (SHA-256 메타데이터 포함) | REQ-FUNC-008 | NC-C-002, DB-011 | H |
| NC-C-004 | NC/Command | Critical 심각도 에스컬레이션 알림 발송 로직 (24h 카운트다운, 4h 간격) | REQ-FUNC-009 | NC-C-001 | M |

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
## Step 2 (계속). AI 거버넌스 태스크
### Epic 11: AI 시스템 거버넌스

| Task ID | Epic | Feature (기능명) | 관련 SRS 섹션 | 선행 태스크 | 복잡도 |
|---|---|---|---|---|---|
| AI-C-001 | AI-Gov/Command | Model Card 메타데이터 JSON 스키마 검증 로직 (배포 파이프라인 연동) | REQ-FUNC-AI-001 | DB-012 | M |
| AI-C-002 | AI-Gov/Command | 정량 기반 배포 차단 — 모델 버전 정량 불일치 시 배포 중단 + 관리자 알림 | REQ-FUNC-AI-002 | DB-012, AI-C-001 | H |
| AI-C-003 | AI-Gov/Command | 일관성 교감 검증(IAA) 계수 계산 + 미달 시 데이터증강 사용 제한 처리 | REQ-FUNC-AI-003 | DB-014 | M |
| AI-C-004 | AI-Gov/Command | Drift 발생 경고 자동화 — Golden Dataset 대비 성능 기준치 초과 시 알림 | REQ-FUNC-AI-004 | DB-014, DB-012 | H |
| AI-C-005 | AI-Gov/Command | 환경별 검증 — N회 반복 출력 분산 및 리젝트 시 전환 | REQ-FUNC-AI-005 | DB-013 | H |
| AI-C-006 | AI-Gov/Command | HitL 강제 프로세스 — 미승인 시간 확인 및 상태 업데이트 403 반환 | REQ-FUNC-AI-006 | DB-013, AUTH-C-005 | M |
| AI-C-007 | AI-Gov/Command | AI 추론 로그 완전 기록 (input/output hash, model version, confidence) | REQ-NF-032 | DB-013 | M |
| AI-C-008 | AI-Gov/Command | 어그리게이트 성향성 경고 — 하위 그룹 성능 차이 >5%p 시 학습 리셋 알림 | REQ-FUNC-AI-008 | DB-013, AI-C-004 | M |

### Epic 12: 글로벌 벤더 등록 가시화 (Phase 2)

| Task ID | Epic | Feature (기능명) | 관련 SRS 섹션 | 선행 태스크 | 복잡도 |
|---|---|---|---|---|---|
| VD-Q-001 | Vendor/Query | 심층 체크리스트 가중 분석 결과 조회 | REQ-FUNC-016 | DB-004 | M |
| VD-C-001 | Vendor/Command | 심사 체크리스트 매핑 + 가중 분석 로직 구현 | REQ-FUNC-016 | VD-Q-001 | H |
| VD-C-002 | Vendor/Command | 자재 소요 예산/기간 시뮬레이션 데이터 시행 로직 | REQ-FUNC-017 | VD-C-001 | M |
| VD-C-003 | Vendor/Command | 글로벌 증적 패키지 정돈 대보관기 (ZIP 다운로드) | REQ-FUNC-018 | VD-C-001 | M |

---

## Step 3. 테스트 태스크 (AC → Test Code 변환)

### Epic 13: Smart Audit 테스트
| Task ID | Epic | Feature (기능명) | 관련 SRS 섹션 | 선행 태스크 | 복잡도 |
|---|---|---|---|---|---|
| TEST-SA-001 | Test/SmartAudit | PDF 정상 생성 + 양식 요소 일치 통합 테스트 | REQ-FUNC-001 AC | SA-C-002 | M |
| TEST-SA-002 | Test/SmartAudit | 필수 필드 100% 매핑 + 정확도 ≥99% 검증 테스트 (Golden Dataset 기반) | REQ-FUNC-002 AC, Appendix A.1 | SA-C-002 | H |
| TEST-SA-003 | Test/SmartAudit | 이상 유형 분해 + 경고 매칭 UI 표시 검증 테스트 | REQ-FUNC-003 AC, Appendix A.6 | SA-Q-005 | M |
| TEST-SA-004 | Test/SmartAudit | 템플릿 필드 구조 렌더링 정합성 테스트 | REQ-FUNC-004 AC | SA-Q-001 | L |
| TEST-SA-005 | Test/SmartAudit | 버전 이력 비교 diff — 이전값과 현재값 병렬 표시 테스트 | REQ-FUNC-005 AC | SA-Q-004 | L |

### Epic 14: NC 시정 테스트
| Task ID | Epic | Feature (기능명) | 관련 SRS 섹션 | 선행 태스크 | 복잡도 |
|---|---|---|---|---|---|
| TEST-NC-001 | Test/NC | NC 사유 파싱 및 시정 초안 생성 커버리지 ≥95% 테스트 (Appendix A.4 기준) | REQ-FUNC-006 AC | NC-C-001 | H |
| TEST-NC-002 | Test/NC | 시정 진행률 갱신 ≤0초 반영 테스트 | REQ-FUNC-007 AC | NC-C-002 | M |
| TEST-NC-003 | Test/NC | 시정 전후 비교 보고서 무결성 해시 포함 검증 테스트 | REQ-FUNC-008 AC | NC-C-003 | M |
| TEST-NC-004 | Test/NC | Critical 건 에스컬레이션 발송 검증 (기한 초과 시 상위 책임자 알림) | REQ-FUNC-009 AC | NC-C-004 | M |
| TEST-NC-005 | Test/NC | 유사 NC 사례 최대 5건 반환 + 결과 정확성 테스트 | REQ-FUNC-010 AC | NC-Q-002 | L |

### Epic 15: Zero-UI 테스트
| Task ID | Epic | Feature (기능명) | 관련 SRS 섹션 | 선행 태스크 | 복잡도 |
|---|---|---|---|---|---|
| TEST-ZUI-001 | Test/ZeroUI | 음성 STT — WER ≤8%, CSR ≥92% 검증 테스트 (Appendix A.2 기준) | REQ-FUNC-011 AC | ZUI-C-001 | H |
| TEST-ZUI-002 | Test/ZeroUI | Vision AI — Precision ≥90%, Recall ≥85% 검증 테스트 (Appendix A.3 기준) | REQ-FUNC-012 AC | ZUI-C-002 | H |
| TEST-ZUI-003 | Test/ZeroUI | 네트워크 장애 시 즉각 오류 알림 + 재시도 유도 테스트 | REQ-FUNC-013 AC | ZUI-C-005 | M |
| TEST-ZUI-004 | Test/ZeroUI | AI 인식 3회 연속 실패 시 수동 입력 Fallback UI 전환 테스트 | REQ-FUNC-015 AC | ZUI-C-006 | M |
| TEST-ZUI-005 | Test/ZeroUI | Edge→Cloud 전송 유실률 0% + SHA-256 무결성 검증 테스트 | REQ-NF-010, REQ-NF-011 | ZUI-C-003 | H |

### Epic 16: Lean 진단 테스트
| Task ID | Epic | Feature (기능명) | 관련 SRS 섹션 | 선행 태스크 | 복잡도 |
|---|---|---|---|---|---|
| TEST-LEAN-001 | Test/Lean | COPQ 4대 낭비 환산 정확도 ≥95% 테스트 (Appendix A.5 기준) | REQ-FUNC-019 AC | LEAN-C-002 | H |
| TEST-LEAN-002 | Test/Lean | ROI 곡선 그래프 렌더링 정합성 테스트 | REQ-FUNC-020 AC | LEAN-Q-001 | M |
| TEST-LEAN-003 | Test/Lean | 경영진 PDF — 차트+테이블 1페이지 생성 테스트 | REQ-FUNC-021 AC | LEAN-C-003 | M |
| TEST-LEAN-004 | Test/Lean | 7일 미만 / 이상치 >30% 시 경고 모달 발생 + 진단 중단 테스트 | REQ-FUNC-022 AC | LEAN-C-001 | L |
| TEST-LEAN-005 | Test/Lean | 진단 파라미터 변경 시 AUDIT_LOG 적재 검증 테스트 | REQ-FUNC-023 AC | LEAN-C-004 | L |

### Epic 17: 데이터 무결성 테스트
| Task ID | Epic | Feature (기능명) | 관련 SRS 섹션 | 선행 태스크 | 복잡도 |
|---|---|---|---|---|---|
| TEST-INT-001 | Test/Integrity | AUDIT_LOG Insert-only 정책 — DELETE/UPDATE 시도 시 거부 테스트 | REQ-FUNC-024 AC | INT-C-003 | M |
| TEST-INT-002 | Test/Integrity | RBAC 2단계 — Admin/User 역할 별 접근 100% 차단 테스트 | REQ-FUNC-026 AC | AUTH-C-005 | M |
| TEST-INT-003 | Test/Integrity | 멱등성 — 동일 요청 재시도 시 중복 데이터 생성 0% 테스트 | REQ-NF-026 | INT-C-005 | M |
| TEST-INT-004 | Test/Integrity | 트랜잭션 경계 롤백 — 고아 엔트리 0건 검증 테스트 | REQ-NF-025 | INT-C-004 | H |

### Epic 18: Auth 보안 테스트
| Task ID | Epic | Feature (기능명) | 관련 SRS 섹션 | 선행 태스크 | 복잡도 |
|---|---|---|---|---|---|
| TEST-AUTH-001 | Test/Auth | 패스워드 정책 위반 생성 0건 테스트 | REQ-NF-SEC-AUTH-003 AC | AUTH-C-002 | L |
| TEST-AUTH-002 | Test/Auth | 계정 잠금 — 5회 실패 후 15분 잠금 + 관리자 알림 테스트 | REQ-NF-SEC-AUTH-005 AC | AUTH-C-003 | L |
| TEST-AUTH-003 | Test/Auth | 세션 유휴 30분 자동 로그아웃 테스트 | REQ-NF-SEC-AUTH-004 AC | AUTH-C-004 | L |
| TEST-AUTH-004 | Test/Auth | Rate Limiting — 초과 시 429 반환 100% 테스트 | REQ-NF-SEC-API-001 AC | API-007 | M |
| TEST-AUTH-005 | Test/Auth | Input Validation — 스키마 위반 요청 차단 100% 테스트 | REQ-NF-SEC-API-004 AC | API-008 | M |

### Epic 19: Bulk Import 테스트
| Task ID | Epic | Feature (기능명) | 관련 SRS 섹션 | 선행 태스크 | 복잡도 |
|---|---|---|---|---|---|
| TEST-BULK-001 | Test/BulkImport | 템플릿 다운로드 기능 검증 테스트 | REQ-FUNC-INT-001 AC | BULK-Q-001 | L |
| TEST-BULK-002 | Test/BulkImport | CSV/Excel 파싱 및 DB 적재 정합성 테스트 | REQ-FUNC-INT-002 AC | BULK-C-001 | M |
| TEST-BULK-003 | Test/BulkImport | 필수값 미입력·중복·스키마 불일치 오류 리포트 생성 테스트 | REQ-FUNC-INT-003 AC | BULK-C-002 | M |

### Epic 20: AI 거버넌스 테스트
| Task ID | Epic | Feature (기능명) | 관련 SRS 섹션 | 선행 태스크 | 복잡도 |
|---|---|---|---|---|---|
| TEST-AI-001 | Test/AI-Gov | Model Card 항목 100% 충족 Validation 테스트 | REQ-FUNC-AI-001 AC | AI-C-001 | M |
| TEST-AI-002 | Test/AI-Gov | 정량 불일치 시 배포 중단 + 알림 발송 테스트 | REQ-FUNC-AI-002 AC | AI-C-002 | M |
| TEST-AI-003 | Test/AI-Gov | HitL 미승인 시 403 반환 테스트 | REQ-FUNC-AI-006 AC | AI-C-006 | L |
| TEST-AI-004 | Test/AI-Gov | 추론 이벤트 대비 로그 누락률 0% 테스트 | REQ-NF-032 AC | AI-C-007 | M |

## Step 4. 비기능요약(NFR) 및 인프라 태스크 + UI 태스크
### Epic 21: 성능·보안 인프라
| Task ID | Epic | Feature (기능명) | 관련 SRS 섹션 | 선행 태스크 | 복잡도 |
|---|---|---|---|---|---|
| INFRA-001 | Infra/Perf | Vercel AI SDK streamText 기반 Streaming 아키텍처 적용 (Timeout 60s 회피) | REQ-NF-001, §3.6.2 | SA-C-002 | H |
| INFRA-002 | Infra/Perf | 클라이언트 측 PDF 렌더링 구현 (html2pdf.js) | REQ-NF-001 | SA-C-002 | M |
| INFRA-003 | Infra/Sec | AES-256 저장 데이터 암호화 확인 (Supabase 설정) | REQ-NF-013 | DB-001 | M |
| INFRA-004 | Infra/Sec | TLS 1.3 강제 적용 확인 (Vercel/Supabase) | REQ-NF-014 | None | L |
| INFRA-005 | Infra/Sec | API Rate Limiting 구현 (엔드포인트·API Key별) | REQ-NF-SEC-API-001 | API-007 | M |
| INFRA-006 | Infra/Sec | 로그 민감정보 자동 마스킹 파이프라인 구현 | REQ-NF-SEC-LOG-003 | DB-010 | H |
| INFRA-007 | Infra/Sec | Meta-Audit Log 구현 (감사 로그 접근 자체를 별도 기록) | REQ-NF-SEC-LOG-002 | DB-010 | M |
| INFRA-008 | Infra/Sec | Supabase/Cloud KMS 기반 DEK 관리 및 90일 로테이션 설정 | REQ-NF-SEC-KEY-001, KEY-003 | DB-001 | H |
| INFRA-009 | Infra/Sec | Cryptographic Erasure 구현 — DEK 파기 기반 일괄 데이터 제거 프로세스 | §4.2.10.4 | INFRA-008 | H |
| INFRA-010 | Infra/Monitor | SLI 4종 모니터링 대시보드 구성 (가용성, 지연, 오류율, 처리량) | REQ-NF-022 | None | M |
| INFRA-011 | Infra/Monitor | 5단계 에스컬레이션 정책 및 Critical→RTO 1h 알림 설정 | REQ-NF-023 | INFRA-010 | M |
| INFRA-012 | Infra/Monitor | 6가지 KPI Warning/Critical 알림 기준치 정의 | REQ-NF-024 | INFRA-010 | M |
| INFRA-013 | Infra/Resilience | 외부 서비스 실패 재시도 정책 (지수 백오프 + Jitter, 최대 3회) | REQ-NF-027 | None | M |
| INFRA-014 | Infra/Resilience | 서킷 브레이커 + Fallback 회로 구현 (캐시/더미 데이터 회피) | REQ-NF-027-B, §4.3.7 | INFRA-013 | H |

### Epic 22: CI/CD 및 DevOps

| Task ID | Epic | Feature (기능명) | 관련 SRS 섹션 | 선행 태스크 | 복잡도 |
|---|---|---|---|---|---|
| DEVOPS-001 | DevOps | Vercel Git Push 자동 배포 파이프라인 구성 | C-TEC-007 | None | L |
| DEVOPS-002 | DevOps | GitHub CodeQL(SAST) + Dependabot 활성화 | §3.6.3, REQ-NF-SEC-VULN-002 | DEVOPS-001 | M |
| DEVOPS-003 | DevOps | GitHub Actions 단위 테스트 자동 실행 (PR 트리거) | §3.6.3 | DEVOPS-001 | M |
| DEVOPS-004 | DevOps | AI 모델 레지스트리 정량 검증 CI 게이트 구현 | REQ-NF-031, AI-C-002 | DEVOPS-003, AI-C-002 | H |

### Epic 23: 개인정보 및 규제 준수
| Task ID | Epic | Feature (기능명) | 관련 SRS 섹션 | 선행 태스크 | 복잡도 |
|---|---|---|---|---|---|
| PRIV-001 | Privacy | Edge 최초 활성화 시 민감정보 동의 프롬프트 구현 (5가지 언어 지원) | REQ-NF-PRIV-001 | ZUI-C-001 | H |
| PRIV-002 | Privacy | 동의 철회 유입 UI + SLA 3일 이내 데이터 파기 프로세스 구현 | REQ-NF-PRIV-002 | PRIV-001, INFRA-009 | H |
| PRIV-003 | Privacy | Edge-side 가명처리 — 음성 위치 비식별 변환 + 영상 자동 마스킹 | REQ-NF-PRIV-003 | ZUI-C-001, ZUI-C-002 | H |
| PRIV-004 | Privacy | 정보주체 권리행사 (열람·정정·삭제·처리정지) 작수워크플로 자동화 프로세스 | REQ-NF-PRIV-004 | PRIV-002 | H |
| PRIV-005 | Privacy | 이용 목적 제한 — 목적 외 쿼리 접근/추출 기술적 차단 로직 | REQ-NF-PRIV-006 | DB-002, AUTH-C-005 | M |
| PRIV-006 | Privacy | 근로자 합의 미완료 시 Edge 배포 Lock 처리 로직 | REQ-NF-LABOR-001 | PRIV-001 | M |

### Epic 24: UI/UX — Web Manager Dashboard

| Task ID | Epic | Feature (기능명) | 관련 SRS 섹션 | 선행 태스크 | 복잡도 |
|---|---|---|---|---|---|
| UI-001 | UI/Dashboard | 로그인·대시보드 관리 화면 (Tailwind + shadcn/ui) | §4.2.4.1 | AUTH-C-001 | M |
| UI-002 | UI/Dashboard | RBAC 사용자 관리 UI (역할 변경, 목록 조회) | REQ-FUNC-026 | AUTH-C-005 | M |
| UI-003 | UI/Dashboard | Audit 리포트 생성 화면 (템플릿 선택 → AI 스트리밍 → PDF 다운로드) | REQ-FUNC-001, 004 | SA-C-002, INFRA-001, INFRA-002 | H |
| UI-004 | UI/Dashboard | 리포트 버전 비교 diff 뷰 화면 | REQ-FUNC-005 | SA-Q-004 | M |
| UI-005 | UI/Dashboard | AI 매핑 XAI 설명가능성 UI (매핑 결과 클릭 시 원문 위치 하이라이트) | REQ-FUNC-AI-007 | SA-C-002 | H |
| UI-006 | UI/Dashboard | NC 케이스 등록 + 시정 초안 확인 화면 | REQ-FUNC-006 | NC-C-001 | M |
| UI-007 | UI/Dashboard | NC 시정 진행률 대시보드 (실시간 갱신) | REQ-FUNC-007 | NC-C-002 | M |
| UI-008 | UI/Dashboard | 시정 전후 비교 보고서 화면 | REQ-FUNC-008 | NC-C-003 | M |
| UI-009 | UI/Dashboard | COPQ 4대 낭비 시각화 대시보드 (차트 렌더링) | REQ-FUNC-019, 020 | LEAN-C-002 | H |
| UI-010 | UI/Dashboard | 데이터 유의성 경고 배너/모달 컴포넌트 | REQ-FUNC-022 | LEAN-C-001 | L |
| UI-011 | UI/Dashboard | 경영진 PDF Export 버튼 + 다운로드 UI | REQ-FUNC-021 | LEAN-C-003 | L |
| UI-012 | UI/Dashboard | 감사 로그 조회 화면 (DPO/Auditor 전용) | REQ-NF-SEC-LOG-001 | INT-C-002 | M |
| UI-013 | UI/Dashboard | 기준정보 벌크 업로드 화면 (파일 선택 → 검증 → 결과 리포트) | REQ-FUNC-INT-001~003 | BULK-C-001, BULK-C-002 | M |
| UI-014 | UI/Dashboard | 품질 이상 탐지 경고 매칭 UI 표시 | REQ-FUNC-003 | SA-Q-005 | M |

### Epic 25: UI/UX — Edge Zero-UI 측
| Task ID | Epic | Feature (기능명) | 관련 SRS 섹션 | 선행 태스크 | 복잡도 |
|---|---|---|---|---|---|
| EDGE-UI-001 | UI/Edge | 음성 입력 인터페이스 (마이크 캡처 → STT 결과 확인 플로트) | REQ-FUNC-011 | ZUI-C-001 | H |
| EDGE-UI-002 | UI/Edge | 카메라 촬영 + Vision AI 판별 결과 표시 인터페이스 | REQ-FUNC-012 | ZUI-C-002 | H |
| EDGE-UI-003 | UI/Edge | 수동 입력 Fallback UI (3회 연속 AI 실패 시 전환) | REQ-FUNC-015 | ZUI-C-006 | M |
| EDGE-UI-004 | UI/Edge | 네트워크 오류 알림 + 재시도 유도 UI | REQ-FUNC-013 | ZUI-C-005 | L |
| EDGE-UI-005 | UI/Edge | 민감정보 동의 팝업 화면 (5가지 언어 지원) | REQ-NF-PRIV-001 | PRIV-001 | M |

### Epic 26: 사용성 NFR

| Task ID | Epic | Feature (기능명) | 관련 SRS 섹션 | 선행 태스크 | 복잡도 |
|---|---|---|---|---|---|
| USE-001 | Usability | WCAG 2.1 Level AA 접근성 적용 (색각이상 등 포함) | REQ-NF-USE-002 | UI-001~014 | M |
| USE-002 | Usability | 오류 복구 경로 구현 (Confirm 팝업, Undo, Draft 자동 저장) | REQ-NF-USE-003 | UI-001~014 | M |
| USE-003 | Usability | 인라인 Tooltip + 30분 튜토리얼 콘텐츠 준비 | REQ-NF-USE-005 | UI-001~014 | L |

---

## 상호간 매핑 요약 (Dependency Graph)

### 크리티컬 패스 (Critical Path)

```
DB-001~017 (스키마)
  → API-001~008 (DTO/에러 코드)
    → MOCK-001~006 (Mock 데이터·MSW)
      → [프론트엔드 UI 개발 착수 가능]

DB-001~017 (스키마)
  → INT-C-001~002 (IntegrityManager 공통 모듈)
    → SA-C-002 (Audit 리포트 생성)
    → NC-C-001 (NC 시정 초안 생성)
    → ZUI-C-003 (Edge→Cloud 동기화)

AUTH-C-001 (인증) → AUTH-C-005 (RBAC) → 전체 Command 태스크 (권한 검증 선행)
```

### Blocks / Depends-on 특기 관계
| 선행 태스크 (Blocks) | 후행 태스크 (Depends on) | 사유 |
|---|---|---|
| DB-001~017 | 모든 Command/Query 태스크 | 테이블 스키마가 SSOT |
| API-001~008 | MOCK-006 및 전체 UI 태스크 | DTO 계약이 프론트 및 검증 |
| INT-C-001~002 | SA-C-003, NC-C-003, LEAN-C-004 | SHA-256 해시 체인 공통 모듈 |
| AUTH-C-001 | AUTH-C-002~005 | 인증 기반 위에 정책 구현 |
| AUTH-C-005 | SA-C-001, AI-C-006, UI-002 | Admin 권한 검증 필요 |
| DB-012 (AI_MODEL_REGISTRY) | AI-C-001~008 전체 | AI 거버넌스 데이터 기반 |
| DB-014 (GOLDEN_DATASET) | AI-C-003, AI-C-004 | 검증 기준 데이터셋 |
| ZUI-C-001, ZUI-C-002 | PRIV-003 | 가명처리 — AI 모델 |
| INFRA-008 (KMS DEK) | INFRA-009 (Cryptographic Erasure) | 키 관리가 파기에 선행 |
| INFRA-009 | PRIV-002 (동의 철회 파기) | 일괄 파기 메커니즘 |
| SA-C-002, INFRA-001, INFRA-002 | UI-003 | 스트리밍+PDF 렌더링 |
| LEAN-C-002 | UI-009 | 차트 데이터 소스 |
| BULK-C-001, BULK-C-002 | UI-013 | 업로드 로직 완성 후 UI |

---

## Sprint 배치 가이드

| Sprint | 주요 태스크 ID | 비고 |
|---|---|---|
| **Sprint 0 (1주)** | DB-001~017, API-001~008, MOCK-001~006 | 계약·데이터 SSOT 확립 |
| **Sprint 1 (2주)** | AUTH-C-001~005, SA-C-002, SA-Q-001~002, ZUI-C-001~003, INT-C-001~003, BULK-C-001~002, INFRA-001~002, DEVOPS-001~003, UI-001~003, EDGE-UI-001~002 | Slice-1 Core (REQ-FUNC-001,002,011,024,026,030) |
| **Sprint 2 (2주)** | NC-C-001~004, ZUI-C-002~006, PRIV-001~003, UI-006~008, EDGE-UI-003~005 | NC 시정 + Vision AI + 가명처리 |
| **Sprint 3 (2주)** | LEAN-C-001~004, AI-C-001~008, UI-009~012, INFRA-010~012 | Lean 진단 + AI 거버넌스 |
| **Sprint 4 (2주)** | VD-C-001~003, PRIV-004~006, USE-001~003, INFRA-006~009, 전체 TEST 태스크 | 마무리·폴리싱·테스트 |

---

## 태스크 통계 요약

| 카테고리 | 태스크 수 |
|---|---|
| DB Schema (Epic 1) | 17 |
| API Spec (Epic 2) | 8 |
| Mock Data (Epic 3) | 6 |
| Auth (Epic 4) | 7 |
| Smart Audit (Epic 5) | 8 |
| NC 시정 (Epic 6) | 6 |
| Zero-UI (Epic 7) | 7 |
| Lean 진단 (Epic 8) | 6 |
| 데이터 무결성 (Epic 9) | 5 |
| Bulk Import (Epic 10) | 3 |
| AI 거버넌스 (Epic 11) | 8 |
| 글로벌 벤더 Phase2 (Epic 12) | 4 |
| 테스트 (Epic 13~20) | 34 |
| 인프라·보안 (Epic 21) | 14 |
| CI/CD (Epic 22) | 4 |
| 개인정보 (Epic 23) | 6 |
| UI/Dashboard (Epic 24) | 14 |
| UI/Edge (Epic 25) | 5 |
| 사용성 (Epic 26) | 3 |
| **합계** | **165** |

---

**— END OF TASK LIST —**
