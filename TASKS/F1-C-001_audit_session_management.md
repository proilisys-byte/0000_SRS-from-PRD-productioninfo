# [F1-C-001] Audit Session Management 처리 명세서

## 1. 문서 목적
이 문서는 PRO ILI SMART 프로젝트의 핵심 도메인인 F1 Smart Audit 영역에서, 실제 현장 감사 행위의 실행 단위가 되는 **Audit Session(감사 세션)**의 생명주기 및 처리 규칙을 표준화하는 공식 Command 명세서이다.

- **문서 필요성**: Smart Audit은 단순한 폼 입력이 아니라 오프라인 감사 현장의 흐름을 시스템으로 옮긴 것이다. 따라서 상태 통제, 권한 제어, 임시저장, STT 연계 등이 엄격하게 정의되어야 한다.
- **Sprint 1 중요성**: Sprint 1의 핵심 목표는 "감사 세션이 생성되어 현장에서 기록되고 제출되는 최소 폐쇄 루프"의 완성이다.
- **후속 문서와의 관계**: 본 문서는 데이터의 입력과 상태 변화 로직(Command)을 다루며, 이는 추후 리포트 생성(Generation), 조회(Query), UI 페이지 및 인수 테스트 문서의 상위 기준(Ground Truth)이 된다.

---

## 2. Audit Session 처리 설계 원칙

### 표 1. Audit Session 처리 설계 원칙
| 원칙 | 설명 | 적용 이유 | 관련 문서 | Sprint 우선순위 |
| :--- | :--- | :--- | :--- | :--- |
| **현장 실무성** | 오프라인 감사의 흐름(중단, 재개, 항목별 부분 저장)을 시스템이 지원해야 함 | 통신이 끊기거나 감사가 지연되는 현장 상황 반영 | 00_PRD_v1.md | Sprint 1 (필수) |
| **상태 기반 통제** | 세션의 상태(Draft, InProgress, Submitted 등)에 따라 허용되는 액션과 권한을 엄격히 통제 | 제출된 데이터가 임의로 변경되는 것을 방지하여 무결성 확보 | F1-AUDIT_v1.md | Sprint 1 (필수) |
| **최소 권한** | 세션 조작은 할당된 감사자, 검토자, 그리고 해당 Site의 관리자만 가능 | 권한 없는 사용자의 데이터 오염 및 조작 방지 | COM-AUTH_v1.md | Sprint 1 (필수) |
| **멀티테넌시 보호** | 모든 세션은 Tenant ID 및 Site ID에 강하게 바인딩되어야 함 | B2B SaaS로서 데이터 격리는 타협 불가능한 최우선 원칙 | DATA-SCHEMA_v1.md | Sprint 1 (필수) |
| **감사 가능성** | 세션의 상태 변경, 제출, 취소 등 중요 액션은 Audit Log에 기록됨 | 누가 언제 데이터를 확정하고 조작했는지 추적 가능해야 함 | DATA-AUDIT_LOG_v1.md | Sprint 1 (필수) |
| **STT 보조 원칙** | STT Zero-UI는 입력을 가속하는 보조 수단이며, 자동 판정이나 최종 통제권을 가지지 않음 | AI 환각을 방지하고, 최종 결정 책임은 감사자(Human)에게 부여 | 08_DECISION_LOG_v1.md | Sprint 1 (필수) |
| **임시저장 허용** | 항목별로 입력된 내역은 최종 제출 전 언제든 임시저장(PartiallySaved) 가능해야 함 | 긴 시간 진행되는 감사의 데이터 유실 방지 | F1-AUDIT_v1.md | Sprint 1 (필수) |

---

## 3. 적용 범위 정의

