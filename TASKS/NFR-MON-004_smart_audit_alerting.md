# NFR-MON-004_smart_audit_alerting.md
# NFR-MON-004: Smart Audit Alerting Specification

## 1. 문서 목적
본 문서는 PRO ILI SMART 플랫폼 Smart Audit 환경에서 발생하는 모니터링 이벤트 중, 운영진의 개입이 필요한 이상 징후를 식별하고 이를 적절한 수신자에게 신속하고 정확하게 경보(Alert)하기 위한 기준을 정의한다. 오탐(False Positive)을 최소화하고 조치 가능한(Actionable) 알림 체계를 구축하는 것을 목표로 한다.

## 2. Alerting 설계 원칙
| 원칙 | 설명 |
| --- | --- |
| Actionable Alerts | 수신자가 내용을 보고 즉각적인 후속 조치(분석, 복구, 고객 안내 등)를 취할 수 있는 경우에만 경보 발생 |
| Alert Fatigue 방지 | 중복 경보를 억제(Grouping/Throttling)하여 경보 피로도 완화 |
| 권한 및 역할 기반 라우팅 | 장애의 성격(보안, 시스템, 비즈니스)에 따라 적절한 운영자(System Admin, Tenant Admin 등)에게 선별적 전파 |
| Audit Log 무결성 최우선 | Audit Log 적재 실패는 규제 위반 사항이므로 최상위 심각도(Critical)로 분류 및 즉각 경보 |

## 3. 적용 범위 정의
| 범위 | 포함 대상 | 제외 대상 (Sprint 2+) |
| --- | --- | --- |
| 대상 서비스 | Audit Session Lifecycle, Audit Report Generation | Legacy 외부 연동 지연, 향후 도입될 AI 오토메이션 알림 |
| 이벤트 타입 | 보안 위반, 시스템 에러(5xx), 프로세스 지연, DB 무결성 예외 | 비즈니스 KPI 미달 알림 (예: 이번 주 심사 건수 부족) |
| 통보 수단 | 이메일, 메신저(Slack/Teams), SMS (Critical 한정) | ITSM 시스템 자동 티켓팅 생성 (추후 연동) |

## 4. Alert 이벤트 분류
| 분류 코드 | 분류명 | 성격 | 대표 사례 |
| --- | --- | --- | --- |
| `ALT-SEC` | Security | 권한 및 보안 통제 위반 | 인가되지 않은 Tenant/Site 접근 시도, 비정상 토큰 반복 사용 |
| `ALT-SYS` | System & Error | 시스템 장애 및 예외 | Route Handler 500 에러 증가, 외부 의존성(LLM API) 장애 |
| `ALT-BIZ` | Business Logic | 비즈니스 프로세스 지연/실패 | 리포트 생성(Generating) 상태 15분 초과 정체, 유효하지 않은 상태 전이 과다 |
| `ALT-DTB` | Data Integrity | 데이터 무결성 훼손 위험 | Audit Log Insert 실패, 트랜잭션 롤백 급증 |

## 5. Alert 우선순위 체계
| 심각도 (Severity) | 정의 | 요구 응답 시간 | 통보 채널 |
| --- | --- | --- | --- |
| `Critical` | 시스템 중단 또는 심각한 데이터 유실/보안 위반 상황 (Audit Log 실패 등) | 즉각 조치 (24x7) | SMS, On-call Paging, 즉각 메신저 |
| `High` | 주요 기능(리포트 생성 등)의 광범위한 실패 또는 잦은 권한 위반 | 1시간 이내 | 메신저 (High Priority 채널) |
| `Medium` | 국지적 실패, 성능 저하, 간헐적 에러 발생 | 업무 시간 내 | 메신저 (일반 채널), Email |
| `Low` | 시스템 지표의 이상 조짐, 장기 방치된 세션 등 경고 성격 | 일일 점검 시 | Email 리포트, 대시보드 표출 |

## 6. Alert 트리거 조건
| Alert ID | Alert명 | 심각도 | 트리거 조건 (예시 임계치) | 분류 |
| --- | --- | --- | --- | --- |
| `A-AUD-001` | Audit Log Insert Failure | Critical | Audit Log 적재 실패 (5xx) 1건 이상 발생 시 즉시 | `ALT-DTB` |
| `A-AUD-002` | Cross-Tenant Access Attempt | High | 5분 내 동일 IP/User의 Cross-Tenant 403 에러 3회 이상 | `ALT-SEC` |
| `A-AUD-003` | Report Generation Stuck | High | `Generating` 상태로 15분을 초과한 세션 발생 | `ALT-BIZ` |
| `A-AUD-004` | High Error Rate (Report API) | High | 5분 간 Report 생성 API의 5xx 에러율 5% 초과 | `ALT-SYS` |
| `A-AUD-005` | Stale Draft Sessions | Low | `Draft` 상태로 14일 이상 변경이 없는 세션 비중 20% 초과 | `ALT-BIZ` |

## 7. 수신자 / 역할 기준
| 대상 역할 (Role) | 관심 영역 | 수신 Alert 목록 (예시) |
| --- | --- | --- |
| System Admin (Platform) | 인프라, 전역 보안, 플랫폼 장애 | `A-AUD-001`, `A-AUD-004` |
| Security Officer | 비정상 접근, 권한 탈취 시도 | `A-AUD-002` |
| Ops / SRE | API 성능, 상태 정체 문제 | `A-AUD-003`, `A-AUD-004` |
| Tenant Admin | 자사 구성원의 세션 정체, 단순 권한 오류 | `A-AUD-005` (요약 리포트로 수신) |

