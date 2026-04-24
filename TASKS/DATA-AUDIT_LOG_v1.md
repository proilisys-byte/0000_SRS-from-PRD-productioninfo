# 감사로그 데이터 구조 및 운영 원칙 (DATA-AUDIT_LOG) v1.0

## 1. 문서 목적
본 문서는 PRO ILI SMART 프로젝트의 데이터 무결성과 규제 준수(Compliance)를 보장하기 위한 핵심 축인 **감사로그(Audit Log)**의 논리적 구조와 운영 정책을 정의하는 상위 설계 문서이다. 이 문서는 시스템에서 발생하는 모든 주요 상태 변경과 접근 이력을 Insert-only 방식으로 보존하여 사후 추적성을 확보하는 것을 목적으로 한다. 본 문서는 향후 백엔드 미들웨어 개발, 관리자(ADM) 콘솔의 감사 로그 뷰어 설계, 보안 비기능 요구사항(NFR) 및 테스트(TEST) 시나리오 작성의 기준이 된다.

---

## 2. 감사로그 설계 원칙

### 표 1. 감사로그 설계 원칙
| 원칙 | 설명 | 적용 이유 |
| :--- | :--- | :--- |
| **Insert-only (추가 전용)** | 한 번 기록된 감사로그 레코드는 어떠한 경우에도 물리적 업데이트(UPDATE)나 삭제(DELETE)를 허용하지 않는다. | 로그 자체가 위변조될 수 있다면 감사 시스템의 존재 이유가 상실됨. |
| **원본 불변성** | 과거의 기록이 잘못되었다고 해서 원본 로그를 수정하지 않는다. | 조작 의심을 피하기 위함. |
| **정정은 추가 이벤트로 처리** | 오기입이나 보완이 필요한 경우, 기존 로그를 가리키는 새로운 "수정(Correction)" 이벤트를 추가로 발행한다. | 회계 장부의 취소표(적수) 발행 방식과 동일한 증명력 확보. |
| **멀티테넌시 맥락 보존** | 모든 로그는 반드시 발생한 고객사(Tenant)와 공장(Site)의 경계 정보를 포함해야 한다. | A 고객사의 감사 시 B 고객사의 로그가 노출되거나 섞이는 것을 원천 차단. |
| **최소 수집 (비식별화)** | 로그 메타데이터에 개인의 비밀번호, 생체 정보 원본, 암호화 키 등 민감 정보를 평문으로 직접 저장하지 않는다. | 감사로그 스토리지가 유출되더라도 2차 피해를 막고 PIPA 위반을 방지. |
| **관리자 이벤트 우선 추적** | 관리자의 권한 부여, 강제 승인, MFA 우회 등의 행위는 일반 유저의 로그보다 높은 중요도로 분류하고 별도로 감시한다. | 시스템의 가장 큰 리스크는 외부 공격보다 내부 관리자의 권한 오남용임. |
| **조회/적재 성능 분리 고려** | 쓰기(Insert) 속도가 서비스 성능을 저하시키지 않도록, 필요시 로그 적재는 비동기 이벤트 큐(또는 별도 파티션)로 처리한다. | 현장에서 동시다발적인 STT 캡처 발생 시 DB Lock 현상 방지. |

---

## 3. 감사 범위와 대상 이벤트