### 표 2. 적용 범위 정의
| 세션 관리 영역 | 설명 | Sprint 적용 시점 | 필수 여부 | 비고 |
| :--- | :--- | :--- | :--- | :--- |
| **세션 기본 생명주기** | 생성, 시작, 임시저장, 제출, 종료 등 기본 상태 전이 | Sprint 1 | Y | Core 기능 |
| **STT 텍스트 보조 입력** | 현장 음성을 텍스트로 변환하여 폼의 특정 필드에 채워넣는 기능 | Sprint 1 | Y | STT Zero-UI 범위 |
| **권한 기반 액션 제어** | Role/Site에 따른 세션 수정/제출 권한 통제 | Sprint 1 | Y | 보안 필수 |
| **Vision AI 자동 판정** | 사진 업로드 시 AI가 양불 판정 및 텍스트 자동 추출 | Sprint 2 | N | Sprint 1 Out of Scope |
| **NC 시정 조치 연계** | 세션 제출 시 부적합 항목에 대한 NC 티켓 자동 생성 | Sprint 2 | N | Sprint 1 Out of Scope |

---

## 4. Audit Session 개념 정의

- **한 줄 정의**: 지정된 `Audit Template`을 바탕으로, 특정 `Site`에서 할당된 `사용자(감사자)`가 실제 감사를 수행하고 데이터를 기록하는 하나의 **실행 인스턴스**.

### 표 3. Audit Session 개념 및 관계 정의
| 구성 요소 | 설명 | 관련 엔티티/문서 | 역할 | 비고 |
| :--- | :--- | :--- | :--- | :--- |
| **Audit Session** | 감사 행위 자체를 담는 상태 저장소 (실행 단위) | Session Table | 진행 상태 및 입력 데이터 홀딩 | 리포트 생성 전 단계 |
| **Audit Template** | 어떤 항목을 감사할지 정의한 설계도 | Template Table | 세션의 뼈대 제공 | 버전 관리 필요 |
| **Audit Report** | 세션이 완료(Finalized)된 후 산출되는 최종 결과 문서 | Report Table | 감사의 최종 결론 및 공유 포맷 | Session 데이터 기반으로 렌더링 |
| **Audit Log** | 시스템 내에서 발생한 사용자 행위 이력 | AuditLog Table | 보안 및 이력 추적용 | Session의 상태 전이가 이곳에 기록됨 |

---

## 5. 세션 생성 기준

### 표 4. 세션 생성 기준
| 항목 | 설명 | 필수 여부 | 생성 시 검증 포인트 | 비고 |
| :--- | :--- | :--- | :--- | :--- |
| **생성 주체** | Site Admin 이상의 권한자 또는 해당 감사를 수행할 Auditor | Y | `tenant_id`, `site_id`와 User Role 교차 검증 | 권한 없는 자 생성 차단 |
| **초기 입력 정보** | 대상 Template ID, 감사 대상 정보(라인/공정 등), 예정일, 감사자 ID | Y | 유효한 Template인지 (Active 상태) 확인 | 삭제된 Template 사용 불가 |
| **바인딩 기준** | 세션 row 생성 시 `tenant_id`, `site_id`를 필수 삽입 | Y | JWT Token의 클레임 값과 일치하는지 비교 | 멀티테넌시 격리 |
| **초기 상태** | 세션 생성 직후의 상태는 `Draft` 또는 `Ready` | Y | 필수 정보 입력 완료 시 `Ready` 전이 | |
| **중복 방지** | 동일 감사자/동일 대상/동일 시간에 이미 진행 중인 세션 여부 체크 | N(권장) | `InProgress` 상태의 동일 속성 세션 확인 | Sprint 1에서는 경고 수준 처리 |

---

## 6. 세션 실행 흐름

