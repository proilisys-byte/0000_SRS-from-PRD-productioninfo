# 작업 목록 (TASK LIST) v1.0

## 1. 문서 목적
본 문서는 PRO ILI SMART 프로젝트의 상위 기준 문서(PRD, SRS, Decision Log, Project Context)를 바탕으로, 실제 개발 조직이 착수할 수 있는 실행형 작업 목록(WBS)을 정의한다. 
본 문서는 전체 4개 Sprint의 작업 단위를 구체화하며, 각 작업의 선행 조건, 산출물, 완료 기준을 제시하여 병목 없는 개발 진행과 명확한 품질 검증을 유도한다. 이 문서는 향후 `06_TASK_DEPENDENCY_DIAGRAM_v1.md` 작성 및 기능/API/UI 상세 설계 문서 생성의 뼈대로 사용된다.

---

## 2. 작업 목록 작성 원칙
1. **실행 중심 분해**: 추상적인 목표(예: "AI 개발") 대신, 담당자가 즉시 설계를 시작할 수 있는 단위(예: "STT 입력 파이프라인 설계")로 분해한다.
2. **기반 공사 최우선**: 권한(Auth/RBAC), 무결성 기록(Insert-only Audit Log), 기본 데이터 적재(Bulk Import) 등 시스템의 척추가 되는 작업을 Sprint 1 상단에 배치한다.
3. **병렬성 극대화**: 프론트엔드(FE), 백엔드(BE), AI, 인프라 담당자가 상호 대기를 최소화하도록 선행/병렬 가능 요소를 명확히 분리한다.
4. **산출물 및 완료 기준 명확화**: 각 Task는 문서 완료, 설계 확정, 구현 완료, 검증 통과 등 객관적인 확인이 가능한 '완료 기준'을 지녀야 한다.

---

## 3. Sprint 운영 기준
- **Sprint 기간**: 각 Sprint는 2주 단위로 운영하며 총 4개 Sprint(8주)로 구성.
- **Sprint 1 원칙**: 시스템의 기반을 다지고, STT 음성 입력 및 Smart Audit 자동 매핑의 최소 E2E 시나리오를 완성해 작동 여부를 증명한다.
- **락업(Lock-up) 규정**: 보안/규제(PIPA 동의, 관리자 MFA, 노조 합의) 요건은 MVP 현장 배포 전 반드시 해결되어야 하는 락업 조건으로 작용한다.

---

## 4. 전체 Task 구성 요약

### 표 1. 전체 Task 구성 요약
| 구분 | 설명 | 포함 Sprint | 비고 |
| :--- | :--- | :--- | :--- |
| **기반 공사 및 코어** | Auth/RBAC, Insert-only DB, Bulk Import, STT 연동, Audit 엔진 | Sprint 1 | 인프라 및 코어 데이터 흐름 완성 |
| **확장 AI 및 NC** | Vision AI 이미지 판별 연동, NC 시정 조치 파이프라인 | Sprint 2 | 추가 데이터 모달리티 및 심화 비즈니스 로직 |
| **비용 가시화 및 거버넌스** | Lean 진단(COPQ) 4대 낭비 환산, ROI 대시보드, AI 버전/편향 관리 | Sprint 3 | 실제 누적 데이터 기반 로직 작동 |
| **최적화 및 규제 락업** | E2E 성능 부하/보안 테스트, PIPA 동의/MFA/노조합의 등 배포 락업 대응 | Sprint 4 | 상용화 및 실제 현장 투입을 위한 관문 |

---

## 5. Sprint별 Task 목록