### 표 2. 감사 대상 이벤트 분류
| 이벤트 카테고리 | 이벤트 예시 | 기록 필요 이유 | 중요도 | Sprint 적용 시점 |
| :--- | :--- | :--- | :--- | :--- |
| **인증 이벤트** | 로그인 성공/실패, 세션 만료, 비정상 IP 접속 | 계정 탈취 및 Brute-force 공격 탐지 | High | Sprint 1 |
| **권한 이벤트** | Role 할당/변경, 권한 그룹 생성/삭제 | 권한 오남용 및 권한 승격(Escalation) 추적 | Critical | Sprint 1 |
| **관리자 이벤트** | 관리자 MFA 등록/해제, Tenant 설정 변경 | 시스템의 척추를 건드리는 행위에 대한 책임 소재 파악 | Critical | Sprint 1 |
| **데이터 변경 이벤트**| Bulk Import 실행, Audit Report 수정/반려 | 비즈니스 데이터의 위변조 방지 및 이력 추적 | Medium | Sprint 1 |
| **업로드 이벤트** | 기준정보 CSV 업로드, 이미지 첨부 | 원본 데이터의 출처 및 유입 시점 증빙 | Medium | Sprint 1 (CSV) |
| **Audit 생성 이벤트** | STT 입력 제출, Smart Audit 매핑 성공/실패 | 품질 리포트 자동 생성의 기원(Origin) 증명 | High | Sprint 1 |
| **동의/규제 이벤트** | PIPA 동의 완료, 동의 철회 | 개인정보 처리의 적법성 증빙 (분쟁 시 법적 근거) | Critical | Sprint 1 |
| **보안 이벤트** | 타 Tenant 데이터 접근 시도, Rate Limit 초과 | 악의적 공격 시도 탐지 및 즉각 대응 | Critical | Sprint 1 |
| **삭제/보존 이벤트** | 데이터 파기 요청 처리, 법적 보존 연장 | 개인정보 파기 의무 준수 확인 | High | Sprint 4 |

---

## 4. 감사로그 데이터 구조 개요

감사로그는 단순한 텍스트 줄(Line)이 아니라, "누가(Actor), 어디서(Context), 무엇을(Subject), 어떻게(Action) 했는가"를 정형화된 JSON 또는 관계형 필드로 쪼개어 저장하는 구조를 띤다.

### 표 3. 감사 엔티티 목록
| 엔티티명 | 설명 | Sprint 우선순위 | 관련 문서군 | 비고 |
| :--- | :--- | :--- | :--- | :--- |
| **AuditLog** | 모든 시스템 상태 변경 및 중요 조회를 기록하는 메인 원장(Ledger) 테이블 | Sprint 1 | `DATA-SCHEMA`, `API` | 파티셔닝 필수 |
| **SecurityEvent** | 단순 변경이 아닌 보안 위협(권한 실패, 타겟 침범 등)만 분리하여 쌓는 경고성 테이블 | Sprint 1 | `ADM`, `NFR` | 알람 발생 용도 |
| **ConsentAuditEvent** | 사용자 동의 이력과 파기 이력 등 PIPA 특화 규제 감사 테이블 | Sprint 1/4 | `UI`, `COM` | 법무팀 제출용 |
| **AuditHashChain** | 이전 로그의 해시값을 현재 로그에 포함시켜 사후 DB 조작 시 체인이 깨지도록 만드는 무결성 보강 테이블 | 확장 대기 (보류) | `NFR` | 옵션 (MVP 이후) |

---

## 5. 핵심 감사 엔티티 정의

