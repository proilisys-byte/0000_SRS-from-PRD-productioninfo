# 데이터 스키마 명세서 (DATA SCHEMA) v1.0

## 1. 문서 목적
본 문서는 PRO ILI SMART 프로젝트의 핵심 데이터 구조를 정의하는 상위 설계 문서다. 이 문서는 `06_TASK_DEPENDENCY_DIAGRAM_v1.md`에 명시된 선후행 관계 중 가장 첫 번째 선행 조건인 DB 설계를 구체화한다. 특히 멀티테넌시(Multi-tenancy), 역할 기반 접근 통제(RBAC), 데이터 위변조 방지(Insert-only Audit Log)를 보장하며, 향후 Prisma 및 Supabase(PostgreSQL) 상에서 물리적 스키마로 즉각 변환될 수 있는 논리적 기준을 제시한다.

---

## 2. 데이터 설계 원칙

### 표 1. 데이터 설계 원칙
| 원칙 | 설명 | 적용 이유 |
| :--- | :--- | :--- |
| **멀티테넌시 우선** | 모든 주요 엔티티는 `tenant_id` 및 `site_id`를 가져야 한다. | 고객사 간 데이터 격리 및 공장(Site)별 개별 권한 제어를 보장하기 위함. |
| **감사 가능성 (Auditability)** | 엔티티는 `created_by`, `created_at`을 필수로 가지며, 변경이 중요한 데이터는 이력을 남긴다. | "누가/언제/무엇을" 했는지 증명해야 하는 규제 준수(Compliance) 요건 달성. |
| **Insert-only 원칙** | 기존 데이터 업데이트(UPDATE)나 물리적 삭제(DELETE) 대신, 이력 버전 추가(APPEND) 방식 권장. | 데이터 위변조 방지 및 Soft-delete를 통한 데이터 복원/추적력 확보. |
| **최소 수집** | PIPA 규제 대상이 될 수 있는 개인정보(음성/얼굴)는 비식별화하거나 최소한만 수집/저장. | GDPR 및 국내 개인정보보호법에 기반한 법적 리스크 최소화. |
| **확장 가능한 상태 모델** | 상태값(Status)을 하드코딩하지 않고, 향후 워크플로우에 따라 유연하게 변할 수 있는 ENUM이나 상태 테이블로 관리. | Sprint 확장에 따라 NC 시정 등 복잡한 결재 프로세스 대응. |
| **Sprint 우선순위 기반 점진적 확장** | Sprint 1에 필요한 엔티티를 우선 확정하고, Vision이나 Lean 기능은 별도 확장 엔티티로 분리한다. | 개발 병목 최소화 및 초기 E2E 데모 달성 속도 향상. |

---

## 3. 데이터 범위 정의

### 표 2. Sprint별 데이터 범위
| Sprint | 포함 엔티티 | 목적 | 비고 |
| :--- | :--- | :--- | :--- |
| **Sprint 1** | Tenant, Site, User, Role, AuditLog, BulkImportJob, STTCaptureSession, AuditReport 등 | 인프라 셋업, Auth/RBAC, Audit Log, STT 코어 검증 | 핵심 베이스라인 (가장 우선 구현) |
| **Sprint 2** | VisionCapture, NCCase, CorrectiveActionPlan 등 | Vision AI 연동 및 NC 시정 워크플로우 처리 | 상태 머신(Status) 설계가 중요 |
| **Sprint 3** | COPQMetric, LeanDiagnosis, AIInferenceLog 등 | 통계형 데이터 적재 및 AI 모델 거버넌스 로깅 | 집계(Aggregation) 쿼리 중심 |

---

## 4. 핵심 엔티티 개요

