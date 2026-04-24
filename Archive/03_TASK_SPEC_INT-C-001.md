---
name: Feature Task
about: SRS 기반의 구체적인 개발 태스크 명세
title: "[Feature] INT-C-001: IntegrityManager — SHA-256 해시 생성 + 타임스탬프 부여 공통 모듈 구현"
labels: 'feature, backend, security, integrity, priority:critical, sprint:1'
assignees: ''
requires_human_review: true
---

## 🎯 Summary
- 기능명: [INT-C-001] IntegrityManager — SHA-256 해시 생성 + 타임스탬프 부여 공통 모듈 구현
- 목적: 시스템 전반에서 재사용되는 데이터 무결성 보장 공통 모듈을 구현한다. 모든 외부 제출 파일(Audit 리포트, NC 시정 보고서, Lean 진단 결과), 감사 로그 엔트리, Edge→Cloud 전송 데이터에 SHA-256 해시와 ISO 8601 타임스탬프를 자동 부여하여 위변조를 원천 차단한다.

> ⚠️ **v2 리팩토링: 3단계 레벨링 전략 적용**
> | Level | 기능 | Sprint | 비고 |
> |---|---|---|---|
> | **Level 1** | 단순 SHA-256 해시 생성 | Sprint 1 | 초기 — 최소 기능 |
> | **Level 2** | Append-only 로그 적재 | Sprint 1 | Insert-only 보장 |
> | **Level 3** | Merkle hash chain | Sprint 3+ | 고급 — 체인 무결성 |

## 🔗 References (Spec & Context)
> 💡 AI Agent & Dev Note: 작업 시작 전 아래 문서를 반드시 먼저 Read/Evaluate 할 것.
- SRS 문서: [`05_SRS_v1.md#REQ-FUNC-024`] — Insert-only 감사 로그 + SHA-256 무결성
- SRS 문서: [`05_SRS_v1.md#REQ-NF-011`] — 무결성 검증 통과율 100%
- SRS 문서: [`05_SRS_v1.md#REQ-NF-030`] — 로그 위변조 탐지 100%
- SRS 문서: [`05_SRS_v1.md#§4.3.6`] — 데이터 무결성 시스템 시퀀스 다이어그램 (IntegrityManager 역할)
- SRS 문서: [`05_SRS_v1.md#§4.3.3`] — Zero-UI 수집기 시퀀스 (Raw SHA-256 검증 흐름)
- 관련 태스크: [`07_TASK_SPEC_DB-010.md`] — AUDIT_LOG 테이블 (hash_sha256, prev_hash 필드)
- 태스크 목록: [`TASKS/06_TASK_LIST_v1.md#Epic9`] — 데이터 무결성 시스템 태스크 목록
- 프로젝트 컨텍스트: [`PROJECT_CONTEXT.md`] — 디렉토리 구조, 환경 변수, 코딩 컨벤션

> 🔍 **requires_human_review: true** — H 복잡도. AI 초안 생성 후 시니어 리뷰 필수.

## 📁 Implementation Paths
- `src/lib/integrity/integrity-manager.ts` — IntegrityManager 메인 모듈
- `src/lib/integrity/canonical-json.ts` — JSON 정규화 유틸리티
- `src/lib/integrity/merkle-chain.ts` — Merkle 해시 체인 유틸리티
- `src/types/integrity.ts` — IntegrityEnvelope 타입 정의

## ✅ Task Breakdown (실행 계획 — 3단계 레벨링)

### Level 1: 단순 SHA-256 (Sprint 1 — 초기)
- [ ] `IntegrityManager` 클래스/모듈 설계 (Singleton 패턴 또는 함수형 모듈)
- [ ] SHA-256 해시 생성 함수 구현
  - `generateHash(data: string | Buffer): string` — 입력 데이터의 SHA-256 해시 반환
  - `generateRecordHash(record: AuditableRecord): string` — 구조화된 레코드 해시 (정규화 후 해시)
  - JSON 정규화(Canonical JSON) 처리 — 키 정렬 후 해시하여 동일 데이터의 해시 일관성 보장
- [ ] 타임스탬프 부여 함수 구현
  - `stampTimestamp(): string` — ISO 8601 UTC 타임스탬프 생성
  - `createIntegrityEnvelope(data: any): IntegrityEnvelope` — 데이터 + 해시 + 타임스탬프 봉투 생성
- [ ] 파일 해시 함수 구현
  - `hashFile(buffer: Buffer): string` — 바이너리 파일(PDF, 이미지 등) SHA-256 해시
  - `hashStream(stream: ReadableStream): Promise<string>` — 스트림 기반 대용량 파일 해시
- [ ] IntegrityEnvelope 타입 정의
  ```typescript
  interface IntegrityEnvelope {
    data: any;
    hash_sha256: string;
    timestamp: string;       // ISO 8601 UTC
    prev_hash?: string;      // Merkle 체인용 (Level 3에서 활성화)
    algorithm: "SHA-256";
  }
  ```
- [ ] Edge→Cloud 전송 데이터 무결성 검증 유틸리티
  - `verifyTransmissionIntegrity(payload: Buffer, expectedHash: string): boolean`