### 표 5. 세션 실행 흐름
| 단계 | 설명 | 입력 | 처리 | 출력 | 실패 시 대응 | Audit Log |
| :--- | :--- | :--- | :--- | :--- | :--- | :--- |
| **1. 생성 (Create)** | 새로운 감사를 위한 껍데기 세션 생성 | Template ID, 대상 정보 | 유효성/권한 검증, DB Insert | Session ID 반환 (상태: Draft/Ready) | 400/403 에러 리턴 | Y |
| **2. 시작 (Start)** | 현장에서 감사자가 감사를 개시함 | Session ID | 상태를 `InProgress`로 업데이트 | 상태 갱신 성공 응답 | 404 (세션없음) | Y |
| **3. 데이터 입력** | 항목별 점수, 텍스트, 이미지 등을 입력 | Item ID, Value | Payload 유효성 검사 (부분 유효성) | 임시 메모리 또는 로컬 DB 반영 | 사용자 화면 에러 표시 | N (입력 단위는 생략) |
| **4. 임시저장** | 서버에 현재까지의 입력값을 안전하게 저장 | Session ID, Data | DB Upsert, 상태 `PartiallySaved` | 저장 완료 응답 | 500 에러 및 로컬 백업 | Y |
| **5. 제출 (Submit)** | 감사를 마치고 검토자에게 데이터 이관 | Session ID, Data | 전체 필수항목 검증, 상태 `Submitted` | 제출 성공 응답 | 400 (누락 항목 알림) | Y |
| **6. 종료/확정** | 리뷰어가 검토 완료 후 세션 데이터 락업 | Session ID | 상태 `Finalized` 전환, 리포트 생성 큐 이동 | 리포트 생성 Event 발행 | 403 (권한 없음) | Y |

---

## 7. 상태 정의

### 표 6. 상태 정의
| 상태 (State) | 의미 | 진입 조건 | 종료 조건 | 허용 액션 | 비고 |
| :--- | :--- | :--- | :--- | :--- | :--- |
| **Draft** | 기본 정보만 입력되고 항목 준비가 안 된 임시 상태 | 세션 최초 생성 시 | 필수 메타데이터 입력 완료 | 삭제, 메타데이터 수정 | |
| **Ready** | 감사를 시작할 준비가 완료된 상태 | Draft에서 메타정보 완성 시 | 현장에서 Start 액션 트리거 | 시작, 정보 수정, 취소 | |
| **InProgress** | 감사가 현재 활발히 진행 중인 상태 | Ready 상태에서 Start 시 | 제출 또는 임시저장 트리거 | 항목 입력, STT 호출, 임시저장 | 모바일 앱에서 주로 활성화 |
| **PartiallySaved** | 진행 중 일시 중단되어 서버에 임시저장된 상태 | InProgress 중 Save 액션 시 | 다시 Start 하거나 Submit 트리거 | 재개, 수정, 계속 진행 | 통신 불안정 환경 대비 |
| **Submitted** | 감사자가 작성을 마치고 제출을 완료한 상태 | 모든 필수 항목 입력 후 Submit | 리뷰어가 승인(Finalize) 또는 반려 | 조회, 리뷰(권한자 한정) | 감사자(Auditor)는 수정 불가 |
| **ReviewPending** | (선택) 리뷰어의 검토를 명시적으로 기다리는 상태 | Submitted 직후 (승인 프로세스 On) | 승인, 반려 트리거 | 리뷰, 반려 코멘트 작성 | Sprint 1에서는 Submitted와 병합 고려 |
| **Finalized** | 세션 데이터가 확정되어 더 이상 변경 불가능한 상태 | 리뷰 완료/승인 시 | 없음 (최종 상태) | 조회, 리포트 생성, 보관 | **Read Only 락업** |
| **Cancelled** | 감사가 모종의 이유로 중간에 취소/무효화된 상태 | 제출 이전 상태에서 Cancel 시 | 없음 (최종 상태) | 조회 | 삭제하지 않고 로그로 남김 |
| **Reopened** | 제출/확정된 세션을 예외적으로 다시 연 상태 | Submitted/Finalized에서 Reject/Reopen 시 | 다시 Submit 트리거 | (InProgress와 동일 액션) | 관리자/리뷰어 권한 필수 |

---

## 8. 상태 전이 규칙