### 표 3. 핵심 엔티티 목록
| 엔티티명 | 설명 | Sprint 우선순위 | 관련 기능군 | 비고 |
| :--- | :--- | :--- | :--- | :--- |
| **Tenant** | 고객사 (최상위 격리 단위) | Sprint 1 | 공통/기반 | UUID 사용 |
| **Site** | 고객사 내 공장/현장 단위 | Sprint 1 | 공통/기반 | Tenant의 하위 |
| **User** | 시스템 사용자 (Auth 계정) | Sprint 1 | 공통/기반 | Supabase Auth 연동 |
| **Role / Permission** | 권한 및 접근 제어 정책 | Sprint 1 | 공통/기반 | RBAC 구현 핵심 |
| **AuditLog** | Insert-only 시스템/데이터 감사 기록 | Sprint 1 | 감사/보안 | 위변조 불가 |
| **ConsentRecord** | PIPA 및 이용약관 동의 내역 | Sprint 1 | 감사/보안 | 락업 해제 조건 |
| **BulkImportJob** | 기준 데이터 대량 업로드 작업 이력 | Sprint 1 | 입력/업로드 | CSV 기반 |
| **STTCaptureSession** | 현장 음성 입력 세션 | Sprint 1 | Smart Audit | 원시 텍스트 |
| **AuditReport** | 스마트 생성된 최종 품질/감사 보고서 | Sprint 1 | Smart Audit | PDF 생성 타겟 |
| **NCCase** | 규정 위반 및 부적합 발생 건 | Sprint 2 | NC 확장 | Workflow 필요 |
| **COPQMetric** | 품질 실패 비용 지표 | Sprint 3 | Lean 확장 | 통계 산출용 |

---

## 5. 엔티티 관계 요약

### 표 4. 엔티티 관계 요약
| 부모 엔티티 | 자식 엔티티 | 관계 유형 | 설명 |
| :--- | :--- | :--- | :--- |
| **Tenant** | **Site** | 1:N | 하나의 고객사는 여러 공장을 가질 수 있음 |
| **Site** | **UserRoleAssignment** | 1:N | 특정 공장에 유저별 권한 부여 |
| **User** | **AuditLog** | 1:N | 유저가 발생시킨 행위 기록 |
| **STTCaptureSession** | **AuditReport** | 1:1 또는 1:N | 음성 세션 기반으로 리포트 생성 (추적 가능성) |
| **AuditReport** | **NCCase** | 1:N | 리포트의 지적 사항이 부적합 건으로 발전 |
| **BulkImportJob** | **BulkImportItem** | 1:N | 업로드 작업 1개당 파싱된 다수의 레코드 |

---

## 6. 엔티티 상세 정의

*(본 섹션은 주요 개념 수준의 필드를 나열하며, 물리적 FK 제약이나 DB 타입은 Prisma/Supabase 구현 시 최적화됨)*

### 6.1. 기반 / 권한 (Sprint 1)

**Tenant (고객사)**
| 필드명 | 타입 | 필수 | 설명 | 제약조건/비고 |
| :--- | :--- | :--- | :--- | :--- |
| `id` | UUID | Y | PK | - |
| `name` | String | Y | 고객사명 | - |
| `status` | Enum | Y | 활성/비활성 상태 | `ACTIVE`, `INACTIVE` |
| `created_at` | DateTime | Y | 생성 일시 | - |

**User (사용자)**
| 필드명 | 타입 | 필수 | 설명 | 제약조건/비고 |
| :--- | :--- | :--- | :--- | :--- |
| `id` | UUID | Y | PK (Supabase Auth ID 매핑) | - |
| `tenant_id` | UUID | Y | 소속 고객사 FK | - |
| `email` | String | Y | 로그인 이메일 | Unique |
| `name` | String | Y | 사용자 이름 | - |
| `is_mfa_enabled`| Boolean| Y | 관리자 등급용 MFA 활성화 여부 | - |

**Role (역할)**
| 필드명 | 타입 | 필수 | 설명 | 제약조건/비고 |
| :--- | :--- | :--- | :--- | :--- |
| `id` | UUID | Y | PK | - |
| `tenant_id` | UUID | Y | 커스텀 권한의 경우 FK | Nullable (System 권한일 경우) |
| `name` | String | Y | 권한명 (System Admin, Inspector 등)| - |
| `permissions` | JSONB | Y | 권한 정책 정의 (배열) | - |

### 6.2. 감사 / 보안 (Sprint 1)