### 표 2. Sprint별 목표
| Sprint | 핵심 목표 | 포함 기능군 | 제외 기능군 | 완료 판단 기준 |
| :--- | :--- | :--- | :--- | :--- |
| **Sprint 1** | 기반 공사 및 최소 시연 E2E (Core) | Auth, 2단계 RBAC, Insert-only Log, Bulk Import, STT Zero-UI, Smart Audit 코어 | Vision AI, NC 시정, Lean/COPQ 대시보드 | CSV 템플릿 업로드 → STT 데이터 적재 → 10분 내 PDF 리포트 생성 동작 시연 완료 |
| **Sprint 2** | NC 시정 파이프라인 및 Vision 확장 | 긴급 NC 시정 자동화 (초안 생성, 트래킹), Vision AI(이미지 판별) | Lean/COPQ 진단 | 통보된 NC 사유 기반 5분 내 시정 초안 생성, 이미지 객체 판별 텍스트화 동작 완료 |
| **Sprint 3** | Lean 진단 및 AI 거버넌스 도입 | COPQ 4대 낭비 환산 대시보드, ROI 추이, VMP 기반 모델 버전 관리 | 기존 ERP 연동, 다중 테넌트 | 실제 누적 데이터 기반 COPQ 대시보드 렌더링, 경영진 PDF 요약 리포트 출력 완료 |
| **Sprint 4** | 안정화 및 성능/규제 최적화 | 부하 테스트, 타임아웃 방어 최적화, 규제 락업 조건(동의 폼, MFA 등) 구현 | 신규 기능 개발 일체 불가 | p95 응답 속도 SLA 충족, 모의 해킹 방어, PIPA 동의 폼 현장 작동 및 락업 해소 완료 |

---

### 5.1 Sprint 1 Task 목록

**표 3-1. Sprint 1 Task 목록**
| Task ID | 작업명 | 작업 유형 | 관련 문서군 | 관련 기능군 | 우선순위 | 선행조건 | 병렬 가능 여부 | 담당 추천 | 핵심 산출물 | 완료 기준 | 비고 |
| :--- | :--- | :--- | :--- | :--- | :--- | :--- | :--- | :--- | :--- | :--- | :--- |
| **T1-001** | 프로젝트 인프라 및 DB 스키마 설계 | Infra/DB | `DATA-DB_SCHEMA_v1`, `DATA-SCHEMA_v1`, `DATA-011...`, `DATA-012...` | Auth, RBAC | P0 | 없음 | 불가 | S-5a (BE) | DB 스키마 명세서 | Supabase 초기 스키마 마이그레이션 적용 완료 | - |
| **T1-002** | Insert-only Audit Log 정책 적용 | DB/Sec | `DATA-AUDIT_LOG_v1` | Audit Log | P0 | T1-001 | 불가 | S-5a (BE) | DB RLS 정책 코드 | 변경/삭제 차단 룰 적용 및 테스트 통과 | 컴플라이언스 필수 |
| **T1-003** | Auth 및 2단계 RBAC 라우트 보호 구현 | API/UI | `COM-AUTH_v1`, `COM-RBAC_v1`, `COM-RH-002_auth...`, `UI-001_login_page` | Auth, RBAC | P0 | T1-001 | 가능 (UI) | S-5a (BE), S-5b (FE) | 권한 매트릭스 설계 | Admin/User 권한 접속 분리 및 인가 테스트 통과 | - |
| **T1-004** | 기준정보 Bulk Import 로직 설계 및 구현 | API | `COM-RH-001...`, `API-002_bulk...`, `ADM-C-001...`, `TEST-ADM-001...` | Bulk Import | P1 | T1-002 | 가능 (UI) | S-5a (BE) | 템플릿 CSV, 파서 코드 | 정규화된 템플릿의 CSV 데이터가 DB에 일괄 적재됨 | - |
| **T1-005** | Bulk Import UI 및 에러 피드백 화면 | UI | `UI-061_bulk_import_admin...`, `ADM-062...`, `API-001_common_error_schema` | Bulk Import | P1 | 없음 | 가능 (BE) | S-5b (FE) | 업로드 화면 UI | 업로드 성공/실패 모달 및 에러 내역 노출 | - |
| **T1-006** | Zero-UI STT 프롬프트 및 파이프라인 연동 | AI/API | `API-AI_PIPELINE_v1` | Zero-UI | P1 | 없음 | 가능 (FE) | S-5c (AI) | STT 매핑 프롬프트 | 음성 입력 시 구조화된 JSON 형태로 파싱 반환 | Gemini Multimodal |
| **T1-007** | 현장용 Zero-UI (음성 입력) 모바일 화면 구현 | UI | `UI-010_audit_workspace...`, `UI-011_audit_session...`, `UI-000_app_shell_layout` | Zero-UI | P1 | 없음 | 가능 (AI) | S-5b (FE) | 모바일 입력 화면 | 마이크 입력 활성화 및 클라우드 전송 테스트 통과 | - |
| **T1-008** | Smart Audit 템플릿 매핑 엔진 설계 | AI/API | `F1-RH-001_audit_route...`, `F1-C-001_audit_session...`, `TEST-F1-001...` | Smart Audit | P1 | T1-006 | 가능 (UI) | S-5c (AI), S-5a | 매핑 엔진 로직 | 저장된 데이터가 ISO 양식 JSON 스키마로 100% 매핑됨 | - |
| **T1-009** | Audit PDF 클라이언트 생성(html2pdf) 구현 | UI | `F1-C-002_audit_report...`, `API-003_audit_report_dto`, `TEST-F1-002...` | Smart Audit | P1 | T1-008 | 불가 | S-5b (FE) | PDF 다운로드 화면 | 매핑된 데이터를 브라우저에서 PDF로 렌더링 후 다운로드 완료 | 서버 부하 방지 |