- [ ] 단위 테스트: 해시 생성, 타임스탬프 형식, 파일 해시 정확성 검증

### Level 2: Append-only 로그 적재 (Sprint 1)
- [ ] Insert-only 적재 로직 (INT-C-002와 연동)
- [ ] prev_hash는 단순 이전 레코드 해시 참조 (체인 검증 없음)
- [ ] 성능 벤치마크: 1,000건 연속 해시 생성 시 처리 시간 측정

### Level 3: Merkle hash chain (Sprint 3+)
- [ ] Merkle 해시 체인 유틸리티
  - `chainHash(currentData: string, prevHash: string): string` — SHA256(prevHash + currentData) 체이닝
  - `verifyChain(records: AuditLogRecord[]): ChainVerificationResult` — 전체 체인 무결성 검증
- [ ] 체인 끊김 자동 탐지 + 알림
- [ ] 체인 복구 유틸리티 (관리자 전용)

## 🧪 Acceptance Criteria (BDD/GWT)

**Scenario 1: 데이터 해시 생성 정확성**
- Given: 동일한 JSON 데이터 `{"action": "CREATE", "target": "report_001"}`가 주어짐
- When: `generateRecordHash()`를 2회 호출함
- Then: 두 번 모두 동일한 SHA-256 해시 값을 반환한다 (결정론적 해시).

**Scenario 2: JSON 정규화 일관성**
- Given: 키 순서만 다른 동일 내용의 JSON 두 개가 주어짐 (`{"a":1,"b":2}` vs `{"b":2,"a":1}`)
- When: 각각에 대해 `generateRecordHash()`를 호출함
- Then: 동일한 SHA-256 해시 값을 반환한다.

**Scenario 3: Merkle 해시 체인 무결성**
- Given: 5건의 순차적 감사 로그 레코드가 체인으로 연결되어 있음
- When: `verifyChain(records)`를 실행함
- Then: 체인 무결성 검증이 100% 통과하고, `{ valid: true, verifiedCount: 5 }`를 반환한다.

**Scenario 4: 체인 위변조 탐지**
- Given: 5건의 체인 중 3번째 레코드의 after_state가 임의로 변경됨
- When: `verifyChain(records)`를 실행함
- Then: `{ valid: false, brokenAt: 3, expectedHash: "...", actualHash: "..." }`를 반환하여 위변조 지점을 정확히 특정한다.

**Scenario 5: IntegrityEnvelope 생성**
- Given: Audit 리포트 데이터가 주어짐
- When: `createIntegrityEnvelope(reportData)`를 호출함
- Then: `hash_sha256`, `timestamp`(ISO 8601), `algorithm: "SHA-256"`이 포함된 봉투 객체가 반환된다.

**Scenario 6: Edge 전송 무결성 검증**
- Given: Edge에서 전송된 음성 데이터 바이너리와 함께 전송 시점의 SHA-256 해시가 수신됨
- When: `verifyTransmissionIntegrity(payload, expectedHash)`를 호출함
- Then: 해시 일치 시 `true`, 불일치 시 `false`를 반환하며 유실/위변조를 즉시 탐지한다.

## ⚙️ Technical & Non-Functional Constraints
- 해시 알고리즘: SHA-256 고정 (Node.js `crypto` 모듈 활용)
- 타임스탬프: ISO 8601 UTC 형식 고정 (`new Date().toISOString()`)
- 성능: 해시 생성은 동기 처리, 대용량 파일은 스트림 기반 비동기 처리
- 무결성: 해시 검증 통과율 100% (REQ-NF-011)
- 위변조 탐지: 체인 끊김 감지율 100% (REQ-NF-030)
- 호환성: Edge Runtime 및 Node.js Runtime 양쪽에서 동작 필수
- 테스트: 결정론적 해시 → 동일 입력 동일 출력 보장

## 🏁 Definition of Done (DoD)
- [ ] 모든 Acceptance Criteria를 충족하는가?
- [ ] IntegrityManager 모듈이 독립적으로 import 가능한가?
- [ ] SHA-256 해시, 타임스탬프, Merkle 체인 기능이 모두 구현되었는가?
- [ ] JSON 정규화(Canonical JSON)가 올바르게 동작하는가?
- [ ] 단위 테스트 커버리지가 핵심 함수 100%인가?
- [ ] ESLint / 정적 분석 경고가 없는가?
- [ ] API 문서(JSDoc)가 모든 public 함수에 작성되었는가?

## 🚧 Dependencies & Blockers
- **Depends on:** DB-010 (AUDIT_LOG 테이블 — hash_sha256, prev_hash 필드 참조)
- **Blocks:** INT-C-002 (Insert-only 적재 + Merkle 체인 구현), SA-C-002 (Audit 리포트 해시), SA-C-003 (리포트 해시 체인 기록), NC-C-003 (시정 비교 보고서 무결성), ZUI-C-003 (Edge→Cloud SHA-256 검증), LEAN-C-004 (진단 파라미터 감사 기록)