**AuditLog (시스템 감사 로그)**
| 필드명 | 타입 | 필수 | 설명 | 제약조건/비고 |
| :--- | :--- | :--- | :--- | :--- |
| `id` | UUID | Y | PK | - |
| `tenant_id` | UUID | Y | 데이터 격리 FK | - |
| `actor_id` | UUID | Y | 행위자(User) FK | - |
| `action_type`| String | Y | 행위 종류 (CREATE, UPDATE 등) | - |
| `entity_type`| String | Y | 변경 대상 엔티티 | 예: `AuditReport` |
| `entity_id` | UUID | Y | 대상 레코드 ID | - |
| `old_payload`| JSONB | N | 변경 전 데이터 | - |
| `new_payload`| JSONB | N | 변경 후 데이터 | Insert-only 무결성 확보 |
| `created_at` | DateTime | Y | 발생 시각 | Immutable |

**ConsentRecord (PIPA 동의 내역)**
| 필드명 | 타입 | 필수 | 설명 | 제약조건/비고 |
| :--- | :--- | :--- | :--- | :--- |
| `id` | UUID | Y | PK | - |
| `user_id` | UUID | Y | 동의한 사용자 FK | - |
| `consent_type`| String | Y | 동의 유형 (음성 수집, 보관 등) | - |
| `is_agreed` | Boolean| Y | 동의 여부 | - |
| `agreed_at` | DateTime | Y | 동의 시각 | - |

### 6.3. 입력 / Smart Audit (Sprint 1)

**BulkImportJob (일괄 업로드 이력)**
| 필드명 | 타입 | 필수 | 설명 | 제약조건/비고 |
| :--- | :--- | :--- | :--- | :--- |
| `id` | UUID | Y | PK | - |
| `tenant_id` | UUID | Y | FK | - |
| `file_url` | String | Y | 원본 CSV 경로 | - |
| `status` | Enum | Y | PENDING, SUCCESS, FAILED | - |
| `created_by` | UUID | Y | 업로더 FK | - |

**STTCaptureSession (음성 수집 세션)**
| 필드명 | 타입 | 필수 | 설명 | 제약조건/비고 |
| :--- | :--- | :--- | :--- | :--- |
| `id` | UUID | Y | PK | - |
| `tenant_id` | UUID | Y | FK | - |
| `site_id` | UUID | Y | 공장/현장 FK | - |
| `raw_text` | Text | Y | 변환된 원시 텍스트 전체 | - |
| `duration_sec`| Int | Y | 녹음 시간 | - |
| `created_by` | UUID | Y | 녹음자 FK | - |

**AuditReport (생성된 감사 보고서)**
| 필드명 | 타입 | 필수 | 설명 | 제약조건/비고 |
| :--- | :--- | :--- | :--- | :--- |
| `id` | UUID | Y | PK | - |
| `tenant_id` | UUID | Y | FK | - |
| `site_id` | UUID | Y | FK | - |
| `session_id` | UUID | N | 생성 기원이 된 STT/Vision 세션 | 추적용 |
| `title` | String | Y | 리포트 제목 | - |
| `summary` | Text | N | AI가 요약한 내용 | - |
| `status` | Enum | Y | DRAFT, REVIEW, APPROVED | - |
| `created_at` | DateTime | Y | - | - |

### 6.4. 후속 확장 (Sprint 2 & 3)

**NCCase (부적합 시정 조치 - Sprint 2)**
| 필드명 | 타입 | 필수 | 설명 | 제약조건/비고 |
| :--- | :--- | :--- | :--- | :--- |
| `id` | UUID | Y | PK | - |
| `report_id` | UUID | Y | 발생 원인이 된 AuditReport FK| - |
| `severity` | Enum | Y | LOW, MEDIUM, HIGH, CRITICAL | - |
| `status` | Enum | Y | OPEN, IN_PROGRESS, RESOLVED | - |
| `due_date` | DateTime | Y | 조치 기한 | - |

**COPQMetric (품질 실패 비용 지표 - Sprint 3)**
| 필드명 | 타입 | 필수 | 설명 | 제약조건/비고 |
| :--- | :--- | :--- | :--- | :--- |
| `id` | UUID | Y | PK | - |
| `tenant_id` | UUID | Y | FK | - |
| `waste_type` | String | Y | 4대 낭비 카테고리 | - |
| `cost_value` | Decimal| Y | 환산된 비용 가치 | - |
| `measured_at`| DateTime | Y | 측정 기준일 | - |

---

## 7. 감사로그 및 보안 데이터 구조