---

### 5.2 Sprint 2 Task 목록

**표 3-2. Sprint 2 Task 목록**
| Task ID | 작업명 | 작업 유형 | 관련 문서군 | 관련 기능군 | 우선순위 | 선행조건 | 병렬 가능 여부 | 담당 추천 | 핵심 산출물 | 완료 기준 | 비고 |
| :--- | :--- | :--- | :--- | :--- | :--- | :--- | :--- | :--- | :--- | :--- | :--- |
| **T2-001** | Vision AI 프롬프트 파이프라인 설계 | AI/API | `T2-001_VISION_AI_SPEC` | Zero-UI | P1 | T1-006 | 가능 (FE) | S-5c (AI) | Vision 프롬프트 | 이미지 내 객체 인식 및 텍스트 묘사 JSON 반환 | Gemini Multimodal |
| **T2-002** | 현장용 카메라 촬영 및 전송 UI 구현 | UI | `UI-010_audit_workspace_page` | Zero-UI | P2 | 없음 | 가능 (AI) | S-5b (FE) | 촬영/업로드 UI | 브라우저 API 카메라 연동 및 이미지 클라우드 스토리지 적재 | 용량 제한 고려 |
| **T2-003** | NC 사유 파싱 및 초안 생성 로직 설계 | AI/API | `T2-003_NC_ACTION_SPEC` | NC 시정 | P1 | T1-008 | 가능 (UI) | S-5a, S-5c | 초안 생성 API | 텍스트 사유 입력 시 5분 내 시정 조치 초안 텍스트 반환 | - |
| **T2-004** | NC 시정 조치 진행률 트래킹 UI 구현 | UI | `UI-***` (TBD) | NC 시정 | P1 | 없음 | 가능 (BE) | S-5b (FE) | 상태 트래킹 보드 | 진행 상태 변경 및 백분율 차트 실시간 갱신 | - |
| **T2-005** | 시정 조치 전/후 무결성 비교 API 및 UI | API/UI | `API-***`, `UI-***` (TBD) | NC 시정 | P2 | T2-003 | 불가 | S-5a, S-5b | 비교 레포트 화면 | 변경 전/후 데이터 병렬 표기 및 해시값 노출 완료 | - |

---

### 5.3 Sprint 3 Task 목록