### 표 7. 상태 전이 규칙
| 현재 상태 | 수행 액션 | 다음 상태 | 자동/수동 여부 | 제약 조건 | 비고 |
| :--- | :--- | :--- | :--- | :--- | :--- |
| `Ready` | `startSession` | `InProgress` | 수동 (API) | 세션 소유자/할당자만 가능 | 시작 시간(startTime) 기록 |
| `InProgress` | `saveProgress` | `PartiallySaved` | 수동/자동 | 없음 (부분 데이터도 허용) | 주기적 Auto-save 구현 권장 |
| `PartiallySaved`| `resumeSession`| `InProgress` | 수동 (API) | 세션 소유자만 가능 | |
| `InProgress`<br>`PartiallySaved` | `submitSession` | `Submitted` | 수동 (API) | **모든 필수 입력 항목이 채워져야 함** | 제출 시간(submitTime) 기록 |
| `Submitted` | `finalizeSession`| `Finalized` | 수동 (API) | Site Admin 또는 Reviewer 권한 필요 | 이후 수정 불가 (Lock-up) |
| `Submitted` | `rejectSession` | `Reopened` | 수동 (API) | Site Admin 또는 Reviewer 권한 필요 | 반려 사유(Reason) 필수 입력 |
| `Draft`, `Ready`<br>`InProgress` | `cancelSession` | `Cancelled` | 수동 (API) | 감사자 또는 Admin | 이미 Submitted 이후는 취소 불가 |

---

## 9. 입력 처리 기준

### 표 8. 입력 처리 기준
| 입력 유형 | 설명 | 처리 방식 | 검증 포인트 | 오류 가능성 | 비고 |
| :--- | :--- | :--- | :--- | :--- | :--- |
| **수동 입력 (Text/Num)**| 사용자가 직접 폼에 타이핑/선택 | 즉시 Local State 반영 후 Save 시 API 전송 | Type, Length, Max/Min 검증 | 타입 불일치 (400 Bad Request) | 기본 동작 |
| **STT 보조 입력** | 마이크 아이콘 클릭 후 음성 발화 | 서버측 Whisper API 거쳐 Text로 변환하여 Input 필드에 주입 | 변환된 텍스트의 유효성 (길이 등) | 백그라운드 소음으로 인한 오인식 | (하단 표 9 참조) |
| **사진 첨부** | 감사 근거를 위한 이미지 업로드 | Object Storage (S3/Supabase) 업로드 후 URL을 Session Payload에 포함 | 파일 크기, 확장자, 악성코드 | 업로드 용량 초과 | |
| **누락 항목 처리** | 필수 항목(Required)을 비워둠 | 임시저장(Save)은 허용, 제출(Submit)은 차단 | Template Schema와 Payload 대조 | 400 (Missing Required Fields) | |

---

## 10. STT 연계 처리 기준

**원칙**: STT는 **"키보드를 대체하는 텍스트 입력 수단"**이다. 감사의 판단 주체는 인간이다.

### 표 9. STT 연계 처리 기준
| 항목 | 설명 | 처리 규칙 | 수동 보정 가능 여부 | 기록 필요 | 비고 |
| :--- | :--- | :--- | :--- | :--- | :--- |
| **활성화 단계** | 언제 STT를 쓸 수 있는가? | `InProgress` 및 `PartiallySaved` 상태의 텍스트 입력 필드에서만 | - | N | |
| **데이터 반영** | STT 결과를 어떻게 넣는가? | STT 응답 텍스트를 대상 Input 필드의 Value에 Append 또는 Replace | 가능 (키보드로 즉시 수정 가능해야 함) | N | UI 레이어에서 처리 |
| **오인식/누락** | 인식이 잘못된 경우 | 별도 에러 처리 없이, 변환된 텍스트를 보여주고 사용자가 스스로 수정 | 사용자 책임 하에 수동 수정 (필수) | N | AI 환각 방지 장치 |
| **이력 관리** | STT 사용 여부를 남기는가? | 해당 필드의 값이 STT를 거쳤는지 여부를 Payload 메타데이터에 boolean 플래그로 남김 | - | Y | 추후 AI 사용성 통계 산출용 |