### 표 6. 감사/보안 중요 엔티티
| 엔티티 | 감사 필요 이유 | 보존 정책 | 삭제 정책 | 민감도 |
| :--- | :--- | :--- | :--- | :--- |
| **AuditLog** | 모든 시스템 상태 변경의 법적 증거 보존 | 최소 3년 (고객사 정책) | 수동 삭제 불가 (보존 기한 도래 시 자동 파기) | 높음 |
| **ConsentRecord**| 개인정보 및 노조 합의 이력 증명 | 사용자 탈퇴 후 파기 | 법적 의무 보존 기간 준수 | 높음 |
| **STTCaptureSession**| 원시 음성 데이터의 출처 파악 및 매핑 증빙 | 리포트 생성 완료 후 30일 내 삭제 권장 | 비식별화 후 분석용으로 전환 또는 삭제 | 매우 높음 (목소리/실명 노출 가능) |

---

## 8. 멀티테넌시 / 권한 / 접근 통제 반영 방식

1. **Row Level Security (RLS)**: 
   Supabase의 RLS 정책을 통해 모든 테이블에 `tenant_id` 검증 로직을 SQL 레벨에서 삽입한다. 
   예: `auth.jwt() -> 'app_metadata' ->> 'tenant_id' = tenant_id`
2. **Access Control (RBAC)**: 
   `UserRoleAssignment`를 통해 `User` - `Site` - `Role`을 N:M으로 매핑한다. (예: A공장에서는 Inspector, B공장에서는 Read-only)
3. **MFA Event 연동**: 
   Admin 등급 유저가 `Tenant` 설정 변경이나 `AuditLog` 엑스포트 시, AuthSession의 `aal2` (MFA 인증 상태)를 필수 확인하고 AdminApprovalEvent를 남긴다.

---

## 9. 데이터 생명주기 및 보존/삭제 정책

- **Soft Delete**: `deleted_at` 필드를 통해 일반 데이터(Report, Item)는 논리적 삭제만 수행한다.
- **Hard Delete (개인정보)**: PIPA에 의거, `ConsentRecord` 철회 또는 보존 기한 만료 시 Batch Job(`CRON`)이 원시 음성/이미지(`UploadedFile`, `STTCaptureSession`)를 영구 삭제(Hard Delete)한다.
- **Insert-only 불변성**: `AuditLog`, `NCCase`의 진행 상태 변경 이벤트 등은 UPDATE가 아닌 Event 레코드를 추가하여 히스토리를 보장한다.

---

## 10. 인덱스 / 조회 최적화 고려사항

### 표 7. 인덱스 / 조회 최적화 고려사항
| 엔티티 | 주요 조회 패턴 | 추천 인덱스 방향 | 비고 |
| :--- | :--- | :--- | :--- |
| **AuditLog** | 특정 Tenant + 날짜 범위 이력 조회 | `(tenant_id, created_at)` 복합 인덱스 | 파티셔닝(Partitioning) 고려 |
| **AuditReport** | Site별 최근 생성 보고서 조회 | `(site_id, created_at DESC)` | - |
| **NCCase** | 마감일(Due Date) 임박 / 심각도(Severity) 조회 | `(tenant_id, status, due_date)` | 워크플로우 대시보드 속도 향상 |

---

## 11. 마이그레이션 및 시드 데이터 전략

### 표 8. 시드 데이터 전략
| 시드 대상 | 필요한 이유 | Sprint | 주의사항 |
| :--- | :--- | :--- | :--- |
| **System Roles** | System Admin, Auditor 등 기본 권한은 DB 초기화 시 고정 필요 | Sprint 1 | ID(UUID)값 고정 시 배포 스크립트 충돌 주의 |
| **Mock Template** | 초기 STT 매핑 테스트용 가상 ISO 심사 규정 템플릿 | Sprint 1 | 프로덕션 배포 시에는 고객사 Bulk Import로 대체 |
| **Demo Tenant** | Sprint 1 데모 시연을 위한 완벽히 격리된 테스트 조직 | Sprint 1 | 데모 전용 `tenant_id` 부여 및 시연 후 데이터 클렌징 |

---

## 12. API / 기능 문서에 미치는 영향