**표 3-3. Sprint 3 Task 목록**
| Task ID | 작업명 | 작업 유형 | 관련 문서군 | 관련 기능군 | 우선순위 | 선행조건 | 병렬 가능 여부 | 담당 추천 | 핵심 산출물 | 완료 기준 | 비고 |
| :--- | :--- | :--- | :--- | :--- | :--- | :--- | :--- | :--- | :--- | :--- | :--- |
| **T3-001** | COPQ 4대 낭비 환산 산식 및 쿼리 구현 | DB/API | `T3-001_ERP_INTEGRATION_v1` | Lean 진단 | P1 | T1-004 | 가능 (UI) | S-5a (BE) | 산식 적용 쿼리 | 7일 누적 데이터 기반 낭비 요소를 금전 가치로 집계 반환 | - |
| **T3-002** | Lean 진단 및 ROI 대시보드 UI 연동 | UI | `UI-***` (TBD) | Lean 진단 | P1 | 없음 | 가능 (BE) | S-5b (FE) | 대시보드 화면 | 환산된 데이터 기반 차트 렌더링 및 경고 배너 정상 작동 | - |
| **T3-003** | 경영진 요약 PDF 생성 로직 적용 | UI/API | `UI-***` (TBD) | Lean 진단 | P2 | T3-002 | 불가 | S-5b (FE) | 요약 보고서 | 대시보드 화면 요약본의 PDF 출력 기능 작동 | - |
| **T3-004** | AI Model Card 메타데이터 검증 도입 | AI | `ADM-***` (TBD) | AI 거버넌스 | P1 | T1-006 | 가능 | S-5c, S-5d | Model Card 명세 | 모델 버전/서명 검증 로직 배포 파이프라인 반영 | - |
| **T3-005** | 편향/Drift 경고 시스템 적용 | AI | `ADM-***` (TBD) | AI 거버넌스 | P2 | T3-004 | 불가 | S-5c (AI) | Drift 알림 로직 | 오차율 임계치 초과 시 알림 메일/웹훅 정상 발송 | - |
| **T3-006** | AI 매핑 XAI 설명가능성 UI (하이라이트) | UI/AI | `UI-***` (TBD) | Smart Audit | P2 | T1-008 | 불가 | S-5b, S-5c | XAI 뷰어 | 매핑 결과 원본 데이터 소스에 하이라이트 매칭 표시 | - |

---

### 5.4 Sprint 4 Task 목록

**표 3-4. Sprint 4 Task 목록**
| Task ID | 작업명 | 작업 유형 | 관련 문서군 | 관련 기능군 | 우선순위 | 선행조건 | 병렬 가능 여부 | 담당 추천 | 핵심 산출물 | 완료 기준 | 비고 |
| :--- | :--- | :--- | :--- | :--- | :--- | :--- | :--- | :--- | :--- | :--- | :--- |
| **T4-001** | 모니터링/SLI(가용성, 오류율 등) 세팅 | Infra | `NFR-MON-001`, `NFR-MON-002`, `NFR-MON-003`, `NFR-MON-004`, `ADM-063...` | 모니터링 | P1 | 시스템 통합 | 가능 | S-5d (Ops) | 대시보드 (Vercel) | 타임아웃, 4xx/5xx 에러율 실시간 지표화 완료 | - |
| **T4-002** | PIPA 5개 국어 동의 폼 강제 적용 로직 | UI/API | `NFR-COMPLIANCE_v1`, `UI-***` | 규제 락업 | P0 | 없음 | 가능 | S-5b, S-5a | 다국어 동의 UI | Edge 디바이스 최초 진입 시 동의 기록 DB 저장 필수화 | 법적 필수 |
| **T4-003** | 관리자 MFA 적용 로직 활성화 | API | `NFR-COMPLIANCE_v1` | 보안 | P0 | T1-003 | 가능 | S-5a (BE) | Auth 설정 갱신 | Admin 로그인 시 TOTP 등 2단계 인증 강제 동작 | - |
| **T4-004** | AI Streaming 타임아웃 방어 검증 | QA/API | `TEST-S1_ACCEPTANCE_v1`, `NFR-MON-003_smart_audit...` | 안정화 | P1 | T1-008 | 불가 | S-5d, S-5a | QA 보고서 | 60초 이상 대규모 매핑 시에도 Vercel 타임아웃 없이 스트리밍 성공 | Edge Runtime |
| **T4-005** | E2E 성능 부하/침투 테스트 수행 | QA | `TEST-S1_ACCEPTANCE_v1` | 보안/성능 | P1 | 기능 프리즈 | 불가 | S-5d (QA) | 취약점/성능 리포트 | p95 응답 속도 기준 도달, 발견 취약점 즉시 핫픽스 적용 | - |