---

## 11. 권한 및 범위 제한

**원칙**: `tenant_id`와 `site_id`를 벗어난 조작은 시스템 레벨에서 원천 차단된다 (Row Level Security 및 Application 레벨 미들웨어 검증).

### 표 10. 권한 및 범위 제한 매트릭스
| 역할 (Role) | 세션 생성 | 진행/임시저장 | 제출 (Submit) | 취소 (Cancel) | 승인 (Finalize) | 재오픈 (Reopen) | 비고 |
| :--- | :---: | :---: | :---: | :---: | :---: | :---: | :--- |
| **System Admin** | Y | Y | Y | Y | Y | Y | 슈퍼 유저 (디버깅 목적 외 사용 자제) |
| **Tenant Admin** | Y | Y | Y | Y | Y | Y | 소속 Tenant 내 모든 Site 통제 |
| **Site Admin** | Y | Y | Y | Y | Y | Y | 소속 Site 내 세션만 조작 가능 |
| **Auditor (할당됨)** | Y | Y | Y | Y (제출 전) | N | N | 자신이 할당된 세션만 접근/수정 가능 |
| **Auditor (미할당)** | N | N | N | N | N | N | 타인의 세션 조작 불가 (View Only 옵션 별도) |
| **Guest / None** | N | N | N | N | N | N | 접근 불가 (403 Forbidden) |

> [!WARNING]
> Sprint 1에서는 승인 프로세스(ReviewPending)를 간소화하여, 제출(`Submitted`) 즉시 확정(`Finalized`) 처리하거나 Site Admin이 일괄 승인하는 단순 Flow를 적용할 수 있다. 이는 요구사항 조율 결과에 따른다.

---

## 12. 오류 및 예외 처리

`API-001_common_error_schema.md` 규격을 준수하여 예외를 반환한다.

### 표 11. 오류 및 예외 처리 기준
| 오류 유형 | 설명 | 발생 시점 | 사용자 대응 (UI) | 운영자 대응 | 에러 스키마 연계 |
| :--- | :--- | :--- | :--- | :--- | :--- |
| **Template 없음** | 존재하지 않거나 비활성화된 템플릿으로 생성 시도 | Create | "유효하지 않은 템플릿입니다." 알림 | Template DB 정합성 확인 | 404 Not Found (code: `TEMPLATE_NOT_FOUND`) |
| **필수값 누락** | 빈 값이 있는 상태로 Submit 호출 | Submit | 누락된 필드를 붉은색으로 하이라이트 | - | 400 Bad Request (code: `VALIDATION_ERROR`) |
| **권한/Site 오류** | 타 Site 데이터 접근 또는 권한 밖 액션 | All | "접근 권한이 없습니다." 알림 | JWT 탈취 또는 비정상 접근 로깅 | 403 Forbidden (code: `ACCESS_DENIED`) |
| **상태 전이 위반** | Submitted 상태의 세션에 Save/Submit을 재호출 | Save/Submit | "이미 제출된 세션은 수정할 수 없습니다." | - | 409 Conflict (code: `INVALID_STATE_TRANSITION`) |
| **STT 변환 실패** | STT API 타임아웃 또는 음성 인식 불가 | InProgress | "음성을 텍스트로 변환할 수 없습니다. 직접 입력해주세요." | 네트워크/외부 API 헬스 체크 | UI 단독 처리 (Toast 메시지) |

---

## 13. 감사 로그 및 추적 연계

**원칙**: 세션의 주요 상태 변화는 데이터베이스 조작 이력으로 `AuditLog` 테이블에 반드시 적재되어야 한다. (Insert-Only 원칙)