## 8. 중복 Alert 억제 기준
| 억제 기법 | 적용 방안 |
| --- | --- |
| Grouping | 동일 `tenant_id` 및 `error_code` 발생 건을 5분 단위로 묶어 단일 알림으로 발송 |
| Throttling | 지속적인 장애 상황(예: 외부 LLM API 다운) 시 첫 알림 후 30분간 동일 알림 억제(Silence) |
| Flapping 방지 | 경계선에서 수치 오르내림으로 인한 반복 알림을 막기 위해, 정상 상태로의 회복 조건은 트리거 조건보다 여유 있게 설정 (예: 에러율 5% 초과 시 알림 -> 1% 미만으로 5분 유지 시 Resolved 처리) |

## 9. 오탐 / 과잉경보 완화 기준
| 시나리오 | 완화(Mitigation) 전략 |
| --- | --- |
| QA/테스트 환경의 에러 알림 | 환경 변수(`NODE_ENV`) 또는 헤더를 통해 Production 환경에서만 High/Critical 경보 활성화 |
| 400 Bad Request에 의한 에러율 급증 | 알림 임계치 산정 시 클라이언트 오류(4xx) 제외, 순수 서버 오류(5xx) 및 명시적 보안 위반(401/403)만 타겟팅 |
| 의도된 Timeout 테스트 (`X-Mock-Scenario`) | Mock 헤더에 의해 유도된 에러/타임아웃은 Alert 통계에서 제외하도록 로거 단에서 태깅 (`mock: true`) |

## 10. 운영 대응 / 에스컬레이션 기준
| 단계 | 조치 사항 | 책임자 |
| --- | --- | --- |
| 1. 알림 수신 및 인지 | Alert 내용 확인 (Trace ID, 대상 Tenant/Session 식별) | L1 Ops / System Admin |
| 2. 대시보드 연계 분석 | `ADM-063` 대시보드를 통해 현황 파악 (연쇄 장애 여부 확인) | L1 Ops |
| 3. 에스컬레이션 | 15분 내 원인 미파악 시, Backend Dev 또는 Infra 팀으로 이관 | L2 Backend/SRE |
| 4. 조치 및 Resolved | 원인 제거, 데이터 복구(`A-AUD-001`의 경우 수동 적재 등) 후 경보 종료 | 담당 엔지니어 |

## 11. Monitoring / Error Schema / Audit Log 연계
| 연계 대상 문서 | 연결 고리 설명 |
| --- | --- |
| `NFR-MON-003` (Monitoring) | 모니터링 메트릭(`M-AUD-xxx`)이 설정된 임계치를 위반할 때 본 문서의 Alert(`A-AUD-xxx`) 발생 |
| `API-001` (Error Schema) | 알림 메시지에 `API-001`에서 정의한 `error_code`와 `trace_id`를 필수 포함시켜 원인 분석 지원 |
| `DATA-AUDIT_LOG_v1` | Audit Log 기록 시스템의 실패는 가장 심각한 이벤트(`A-AUD-001`)로 직결됨 |

## 12. Sprint 우선순위
| 항목 | 우선순위 (Sprint 1) | 비고 |
| --- | --- | --- |
| `Critical` 및 `High` 등급 룰셋 적용 | **P1 (필수)** | 시스템 안정성 및 보안 확보를 위해 필수 |
| 중복 경보 억제(Grouping) 기본 설정 | **P1 (필수)** | Alert Fatigue 예방 |
| `Low` 등급 알림 및 요약 리포팅 이메일 | P3 | 운영 안정화 후 Sprint 2+ 에 구현 |

## 13. 완료 기준 (Definition of Done)
- [ ] `A-AUD-001`~`004`에 대한 Alert Rule이 모니터링 툴(Prometheus Alertmanager 등)에 설정 완료됨
- [ ] 알림 메시지에 `trace_id`, `tenant_id`, `error_code`가 정상 포함되어 메신저(Slack 등)로 전송됨을 테스트 완료
- [ ] MOCK 환경의 테스트 에러가 프로덕션 알림으로 오탐되지 않음을 확인 완료

## 14. 다음 단계 작성 가이드
- SRE / Ops 담당자: Alertmanager Configuration(YAML) 작성 및 Notification 채널(Webhook, Slack Bot) 연동 설정
- 백엔드 개발자: 의도된 Mock 테스트 트래픽이 Alert Rule에 잡히지 않도록 커스텀 로그 태그(Ex. `is_synthetic_test`) 추가 배포

## 15. Gemini 3.1 Pro 자체 검토 체크리스트
- [x] 경보가 무분별하게 울리지 않도록 조치 가능한(Actionable) 이벤트 위주로 분류했는가?
- [x] Audit Log 실패, Tenant/Site 침범 등 보안/컴플라이언스 위배 사항이 심도 있게 다뤄졌는가?
- [x] 모니터링 문서(`NFR-MON-003`)와 중복되지 않으면서 상호 연결 고리를 명확히 했는가?