### 표 4. AuditLog 핵심 필드 정의
| 필드명 | 타입 | 필수 | 설명 | 예시값 | 제약조건/비고 |
| :--- | :--- | :--- | :--- | :--- | :--- |
| `id` | UUID | Y | 로그 고유 식별자 | `123e4567-e89b...` | PK |
| `tenant_id` | UUID | Y | 발생한 고객사 식별자 | `tenant-uuid` | 데이터 격리용 (System 이벤트의 경우 Null 허용) |
| `site_id` | UUID | N | 발생한 공장/현장 식별자 | `site-uuid` | - |
| `actor_user_id`| UUID | Y | 행위자 식별자 | `user-uuid` | 시스템 자동 실행의 경우 `SYSTEM` 명시 |
| `actor_role` | String | Y | 행위 당시의 역할 | `TENANT_ADMIN` | 권한 강등 후 과거 로그 분석을 위해 당시 역할 스냅샷 저장 |
| `event_category`| Enum | Y | 대분류 (AUTH, DATA, ADMIN 등) | `DATA_CHANGE` | - |
| `event_type` | String | Y | 소분류 (행위의 구체적 명칭) | `AUDIT_REPORT_CREATED` | - |
| `target_type` | String | Y | 조작 대상 엔티티 명칭 | `AuditReport` | - |
| `target_id` | UUID | Y | 조작 대상 엔티티 식별자 | `report-uuid` | - |
| `action` | Enum | Y | 행위 유형 | `CREATE`, `UPDATE`, `READ`, `DELETE` | `READ`는 민감정보 열람 시에만 기록 |
| `status` | Enum | Y | 행위 결과 | `SUCCESS`, `FAILURE`, `DENIED` | 권한 없음(`DENIED`)은 중요 지표 |
| `occurred_at` | DateTime | Y | 발생 시각 | `2026-04-24T14:00:00Z`| DB 기본 타임스탬프 (불변) |
| `ip_address` | String | N | 접속 IP | `192.168.1.1` | - |
| `metadata` | JSONB | N | 변경 전/후 값이나 추가 컨텍스트 | `{"old_status": "DRAFT", "new_status": "APPROVED"}` | 비밀번호 등 평문 금지 |

---

## 6. 이벤트 분류 체계

### 표 5. 이벤트 분류 체계
| 카테고리 | 세부 이벤트 | 설명 | 감사 강도 | 관리자 검토 필요 여부 |
| :--- | :--- | :--- | :--- | :--- |
| **AUTH** | `LOGIN_SUCCESS`, `LOGIN_FAILED`, `MFA_VERIFIED` | 사용자 인증 관련 플로우 | 중간 | 실패 누적 시 자동 검토 |
| **RBAC** | `ROLE_GRANTED`, `ROLE_REVOKED` | 사용자 권한 및 역할 변경 | 매우 높음 | 즉시/정기 감사 필수 |
| **ADMIN** | `TENANT_CREATED`, `SYSTEM_SETTING_CHANGED` | 최상위 시스템 설정 변경 | 매우 높음 | 필수 |
| **DATA** | `REPORT_CREATED`, `BULK_IMPORTED` | 비즈니스 로직에 의한 데이터 적재 | 중간 | 분쟁 발생 시 조회용 |
| **COMPLIANCE**| `CONSENT_AGREED`, `CONSENT_WITHDRAWN` | PIPA 동의 및 철회 상태 | 높음 | 삭제 요청 발생 시 필수 |
| **SECURITY** | `UNAUTHORIZED_ACCESS`, `CROSS_TENANT_ATTACK`| 비정상 접근 및 인가 실패 | 매우 높음 | 즉각 알림 및 검토 필수 |

---

## 7. Insert-only 및 무결성 보강 방식

### 표 6. 무결성 보강 방식
| 방식 | 설명 | 장점 | 한계 | MVP 포함 여부 |
| :--- | :--- | :--- | :--- | :--- |
| **App-level Append Only** | 어플리케이션 코드(Prisma) 단에서 Audit 테이블에 대해 `create`만 허용하고 `update`, `delete` 함수 사용 금지. | 구현이 쉽고 직관적임. | DBA가 DB 툴로 직접 접속하여 쿼리(SQL)를 수정하는 것은 막지 못함. | **Y** |
| **DB-level RLS / Trigger**| Supabase(PostgreSQL)의 RLS에서 `INSERT`만 `true`로 열고, `UPDATE`, `DELETE`를 강제 블록함. | DB 접속 자체를 방어하여 물리적 변조 차단. | 스키마 마이그레이션 시 불편함. | **Y** (Sprint 1 필수) |
| **Hash Chaining** | 이전 로그 레코드의 해시값을 현재 로그의 `previous_hash` 필드에 포함하여 블록체인처럼 연결. | 중간 레코드 삭제/수정 시 전체 체인이 깨져 즉각 변조 적발 가능. | 구현 복잡도가 높고 동시 다발적 Insert 시 병목 발생(락 대기). | N (우선순위 낮음) |
| **External Archiving** | 매일 새벽 배치로 Audit 테이블을 Write-once 스토리지(예: AWS S3 Object Lock)로 백업. | 외부 침입으로 DB가 통째로 삭제되어도 증적 보존. | 외부 인프라 연동 비용. | N (Sprint 4 고려) |