### 표 12. 감사 로그 및 추적 연계 기준
| 이벤트 (Action) | 설명 | 기록 필드 (Payload) | 중요도 | 관련 문서 |
| :--- | :--- | :--- | :--- | :--- |
| `SESSION_CREATED` | 신규 세션 생성 | `session_id`, `template_id`, `actor_id` | High | DATA-AUDIT_LOG_v1.md |
| `SESSION_STARTED` | 세션 `InProgress` 전이 | `session_id`, `start_time` | Medium | - |
| `SESSION_SAVED` | 임시저장 수행 | `session_id`, (데이터 스냅샷 해시값 권장) | Low | (단순 업데이트는 로깅 빈도 조절 필요) |
| `SESSION_SUBMITTED`| 세션 제출 | `session_id`, `submitter_id` | High | 무결성 검증의 기준점 |
| `SESSION_FINALIZED`| 세션 락업 확정 | `session_id`, `reviewer_id` | Critical | **후속 리포트 생성의 트리거** |
| `SESSION_REOPENED` | 락업 해제 및 재오픈 | `session_id`, `admin_id`, `reason` | Critical | 보안 고위험 액션 (모니터링 대상) |

---

## 14. 후속 문서 영향

이 문서는 다음 명세서들을 작성하기 위한 베이스라인(Ground Truth) 정보로 사용된다.

### 표 13. 후속 문서 연결 기준
| 후속 문서명 | 연결 이유 | 이 문서에서 넘겨줄 기준 정보 | 작성 우선순위 |
| :--- | :--- | :--- | :--- |
| `F1-C-002_audit_report_generation.md` | 세션 완료 후 리포트 생성 | **세션 상태가 `Finalized`가 되었을 때만 리포트 생성이 트리거된다는 규칙** | 1순위 |
| `F1-RH-001_audit_route_handler.md` | 백엔드 API 라우팅 규격 | 세션 생성/시작/저장/제출 등 Command API의 Request/Response 및 상태 전이 규칙 | 2순위 |
| `F1-Q-001_audit_report_query.md` | 완료된 데이터 조회 | `Finalized` 상태인 세션 데이터 포맷 및 권한 필터링 기준 | 3순위 |
| `UI-010_audit_workspace_page.md` | 사용자 프론트엔드 UI | 상태(Draft~Submitted)에 따른 버튼(임시저장, 제출) 활성화/비활성화 조건 및 STT 마이크 노출 조건 | 4순위 |
| `TEST-F1-001_audit_session_flow_test.md` | 인수 테스트 시나리오 | 표 7(상태 전이 규칙), 표 10(권한 매트릭스), 표 11(오류 처리)의 기댓값 | 5순위 |

---

## 15. Sprint 기준 우선순위

### 표 14. Sprint 우선순위 정리
| 세션 기능/규칙 | Sprint | 이유 | 선행조건 | 후속 영향 | 비고 |
| :--- | :---: | :--- | :--- | :--- | :--- |
| 기본 CRUD 및 상태 전이 (Draft~Finalized) | 1 | Smart Audit의 핵심 뼈대 | DB Schema, Auth 연동 | 리포트 생성, UI 구현 | 필수 |
| Tenant/Site 기반 권한 통제 | 1 | B2B SaaS 데이터 격리 원칙 | RBAC 정책 확정 | 백엔드 미들웨어 구현 | 필수 |
| STT 텍스트 보조 입력 | 1 | Sprint 1 차별화 (Zero-UI) 요소 | Whisper API 연동 테스트 | UI 컴포넌트 개발 | 필수 |
| Audit Log 기록 (주요 상태 변화) | 1 | 데이터 신뢰성 확보 및 추적 | Audit Log Schema | 관리자 대시보드 | 필수 |
| Vision AI 양불 판정 | 2 | 핵심 기능 안정화 후 고도화 | 세션 데이터 구조 안정화 | AI 파이프라인 연계 | 보류 |
| NC(부적합) 자동 발행 연계 | 2 | 후속 조치 워크플로우는 2차 스펙 | 세션/리포트 포맷 확정 | NC 도메인 설계 | 보류 |

---

## 16. 완료 기준 (Definition of Done)

본 문서의 작성 및 이관이 완료되었다고 판단하는 기준은 다음과 같다.

