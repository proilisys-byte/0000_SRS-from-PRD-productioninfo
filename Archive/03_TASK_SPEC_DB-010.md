---
name: Feature Task
about: SRS 기반의 구체적인 개발 태스크 명세
title: "[Feature] DB-010: AUDIT_LOG Insert-only 테이블 스키마 작성 (Merkle 해시 체인)"
labels: 'feature, backend, database, security, priority:critical, sprint:0'
assignees: ''
requires_human_review: true
---

## 🎯 Summary
- 기능명: [DB-010] AUDIT_LOG Insert-only 테이블 스키마 작성 (Merkle 해시 체인, storage_type 고정)
- 목적: 시스템 전체의 감사 추적(Audit Trail) 데이터를 위변조 불가능한 방식으로 기록하는 Insert-only AUDIT_LOG 테이블을 설계한다. 각 레코드는 SHA-256 해시와 이전 레코드의 해시를 체이닝(Merkle Hash Chain)하여 무결성을 암호학적으로 보장한다. 이 테이블은 Smart Audit, NC 시정, Lean 진단, RBAC 권한 변경 등 시스템 전반의 모든 감사 이벤트를 기록하는 SSOT(Single Source of Truth)이다.

## 🔗 References (Spec & Context)
> 💡 AI Agent & Dev Note: 작업 시작 전 아래 문서를 반드시 먼저 Read/Evaluate 할 것.
- SRS 문서: [`05_SRS_v1.md#§4.3.6`] — 데이터 무결성 시스템 시퀀스 다이어그램
- SRS 문서: [`05_SRS_v1.md#REQ-FUNC-024`] — Insert-only 방식 감사 로그 적재 요구사항
- SRS 문서: [`05_SRS_v1.md#REQ-NF-016`] — 감사 로그 보존 기간 ≥ 3년
- SRS 문서: [`05_SRS_v1.md#REQ-NF-030`] — 로그 위변조 탐지 100%
- SRS 문서: [`05_SRS_v1.md#§4.2.10.4`] — Cryptographic Erasure 규제 충돌 해소 로직
- SRS 문서: [`05_SRS_v1.md#§3.4.3`] — ERD (INSERT_ONLY_AUDIT_LOG 엔티티)
- 관련 태스크: [`07_TASK_SPEC_SA-C-002.md`] — SA-C-002가 AUDIT_LOG 기록을 요구
- 프로젝트 컨텍스트: [`PROJECT_CONTEXT.md`] — 디렉토리 구조, 환경 변수, 코딩 컨벤션

> 🔍 **requires_human_review: true** — H 복잡도. Merkle 해시 체인 설계 리뷰 필수.

## 📁 Implementation Paths
- `prisma/schema.prisma` — AuditLog 모델 정의 추가
- `prisma/migrations/` — AUDIT_LOG 테이블 마이그레이션
- `src/types/audit-log.ts` — AuditLog 관련 타입 정의 (event_type, action enum)

## ✅ Task Breakdown (실행 계획)
- [ ] `AuditLog` Prisma 모델 정의 (log_id UUID PK, tenant_id FK)
- [ ] 핵심 필드 정의: event_type enum, actor_id FK, target_entity, target_id, action enum
- [ ] 감사 데이터 필드: before_state JSONB, after_state JSONB, metadata JSONB
- [ ] 해시 체인 필드: hash_sha256 String, prev_hash String (이전 레코드 해시 참조)
- [ ] 타임스탬프 필드: created_at DateTime (수정 불가, updatedAt 없음)
- [ ] storage_type 필드: String @default("WORM") — 고정값, 변경 불가
- [ ] 인덱스 설정: tenant_id + created_at 복합 인덱스, event_type 인덱스, actor_id 인덱스
- [ ] Prisma 마이그레이션 생성 및 SQLite 환경 적용 확인
- [ ] JSONB ↔ TEXT 호환 처리 (SQLite 환경 대응, §3.6.1 Prisma Client Extensions 참조)
- [ ] 단위 테스트: Insert 동작 및 해시 체인 연속성 검증