*(결론: MVP 단계에서는 DB-level RLS 차단을 통해 무결성을 확보하고 Hash Chain은 과도한 복잡성을 야기하므로 제외한다.)*

---

## 8. 인증/권한/보안 이벤트 연계 방식

### 흐름 1. Sprint 1 핵심 감사 흐름 (일반 사용자)
1. **로그인**: `actor_user_id` 기준 `LOGIN_SUCCESS` 기록. (이벤트: AUTH)
2. **세션 발급**: JWT 토큰과 함께 세션 컨텍스트 유지.
3. **Bulk Import 실행**: Admin이 기준정보 업로드. `BULK_IMPORTED`와 함께 파일명, 성공 레코드 수를 `metadata`에 기록. (이벤트: DATA)
4. **STT 입력 제출**: `site_id` 컨텍스트 하에 음성 캡처. `STT_SESSION_CREATED` 기록. (이벤트: DATA)
5. **Audit Report 생성**: STT 매핑 결과를 바탕으로 리포트 생성. `REPORT_CREATED` 기록. (이벤트: DATA)

### 흐름 2. 관리자 통제 감사 흐름
1. **관리자 로그인**: `actor_role=TENANT_ADMIN` 인식.
2. **권한 변경 시도**: 특정 유저에게 `SITE_USER` 역할 부여 API 호출.
3. **MFA 개입 (옵션)**: 고위험 API로 지정된 경우 `aal2` (MFA 인증 상태) 재검증. 통과 실패 시 `MFA_FAILED` 및 `DENIED` 기록.
4. **성공 기록**: 통과 시 `ROLE_GRANTED` 이벤트를 `AuditLog`에 기록. 대상자의 ID는 `target_id`, 변경 권한은 `metadata`에 저장.

### 흐름 3. 규제/보안 감사 흐름
1. **비정상 접근 시도**: 유저 A(Tenant X 소속)가 Tenant Y의 API 엔드포인트 호출.
2. **미들웨어 차단**: 인증 가드가 이를 `403 Forbidden`으로 튕겨냄.
3. **보안 이벤트 연계**: 일반 `AuditLog` 외에 `SecurityEvent` 테이블에 `CROSS_TENANT_ATTACK`으로 기록하여 관리자 콘솔 대시보드에 즉시 붉은색 경고 노출.

---

## 9. 멀티테넌시 반영 방식

감사로그 테이블은 단일 거대 테이블이 되지만, RLS(Row Level Security)를 통해 **A 고객사의 관리자는 쿼리를 어떻게 날리더라도 A 고객사(`tenant_id = 'A'`)의 로그만 조회**할 수 있도록 통제된다. 
- 단, `SYSTEM_ADMIN` (PRO ILI 운영자)은 트러블슈팅 목적으로 모든 Tenant의 감사 로그 메타데이터를 볼 수 있으나, 민감 정보(STT 텍스트 등)는 마스킹 처리되어야 한다.

---

## 10. 보존 / 삭제 / 접근 정책