1. **상태 머신 정의 완료**: 세션의 모든 상태(Draft -> Finalized)와 엣지 케이스(Reopen, Cancel) 전이 규칙이 모호함 없이 정의됨.
2. **역할(RBAC) 충돌 해소**: Tenant Admin, Site Admin, Auditor 간의 권한 경계가 표 10에 명확히 명시됨.
3. **STT 경계 명확화**: STT가 자동 판정 엔진이 아닌, 텍스트 입력 보조 수단임이 명시되고 예외 처리(수동 보정) 룰이 확립됨.
4. **리포트 분리 완료**: 리포트 생성 로직이 이 문서에 섞이지 않고, `Finalized` 상태에서 후속 문서로 제어권이 넘어간다는 점이 명시됨.
5. **승인 대기(Ready for Next)**: 백엔드/프론트엔드 팀이 이 문서를 바탕으로 API DTO 및 UI Mockup을 설계할 수 있는 수준에 도달함.

---

## 17. 다음 단계 작성 가이드

이 문서 작성 후, 전체 Smart Audit 도메인의 완성을 위해 다음 문서를 순서대로 작성할 것을 권장한다.

1. **`F1-C-002_audit_report_generation.md`**: 본 문서에서 `Finalized`된 세션 데이터를 읽어, 불변의 PDF/JSON 리포트로 말아내는 로직을 정의해야 함.
2. **`F1-RH-001_audit_route_handler.md`**: 정의된 상태 전이와 규칙을 실제 Next.js App Router (또는 API 서버)의 엔드포인트 명세로 번역해야 함.
3. **`UI-010_audit_workspace_page.md`**: 세션 상태에 따라 현장 감사자의 모바일/웹 화면이 어떻게 바뀌고, STT 버튼이 어떻게 반응하는지 설계해야 함.
4. **`F1-Q-001_audit_report_query.md`**: 확정된 리포트와 세션 데이터를 목록화하고 검색/필터링하는 조회용 성능 최적화 명세가 필요함.
5. **`TEST-F1-001_audit_session_flow_test.md`**: 본 문서의 규칙(권한 차단, 상태 전이 에러 등)을 자동화된 E2E 및 통합 테스트 시나리오로 작성하여 QA 기준을 세워야 함.

---

## Appendix A. 세션 불변성 보장 구현 코드 (F1-C-001_session_management.md 통합분)

> **[통합 이력]** 본 섹션은 기존 `F1-C-001_session_management.md` (8KB)의 고유 구현 상세(RLS SQL, 앱 레벨 선검사, 삭제 금지 정책)를 정본 문서에 병합한 것이다. 원본 파일은 중복 해소를 위해 Archive 처리됨.

### A-1. Supabase RLS(Row Level Security) 적용 SQL

Supabase 단에서 쿼리 실행을 거부하는 방어벽을 구축한다.

```sql
-- audit_data_entries (세션 데이터) 테이블 수정/삭제 차단
CREATE POLICY "Block update on entries if session completed" ON audit_data_entries
FOR UPDATE
USING (
  (SELECT status FROM audit_sessions WHERE id = audit_data_entries.session_id) NOT IN ('COMPLETED', 'ARCHIVED')
);

CREATE POLICY "Block delete strictly" ON audit_data_entries
FOR DELETE
USING (false); -- 데이터 삭제는 상태 무관 영구 금지 (Insert-only)
```

### A-2. 앱 레벨 선검사 (Server Action Guard)

Server Action이나 Route Handler의 최상단에서 DB RLS 도달 전에 미리 쳐내는 로직.

```typescript
const session = await prisma.audit_sessions.findUnique({ select: { status: true }});
if (['COMPLETED', 'ARCHIVED'].includes(session.status)) {
  throw new AppError('INVALID_STATE_TRANSITION', '완료된 세션은 수정할 수 없습니다.', 422);
}
```

### A-3. 삭제 금지 정책