## 🧪 Acceptance Criteria (BDD/GWT)

**Scenario 1: 감사 로그 정상 Insert**
- Given: 유효한 tenant_id와 actor_id가 존재하고, 이전 로그의 hash가 알려져 있음
- When: event_type="REPORT_GENERATED", action="CREATE"로 AUDIT_LOG Insert를 수행함
- Then: 레코드가 정상 생성되고, hash_sha256은 `SHA256(prev_hash + event_data + timestamp)`으로 산출되며, prev_hash는 직전 레코드의 hash_sha256과 일치한다.

**Scenario 2: 해시 체인 연속성 검증**
- Given: AUDIT_LOG에 10건의 레코드가 순차적으로 적재되어 있음
- When: 전체 해시 체인 검증 로직을 실행함
- Then: 모든 레코드의 hash_sha256이 이전 레코드의 prev_hash와 올바르게 체이닝되어 있으며, 체인 무결성 검증 결과가 100% 통과한다.

**Scenario 3: before/after 상태 기록**
- Given: 사용자 역할이 "User"에서 "Admin"으로 변경되는 이벤트가 발생함
- When: 권한 변경 감사 로그가 적재됨
- Then: before_state에 `{"role": "User"}`, after_state에 `{"role": "Admin"}`이 정확히 기록된다.

**Scenario 4: 최초 레코드 (Genesis) 처리**
- Given: 해당 tenant에 AUDIT_LOG 레코드가 0건인 상태임
- When: 최초 감사 로그를 Insert함
- Then: prev_hash는 사전 정의된 Genesis Hash (예: `"0x0"` 또는 빈 문자열의 SHA-256)로 설정되고, hash_sha256이 정상 산출된다.

## ⚙️ Technical & Non-Functional Constraints
- 무결성: SHA-256 해시 + Merkle 체인 필수 (REQ-FUNC-024, REQ-NF-030)
- 보존: 최소 3년 보존 의무 (REQ-NF-016)
- 불변성: created_at만 존재, updated_at 없음. 모든 레코드는 Insert 후 수정 불가
- DB 정책: Supabase 운영 환경에서 DELETE/UPDATE 차단 DB Policy 적용 (DB-017에서 구현)
- 성능: 대량 Insert 시에도 해시 체인 계산이 병목이 되지 않도록 비동기 처리 고려
- JSONB 호환: SQLite에서는 TEXT로 저장, PostgreSQL에서는 JSONB로 저장 (§3.6.1)
- 접근 제어: AUDIT_LOG 조회는 DPO·Auditor 역할만 허용 (REQ-NF-SEC-LOG-001)

## 🏁 Definition of Done (DoD)
- [ ] 모든 Acceptance Criteria를 충족하는가?
- [ ] Prisma 마이그레이션 파일이 생성되고 SQLite에서 정상 적용되는가?
- [ ] 해시 체인 필드(hash_sha256, prev_hash)가 스키마에 포함되어 있는가?
- [ ] JSONB 필드의 SQLite↔PostgreSQL 호환 처리가 확인되었는가?
- [ ] before_state / after_state JSONB 구조가 문서화되었는가?
- [ ] event_type / action enum 정의가 확장 가능한 구조인가?
- [ ] 단위 테스트가 추가되었고 통과하는가?
- [ ] ESLint / Prisma validate 경고가 없는가?

## 🚧 Dependencies & Blockers
- **Depends on:** DB-001 (SITE/TENANT 테이블 — tenant_id FK 참조)
- **Blocks:** DB-017 (Insert-only DB Policy 적용), INT-C-001 (IntegrityManager 공통 모듈), INT-C-002 (Merkle 해시 체인 적재 로직), INT-C-003 (DELETE/UPDATE 거부 처리), SA-C-003 (리포트 해시 체인 기록), LEAN-C-004 (진단 파라미터 감사 기록), AUTH-C-005 (RBAC 권한 변경 감사 로그), INFRA-006 (민감정보 마스킹), INFRA-007 (Meta-Audit Log)