### 표 7. 보존 / 삭제 / 접근 정책
| 항목 | 정책 | 적용 대상 | 예외 | 리스크 |
| :--- | :--- | :--- | :--- | :--- |
| **감사로그 보존 기간** | 발생일로부터 최소 3년 보존 (기본 설정) | `AuditLog` 테이블 전체 | 법적 분쟁 중인 대상 건 (영구 보존) | 스토리지 비용 증가 |
| **감사로그 삭제 금지** | 보존 기간 내 어떤 사용자/관리자도 물리 삭제 불가 | 일반/Admin 모두 적용 | 시스템 보존 연한 만료 시 자동 파기 배치(Batch) | 용량 초과에 의한 DB 장애 |
| **개인정보 마스킹** | 로그의 `metadata` 필드에 들어가는 이메일, 전화번호 등은 마스킹 (예: `a***@test.com`) | `metadata` JSONB 내 식별 정보 | 식별 ID(UUID)는 마스킹 대상 아님 | 마스킹 누락 시 로그가 규제 대상이 됨 |
| **관리자 조회 권한** | Tenant Admin은 자기 고객사의 로그 전면 열람 가능 | `TENANT_ADMIN` 역할자 | Super Admin은 열람 시 별도 승인 프로세스 | 무단 조회 후 사찰 논란 |
| **다운로드(Export)** | 로그 CSV 다운로드 API 호출 시, 호출 자체를 다시 감사로그(`LOG_EXPORTED`)로 남김 | 콘솔 내 내보내기 기능 | 없음 | 대량 유출 |

---

## 11. 조회 / 인덱스 / 성능 고려사항

### 표 8. 조회 / 인덱스 고려사항
| 조회 패턴 | 필요한 필드 | 추천 인덱스 방향 | 비고 |
| :--- | :--- | :--- | :--- |
| **특정 고객사 전체 이력 조회** | `tenant_id`, `occurred_at` | `(tenant_id, occurred_at DESC)` 복합 인덱스 | 대시보드의 기본 쿼리 |
| **특정 사용자의 이상 행동 추적**| `tenant_id`, `actor_user_id` | `(tenant_id, actor_user_id, occurred_at)` | 퇴사자 문제 발생 시 추적 용도 |
| **특정 자원의 변경 이력** | `target_type`, `target_id` | `(target_type, target_id)` | Audit Report 1건의 생성~승인 히스토리 뷰어 |
| **보안 위협 신속 탐지** | `event_category`, `status` | `status = 'DENIED'` 인덱스 필터링 또는 부분 인덱스 | 빠른 알람 쿼리용 |

*(주의: AuditLog는 테이블 사이즈가 방대해지므로, 초기 설계부터 PostgreSQL Table Partitioning (월 단위 등)을 염두에 두어야 한다.)*

---

## 12. 운영 및 규제 대응 관점

- **PIPA 관점**: 사용자가 동의를 철회하여 데이터를 "삭제" 처리해야 할 경우, `AuditReport` 원본이나 `STTCapture` 데이터는 Hard Delete 또는 완벽한 비식별화 조치를 취한다. 그러나 **"해당 사용자가 데이터를 지우고 떠났다"라는 이력 자체(`DATA_DELETED` 이벤트)는 감사로그에 영구 보존**되어야 규제 기관에 파기 증명을 할 수 있다. (즉, 원본 데이터와 감사 로그의 수명을 분리한다.)

---

## 13. API / ADM / TEST / NFR 문서에 미치는 영향

- **API**: 비즈니스 로직 끝단에 무조건 `logger.audit()` 형태의 유틸리티가 강제 호출되도록 Middleware나 Interceptor 스펙을 잡아야 한다.
- **ADM**: 최고 관리자 콘솔(System Console)에 `SecurityEvent` 실시간 알람 패널과 `AuditLog` 필터링 뷰어가 설계되어야 한다.
- **TEST**: 테스트 시나리오는 단순히 기능이 동작하는지를 넘어, **기능 동작 직후 DB에 올바른 `AuditLog`가 1줄 생성되었는지**를 단언(Assert)해야 한다.
- **NFR**: 매달 쌓이는 감사로그의 양을 추산하여, Supabase 요금제나 DB 스토리지 증설 주기를 성능/운영 문서에 명시해야 한다.

---

## 14. 후속 문서 분해 기준