---

## 6. 공통 선행 작업

### 표 4. 공통 선행 작업
| 작업 | 필요한 이유 | 영향을 받는 후속 작업 | 선행 확정 여부 |
| :--- | :--- | :--- | :--- |
| **Supabase 프로젝트 및 인프라 프로비저닝** | 데이터 모델 적용 및 Auth API 연동을 위한 클라우드 뼈대 필수 | 모든 API 및 UI(T1-001 ~ T4-005) | 확정 필수 |
| **Gemini API Key 획득 및 Vercel 등록** | STT, Vision, 매핑 등 핵심 AI 파이프라인 연동의 근간 | Zero-UI(T1-006, T2-001), Smart Audit(T1-008) | 확정 필수 |
| **피해 방지 행정 조치 (노조 합의)** | 불법 감시 논란에 따른 배포 락업(중단) 리스크 원천 차단 | Edge 배포 및 현장 검증 (Sprint 4) | 경영진 병렬 진행 요망 |

---

## 7. 병렬 가능 작업

### 표 5. 병렬 가능 작업
| 작업군 | 병렬 가능 이유 | 주의할 충돌 요소 | 담당 역할 |
| :--- | :--- | :--- | :--- |
| **DB 설계(BE) ↔ UI 껍데기(FE)** | 데이터 명세만 JSON으로 약속하면 화면 구성은 API 연동 전 목업으로 진행 가능. | DTO 스키마 명명 규칙 불일치 시 직렬화 에러 발생 | BE, FE |
| **AI 프롬프트(AI) ↔ 입력 폼 UI(FE)** | 입출력 포맷(Audio/Image → JSON)만 정의하면 내부 LLM 로직 개선과 UI 렌더링은 완전 독립적임. | AI 타임아웃 대비 UI 로딩/스트리밍 상태 처리 누락 주의 | AI, FE |
| **규제 폼 화면(FE) ↔ 인프라 모니터링(Ops)** | PIPA 동의 등 화면 작업은 기존 API를 단순 활용하며, 인프라 계측 로직과 시스템 의존성이 없음. | 모니터링 툴(Analytics) 스크립트 충돌 주의 | FE, Ops |

---

## 8. 리스크가 큰 작업

### 표 6. 리스크가 큰 작업
| 작업 | 리스크 내용 | 발생 가능성 | 영향도 | 완화 방안 |
| :--- | :--- | :--- | :--- | :--- |
| **Smart Audit 매핑 엔진 (T1-008)** | Vercel Serverless 한계(60초 타임아웃)로 대형 리포트 매핑 시 중단 가능. | 높음 | 매우 큼 (시연 실패) | Vercel AI SDK Edge Runtime + UI Streaming을 도입하여 연결 타임아웃 우회 적용. |
| **Insert-only Audit Log (T1-002)** | 잘못된 DB 트랜잭션으로 인해 로깅이 꼬이거나 속도 저하 발생. | 중간 | 큼 (무결성 훼손) | Prisma Middleware가 아닌 Supabase RLS 및 트리거 기반의 DB 레벨 제어를 최우선 고려. |
| **PIPA 동의 강제화 (T4-002)** | 법적 동의서 번역 지연 및 예외 시나리오 누락 시 서비스 오픈 불가. | 높음 | 매우 큼 (법적 블록) | 프로젝트 초기에 5개 국어 동의 폼 내용 및 레이아웃을 법무팀/경영진에 사전 승인 요청. |
| **PDF 클라이언트 렌더링 (T1-009)** | 브라우저 스펙에 따라 html2pdf 변환 시 폰트/레이아웃이 깨짐. | 높음 | 중간 (사용성 저하) | PDF 전용 단순화된 프린트용 CSS(media print) 분리 및 웹폰트 명확히 고정. |

