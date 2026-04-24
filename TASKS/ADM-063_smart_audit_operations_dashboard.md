# ADM-063_smart_audit_operations_dashboard.md
# ADM-063: Smart Audit Operations Dashboard Specification

## 1. 문서 목적
본 문서는 PRO ILI SMART 플랫폼 운영자가 Smart Audit(F1) 라인의 상태, 병목 현상, 보안/권한 위반 및 예외 상황을 모니터링하고 대응하기 위한 통합 대시보드(Operations Dashboard)의 화면 구조와 데이터 표시 기준을 정의한다.

## 2. 운영 대시보드 설계 원칙
| 원칙 | 설명 |
| --- | --- |
| Action-Oriented UI | 지표 확인에 그치지 않고, 문제가 발생한 세션/리포트로 신속하게 드릴다운(Drill-down)하여 원인을 파악할 수 있도록 링크/버튼 제공 |
| Data Privacy & RBAC | 운영자 권한 등급에 따라 대시보드에 노출되는 세션 데이터(민감 정보, 오디오 STT 내용 등)는 마스킹 또는 접근 차단 |
| 상태 가시성 강화 | Session(Draft~Closed)과 Report(Generating~Generated, Superseded)의 상태를 명확히 구분하여 시각화 |
| Alert 통합 표시 | `NFR-MON-004` 기반으로 발생한 진행 중(Active) 알림을 대시보드 최상단에 노출 |

## 3. 적용 범위 정의
| 범위 | 포함 대상 | 제외 대상 (Sprint 2+) |
| --- | --- | --- |
| 대상 데이터 | 전체 Tenant의 Smart Audit Session 및 Report 상태 데이터 (운영자 권한 내) | Bulk Import 작업 이력 (`ADM-062`에서 별도 관리) |
| 주요 기능 | 시스템 KPI 요약, 상태별 세션 목록, 에러/알림 패널, 추적용 로그 조회 링크 | 세션/리포트 데이터 직접 수정 기능 (Read-only 원칙) |

## 4. 역할별 대시보드 관점
| 역할 (Role) | 대시보드 조회 범위 및 특징 | 조치 권한 |
| --- | --- | --- |
| System Admin | 전사(All Tenants) 트래픽, 시스템 에러(5xx), 보안 알림 중심 조회 | 로그 시스템 딥링크 진입, 알림 Mute 처리 |
| Tenant Admin | 소속 Tenant 하위의 세션 진행률, 정체된 세션, 감사 결과 통계 | 특정 세션 상세 화면(`UI-010`) 이동 (Read 권한 필요) |
| Site Manager | 본인 관할 Site의 세션/리포트 현황 파악 | 현장 담당자에게 상태 독려/연락 |

## 5. 화면 구조 개요
대시보드는 다음 4개의 주요 영역으로 구성된다.
1. **Global Control & Alert Header**: Tenant/Site 필터, 시간대 필터, 최상단 Active Alerts 표시 영역
2. **KPI Summary Cards**: 상태별 누적 수량, 성공률, 평균 소요 시간 요약
3. **Session & Report Distribution**: 상태 전이 병목을 확인하는 파이프라인(Funnel) 차트 및 상태 분포도
4. **Exception & Drill-down List**: 지연 중(Stuck), 에러 발생, 권한 오류가 발생한 세션 리스트

## 6. KPI 카드 정의
대시보드 상단에 배치될 핵심 지표 타일:
| 카드명 | 데이터 소스 (`NFR-MON-003` 매핑) | 표시 형태 | 색상/임계치 시각화 |
| --- | --- | --- | --- |
| Total Active Sessions | `audit_sessions` (상태 != Closed/Cancelled) | 숫자 + 증감 추이 | N/A |
| Report Generating Time | `M-AUD-002` | 평균 분(Min) | 10분 초과 시 Warning(노란색) |
| Failed Reports | `M-AUD-003` | 실패 건수 | 0 초과 시 Danger(빨간색) |
| Security/Auth Blocks | `M-AUD-004` (403 에러 횟수) | 차단 건수 | 0 초과 시 Danger(빨간색) |

## 7. 상태 분포 / 병목 가시화 기준
| 차트 유형 | 목적 | 표시 기준 및 항목 |
| --- | --- | --- |
| Funnel Chart | 단계별 이탈/정체 확인 | `Draft` -> `Submitted` -> `Generating` -> `Generated` -> `Finalized` 볼륨 비교 |
| Stacked Bar | 상태별 누적 현황 | X축: 날짜, Y축: 세션 수 (상태별 색상 누적) |
| Time in State | 병목 구간 파악 | 현재 각 상태(`InProgress`, `Generating` 등)에 머물고 있는 평균 시간 |

## 8. 오류 / 경보 패널 기준
현재 진행 중인 문제 상황을 노출하는 패널:
| 항목 | 표시 내용 | 연계 기준 |
| --- | --- | --- |
| Active Alerts | `Critical`/`High` 알림 제목, 발생 시간, 지속 시간 | `NFR-MON-004`의 Alert Rule 트리거와 실시간 연동 |
| Top Error Codes | 가장 빈번하게 발생하는 예외 코드 (예: `F1-GEN-002`) | `API-001` 공통 에러 스키마의 `error_code` 기준 집계 |
| Audit Log Failures | Audit DB 적재 실패 건수 알림 (규제 리스크) | 발생 시 최상단 고정 노출 (Red 뱃지) |