- 본 데이터 스키마는 **API I/O 명세**의 베이스가 된다.
- STT 파이프라인(AI API)은 반드시 `STTCaptureSession` 레코드를 먼저 생성한 후 해당 ID를 참조하여 텍스트 스트리밍을 저장해야 한다.
- Bulk Import 로직은 파싱 도중 에러가 나면 `BulkImportJob` 상태를 `FAILED`로 변경하고, 성공 건수만 롤백 없이 처리할지 All-or-Nothing으로 갈지 API 설계에서 트랜잭션을 결정해야 한다.

---

## 13. 후속 데이터 문서 분해 기준

### 표 9. 후속 문서 연결 기준
| 데이터 영역 | 후속 문서 | 왜 필요한가 | 작성 우선순위 |
| :--- | :--- | :--- | :--- |
| **Infra & Migration** | `DATA-SUPABASE_RLS_v1.md` | 논리 스키마를 Supabase 물리 계층의 RLS 및 정책 SQL로 변환하기 위함 | 매우 높음 (Sprint 1 시작 전) |
| **RBAC / Auth** | `COM-RBAC_MATRIX_v1.md` | Role 엔티티 안의 `permissions` JSONB 구조와 API 접근 제어 규칙 매핑 | 매우 높음 |
| **Smart Audit** | `F1-SMART_AUDIT_DATA_v1.md` | `STTCapture` → `AuditReport`로 이어지는 매핑 및 AI JSON Output 구조 설계 | 높음 |

---

## 14. 다음 단계 작성 가이드

데이터 스키마 정의가 완료되었으므로, 이를 기반으로 구체적인 실행 계획과 API/UI 명세로 넘어가야 한다.

1. **가장 먼저 작성해야 할 후속 문서 Top 10**
   - `DATA-SUPABASE_RLS_v1.md` (물리 DB 및 보안 계층 설계)
   - `COM-RBAC_MATRIX_v1.md` (권한 구조도)
   - `API-BULK_IMPORT_v1.md` (기준정보 적재 API)
   - `API-AI_PIPELINE_v1.md` (STT 및 Smart Audit 연동 스펙)
   - `UI-LAYOUT_ROUTING_v1.md` (공통 레이아웃 및 Auth Guard)
   - `F1-SMART_AUDIT_v1.md` (리포트 생성 및 매핑 로직)
   - `F2-NC_ACTION_v1.md` (시정 조치 플로우)
   - `NFR-COMPLIANCE_v1.md` (PIPA 및 동의 정책 상세)
   - `TEST-S1_DEMO_v1.md` (Sprint 1 검증 시나리오)
   - `ADM-SYSTEM_CONSOLE_v1.md` (관리자 모니터링 및 설정 UI)

2. **`API-*` 문서로 먼저 분해해야 할 영역**
   - 클라이언트와 백엔드가 분리되어 병렬 개발해야 하는 지점 (Bulk Import, STT 스트리밍, Report CRUD).

3. **`COM-*` 문서로 먼저 분해해야 할 영역**
   - 전역적으로 사용되는 인증(Auth), 권한(RBAC), 에러 핸들링, 로깅 파이프라인.

4. **`F1-*`, `ADM-*`, `TEST-*`로 이어져야 할 영역**
   - F1 (기능 로직): Smart Audit 등 핵심 비즈니스 로직.
   - ADM (관리자): 시스템 권한, 감사 로그 모니터링 화면.
   - TEST (테스트): 데이터 스키마를 무너뜨리지 않는 선에서의 E2E/통합 테스트 시나리오.

5. **Opus 4.6 (아키텍트) 추가 검토 포인트 3가지**
   - **Prisma Extension과 Supabase RLS 호환성**: `auth.uid()` 바인딩을 Prisma 미들웨어에서 어떻게 주입하여 RLS를 통과시킬 것인가.
   - **JSONB vs Relational 결합도**: `AuditReport`의 동적 필드를 `JSONB`로 처리할지, `AuditTemplateField` 형태의 정규화된 테이블로 유지할지에 대한 쿼리 퍼포먼스 비교.
   - **Insert-only Audit Log의 DB 용량 이슈**: 모든 이벤트를 무한정 쌓을 경우 발생하는 스토리지 비용 증가 및 파티셔닝(Table Partitioning) 적용 전략.