### 표 9. 후속 문서 연결 기준
| 영역 | 후속 문서 | 왜 필요한가 | 작성 우선순위 |
| :--- | :--- | :--- | :--- |
| **미들웨어 스펙** | `API-AUDIT_INTERCEPTOR_v1.md` | API 계층에서 비즈니스 로직과 감사로그 기록을 분리/자동화하는 코드 아키텍처 정의 | 매우 높음 |
| **관리자 UI** | `ADM-AUDIT_VIEWER_v1.md` | 관리자가 로그를 검색/필터링하고 엑스포트하는 UI 및 권한 정책 설계 | 중간 |
| **성능/보존 정책** | `NFR-DATA_RETENTION_v1.md` | 파티셔닝 전략과 AWS S3 등 외부 아카이빙 배치 설계 | 낮음 (MVP 이후) |

---

## 15. 다음 단계 작성 가이드

감사로그 및 데이터/인증 기반 설계가 모두 완료됨에 따라, 개발 조직은 아래와 같은 구체적 실행 명세 단계로 넘어가야 한다.

1. **가장 먼저 작성해야 할 후속 문서 Top 10**
   - `DATA-SUPABASE_RLS_v1.md` (Tenant 분리 및 Audit 테이블 수정 방지 쿼리)
   - `API-AUTH_MIDDLEWARE_v1.md` (RBAC 가드 및 인증 토큰 파싱)
   - `API-AUDIT_INTERCEPTOR_v1.md` (이벤트 로거 자동화 스펙)
   - `API-BULK_IMPORT_v1.md` (CSV 적재 프로세스 설계)
   - `UI-CONSENT_FLOW_v1.md` (PIPA 동의 화면 흐름)
   - `ADM-SECURITY_CONSOLE_v1.md` (보안/감사 모니터링 대시보드)
   - `F1-SMART_AUDIT_DATA_v1.md` (리포트 매핑 스키마 및 AI JSON 응답 구조)
   - `TEST-AUTH_RBAC_v1.md` (Tenant 침범 및 권한 에러 테스트)
   - `TEST-AUDIT_INTEGRITY_v1.md` (로그 위변조 방지 및 적재 확인 테스트)
   - `NFR-MFA_ENFORCEMENT_v1.md` (관리자 강제 MFA 시나리오)

2. **`ADM-*` 문서로 먼저 분해해야 할 영역**
   - 사용자/권한 관리 콘솔 (초대, 상태 변경).
   - 시스템 감사 뷰어 (조건 검색 및 CSV 내보내기).

3. **`TEST-*` 문서로 바로 이어져야 할 영역**
   - 관리자 권한 없이 AuditLog 테이블에 직접 Update/Delete를 시도했을 때 DB가 튕겨내는지 검증하는 단위 테스트.

4. **`NFR-*` 문서로 연결해야 할 영역**
   - 비정상 접근 시도가 초당 X회 이상 발생할 때의 알람 파이프라인.
   - 월별 데이터 파티셔닝 전략.

5. **Opus 4.6 (아키텍트) 추가 검토 포인트 3가지**
   - **Prisma Extension 기반 자동 로깅**: Prisma의 `.$extends` 미들웨어를 사용하여 개발자가 의식하지 않아도 모든 Model Mutation에 대해 자동으로 Audit 레코드를 삽입할 수 있는 추상화 레이어 설계.
   - **비동기 큐 (Async Queue) 처리**: API 응답 시간 지연을 막기 위해 Audit Log Insert를 Vercel Edge Functions의 `waitUntil`이나 Inngest/Upstash와 같은 비동기 큐로 밀어낼지 여부 결정.
   - **JSONB 검색 한계 극복**: 향후 관리자가 `metadata` 내부의 특정 값(예: 특정 변경 전 이메일 주소)으로 검색을 요구할 경우, JSONB 인덱스(GIN/GiST) 튜닝 방안.