---

## 9. 후속 문서 연결 기준

### 표 7. 후속 문서 연결 기준
| Task ID 또는 작업군 | 관련 완료 문서 (대응) | 왜 필요한가 | 현황 |
| :--- | :--- | :--- | :--- |
| **T1 전체 흐름** | `06_TASK_DEPENDENCY_DIAGRAM_v2.md` | 어떤 작업이 먼저 끝나야 다음 사람이 작업할 수 있는지 병목 해소를 위한 시각화 필수 | **완료 (v2 반영)** |
| **T1-001, T1-002** | `DATA-DB_SCHEMA_v1.md`, `DATA-SCHEMA_v1.md`, `DATA-AUDIT_LOG_v1.md` | Insert-only, RBAC 등 무결성을 강제하는 테이블과 제약조건(Constraints)의 물리적 정의 필요 | **완료** |
| **T1-004, T1-006, T1-008** | `COM-RH-001...`, `F1-RH-001...`, `API-002...`, `API-003...`, `API-AI_PIPELINE_v1.md` | Server Actions/Route Handlers의 I/O 명세(Zod Schema)와 에러 처리 구조 고정 | **완료** |
| **T1-005, T1-007, T1-009** | `UI-000...`, `UI-010...`, `UI-061...`, `F1-C-002...` | shadcn/ui 기반 토큰 사용, Zero-UI 음성 마이크 상태 관리 상태도 명세 | **완료** |
| **T4 작업군** | `NFR-COMPLIANCE_v1.md`, `NFR-MON-001~004.md` | 모의 해킹, 동의 폼 락업, MFA 등 운영 시 배포 중단 요건 명문화 | **완료** |

---

## 10. 다음 문서 작성 가이드

개발 파트 리더 및 담당자는 이 `01_TASK_LIST_v1.md` 문서를 기반으로 즉각적인 착수를 위해 다음 문서들을 분해/작성해야 합니다.

1. **`06_TASK_DEPENDENCY_DIAGRAM_v2.md` 참조 핵심 (작성 완료):**
   - **Critical Path 표시**: `[T1-001(DB세팅)] → [T1-002(Log적용)] → [T1-004(Bulk Import)] → [T1-006(STT연동)] → [T1-008(매핑)]` 로 이어지는 Sprint 1 시연용 핵심 척추가 시각화되어 있음.
   - **블로커 명시**: PIPA 동의(T4-002)와 관리자 MFA(T4-003) 노드를 'MVP 최종 배포(Release)' 앞을 가로막는 절대 게이트로 관리 중.

2. **최우선 참조 세부 문서 (Top 3):**
   - `DATA-DB_SCHEMA_v1.md` 및 `DATA-SCHEMA_v1.md` (T1-001 대응): Supabase 테이블 명세.
   - `COM-RH-001_bulk_import_route_handler.md` (T1-004 대응): CSV 업로드용 라우트 핸들러 및 DTO 규격 정의서.
   - `API-AI_PIPELINE_v1.md` 및 `F1-RH-001_audit_route_handler.md` (T1-006, T1-008 대응): Gemini 프롬프트 버저닝 및 입출력 JSON 형태 명세.

3. **세부 문서 분해 시 주의사항:**
   - API 문서는 전통적 REST 방식이 아닌, **Next.js Server Actions(함수 시그니처, Zod 검증)와 Route Handlers 기반**으로만 작성한다.
   - UI 문서는 커스텀 CSS 사용을 금지하고, Tailwind CSS + shadcn/ui 컴포넌트 조합 원칙을 강제한다.
   - DATA 문서는 로컬 SQLite 개발 ↔ 프로덕션 Supabase(PostgreSQL) 호환을 위한 `Prisma Client Extensions` JSON 대응 전략을 반드시 포함한다.