## 9. 세션 / 리포트 리스트 및 드릴다운 기준
에러나 지연이 발생한 구체적인 대상을 표 형태로 제공:
| 컬럼명 | 설명 | 드릴다운 액션 |
| --- | --- | --- |
| Session ID | 해당 세션의 고유 식별자 | 클릭 시 `UI-010_audit_workspace_page` 상세조회 이동 |
| Tenant / Site | 소속 테넌트 및 사업장 정보 | 필터링 토글 |
| Status (Session) | 현재 세션 상태 (예: `Submitted`) | - |
| Status (Report) | 리포트 상태. 지연/에러 시 아이콘 강조 | 에러 메시지 툴팁 제공 |
| Time in Status | 현재 상태 지속 시간 | 15분 이상(`Generating`) 시 빨간색 강조 |
| Trace ID | 에러 발생 시 발급된 추적 ID | 복사 버튼 및 통합 로그 시스템(Kibana/Grafana) 검색 링크 |

## 10. Superseded / 최신본 / 이력본 표시 기준
| 상태 표시 | 대시보드 처리 원칙 | 목적 |
| --- | --- | --- |
| `Superseded` 처리 | 오류 리스트 노출 시 `Superseded`(과거 이력본)는 기본적으로 숨김 처리 | 운영자의 피로도 감소 및 최신본 집중 |
| 이력 추적 | 특정 `Session ID`로 상세 검색 시, 해당 세션에 딸린 모든 리포트 이력(Superseded 포함)을 트리 구조로 확인 가능 | Reopen에 의한 중복 생성 히스토리 파악 |

## 11. 행동 버튼 / 운영 조치 진입 진입 기준
| 버튼 / 액션명 | 권한 | 동작 |
| --- | --- | --- |
| View Log Details | System Admin | `trace_id`를 파라미터로 하여 Kibana/Datadog 로그 조회 딥링크 이동 |
| View Workspace | Tenant Admin / System Admin | 세션 상세 화면(`UI-010`)으로 이동 (단, 마스킹 규칙 적용) |
| Acknowledge Alert | Ops | 발생한 알림을 인지 상태로 변경하여 Dashboard 강조 해제 |

## 12. 권한 / 보안 / 민감정보 노출 기준
| 데이터 속성 | System Admin 뷰 | Tenant Admin 뷰 |
| --- | --- | --- |
| 메타데이터 (ID, 상태, 시간) | 전체 테넌트 조회 가능 | 소속 테넌트만 조회 가능 |
| Audit Report 본문 및 결과 | **접근 불가 (또는 마스킹)** | 권한 소유 시 조회 가능 |
| 에러 상세 Stack Trace | 조회 가능 (디버깅 목적) | 간략화된 메시지만 제공 |

*참고: System Admin은 플랫폼 전체 장애/성능을 모니터링하지만, 개별 Tenant의 영업 비밀/감사 결과(민감정보)를 무단 열람할 수 없도록 UI 단독 진입을 제한한다.*

## 13. Monitoring / Alerting / RH / UI 연계 기준
| 대상 문서 | 연계 포인트 |
| --- | --- |
| `NFR-MON-003/004` | 대시보드의 KPI 및 Active Alert 데이터 소스로 직접 활용 |
| `F1-RH-001` | Route Handler의 상태 전이 이벤트를 List와 Funnel 차트에 반영 |
| `UI-010` (Workspace) | 대시보드에서 문제 세션 클릭 시, UI-010 화면으로 컨텍스트(Session ID) 유지하며 전환 |

## 14. Sprint 우선순위
| 항목 | 우선순위 (Sprint 1) | 비고 |
| --- | --- | --- |
| KPI 요약 카드 및 오류 리스트 기본 구현 | **P1 (필수)** | 최소한의 운영 대응 시야 확보 |
| Tenant 권한 기반 데이터 격리 처리 | **P1 (필수)** | 보안 컴플라이언스 |
| 상태 파이프라인(Funnel) 차트 | P2 | 단순 리스트 뷰 우선 후 점진적 고도화 |
| `trace_id` 기반 로그 시스템 딥링크 | P2 | 수동 복사/붙여넣기로 1차 대체 가능 |

## 15. 완료 기준 (Definition of Done)
- [ ] System Admin / Tenant Admin 권한에 따라 대시보드 리스트의 조회 범위가 분리됨을 확인
- [ ] 15분 초과 정체 상태 및 에러 발생 세션이 리스트 최상단에 강조 표시됨을 확인
- [ ] `trace_id`를 쉽게 복사할 수 있는 UI 컴포넌트가 구성되어 디버깅 활용에 문제 없음을 확인

## 16. 다음 단계 작성 가이드
- 프론트엔드 개발자: `UI-061_bulk_import_admin_dashboard.md`의 레이아웃 패턴을 재사용하여 컴포넌트 일관성 유지
- 백엔드 개발자: 대시보드 전용 집계 쿼리(Aggregations) 구현 시, 운영 DB 부하를 막기 위해 Read Replica를 바라보거나 캐싱(Redis) 적용 검토

## 17. Gemini 3.1 Pro 자체 검토 체크리스트
- [x] 단순 모니터링 뷰가 아니라, 운영자가 "무엇을 클릭하고 조치할지" 액션 중심으로 설계되었는가?
- [x] 권한 없는 시스템 관리자가 고객의 민감한 리포트 내용을 무단 열람하지 못하도록 접근 통제 기준을 세웠는가?
- [x] Superseded 상태의 과거본이 운영 화면을 어지럽히지 않도록 노출 제어 규칙을 명시했는가?