- 프로젝트 내에 `DELETE FROM audit_sessions` 코드는 존재해서는 안 된다.
- 오기입으로 인한 취소 처리는 `status = 'ARCHIVED'` 상태 업데이트로만 수행(Soft Delete의 변형)하여 DB에 영구 보존한다.

### A-4. Cursor Pagination 기반 세션 목록 조회 (UI-011 연동)

무한 스크롤(Infinite Scroll) 환경에서 대용량 세션을 지연 없이 조회하기 위한 Cursor Pagination 기반 쿼리 명세이다.

**필터 파라미터 스키마 (Zod)**
```typescript
export const SessionQuerySchema = z.object({
  status: z.enum(['DRAFT', 'IN_PROGRESS', 'PENDING_REVIEW', 'COMPLETED', 'ARCHIVED', 'ALL']).default('ALL'),
  auditType: z.string().optional(),
  assigneeId: z.string().uuid().optional(),
  cursor: z.string().optional(), // Prisma의 경우 대상 레코드의 ID (UUID)
  limit: z.number().min(5).max(100).default(20),
});
```

**Prisma 쿼리 예시**
```typescript
const { status, assigneeId, cursor, limit } = SessionQuerySchema.parse(reqQuery);

const queryArgs: any = {
  take: limit + 1, // 다음 페이지 유무 확인을 위해 하나 더 가져옴
  orderBy: { updated_at: 'desc' },
  where: {
    ...(status !== 'ALL' && { status }),
    ...(assigneeId && { assignee_id: assigneeId }),
  }
};

if (cursor) {
  queryArgs.cursor = { id: cursor };
  queryArgs.skip = 1; // 기준 cursor 레코드는 제외하고 다음부터 반환
}

const sessions = await prisma.audit_sessions.findMany(queryArgs);

let nextCursor: string | undefined = undefined;
if (sessions.length > limit) {
  const nextItem = sessions.pop(); // 초과 조회한 1건 제거
  nextCursor = nextItem!.id;
}

return {
  items: sessions,
  nextCursor,
  hasMore: nextCursor !== undefined
};
```

---

### Gemini 3.1 Pro 자체 검토 체크포인트

1. [x] **세션 생명주기 완비**: Create -> Start -> Save -> Submit -> Finalize 흐름이 누락 없이 표와 텍스트로 정의되었는가?
2. [x] **상태 전이 판정 가능성**: 표 7을 통해 프론트/백엔드 개발자가 `if (state === X)` 로직을 명확히 짤 수 있게 설계되었는가?
3. [x] **STT 철학 준수**: STT를 '마법의 지팡이'로 과장하지 않고, 오류 시 수동 보정이 필수인 '키보드 대체재'로 정확히 스코핑했는가?
4. [x] **관심사 분리 (Session vs Report)**: 세션 관리에 리포트 포매팅/PDF 생성 로직이 침범하지 않고 깔끔하게 분리되었는가?
5. [x] **멀티테넌시/RBAC 경계**: `tenant_id`, `site_id` 기반 데이터 격리와 사용자 역할별 통제권이 표 10에 정확히 반영되었는가?
6. [x] **감사 가능성 (Audit Log)**: 세션의 상태 조작 행위가 `DATA-AUDIT_LOG_v1.md` 스키마와 연계되어 기록됨을 명시했는가?
7. [x] **공통 에러 대응**: `API-001_common_error_schema.md`에 맞게 HTTP Status와 에러 코드가 매핑되었는가?
8. [x] **연계 문서 호환성**: 이 문서의 산출물이 Route Handler, UI, Test 문서로 자연스럽게 흘러갈 수 있도록 브릿지 정보를 제공하는가?
9. [x] **Sprint 1 스코핑**: Vision AI나 NC 연동 같은 오버엔지니어링 요소를 배제하고 Sprint 1 목표에 집중했는가?
10. [x] **개발자 친화성**: 장황한 서술을 배제하고, 실무진이 코딩 시 바로 참조할 수 있는 '표' 중심으로 가독성을 극대화했는가?
