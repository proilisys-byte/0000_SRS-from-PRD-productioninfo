# 06. 기술 부채 및 아키텍처 설계 결정 (Tech Debt & Arch Decisions)

## 1. 문서 목적
본 문서는 PRO ILI SMART 프로젝트가 Sprint 2에 진입하기 전에 반드시 해결하거나 명확한 정책을 수립해야 할 **기술 부채(Tech Debt)** 및 **아키텍처 결정 사항(Architecture Decisions)**을 식별하고 관리하기 위해 작성되었습니다. 코드 구현 전 발생할 수 있는 치명적 장애와 재작업을 선제적으로 방지하는 것이 목표입니다.

## 2. 기술 부채와 설계 결정의 범위
- **범위:** 시스템의 확장성, 안정성, 보안, 배포에 직접적인 영향을 미치는 백엔드/인프라 이슈 중심 (Vercel, Supabase, Prisma, Gemini 연동).
- **제외:** 단순 UI/UX 컴포넌트 개선, 비즈니스 로직 레벨의 정책(예: 승인 결재선 등)은 제외.

## 3. 현재 확인된 기술 리스크
다음은 F1 도메인 및 공통 설계 문서 리뷰 결과 도출된 핵심 기술 리스크 목록입니다.

### 표 1. 기술 리스크 목록
| 리스크 ID | 리스크 명 | 발생 영역 | 리스크 설명 |
|---|---|---|---|
| **TR-001** | Gemini API Rate Limit 초과 | AI 연동 (F1 리포트) | 다수 테넌트 동시 리포트 산출 요청 시 Gemini 할당량 한계로 인한 서비스 중단 위협 |
| **TR-002** | 권한 제어(RLS vs Prisma) 충돌 | 데이터베이스 (Supabase) | Supabase의 Row Level Security(RLS)와 Prisma Client 권한 로직 간 이중 적용/충돌로 인한 성능 저하 및 에러 추적 난해 |
| **TR-003** | PDF 한글 렌더링 깨짐 | 백엔드 산출물 (리포트) | 서버리스 환경(Vercel)에서 Puppeteer/Playwright 동작 및 한글 폰트(CJK) 로딩 불가로 인한 PDF 렌더링 실패 |
| **TR-004** | Vercel Timeout 및 Streaming 처리 | API Gateway (Vercel) | Vercel Serverless Function(Hobby/Pro 제약: 10s~60s) 시간 초과로 대용량 리포트 생성 및 다운로드 API 실패 |
| **TR-005** | 로컬(SQLite) ↔ 상용(PostgreSQL) 이식성 | DB Migration (Prisma) | 로컬 개발 환경(SQLite)과 Supabase(PostgreSQL) 간의 JSON, Array, Enum 타입 비호환성에 따른 마이그레이션 에러 |

## 4. 항목별 분석

### 표 2. 항목별 선택지 및 권장안 비교
| 리스크 ID | 발생 원인 / 문제점 | 선택지 (Options) | 권장안 (Recommendation) |
|---|---|---|---|
| **TR-001** | 대규모 텍스트 요약 시 API 호출량 급증 | A. 요청 동기 처리 (에러 발생 시 사용자 재시도 유도) <br> B. **비동기 큐잉(Queue) + 백오프(Backoff) 적용** | **옵션 B:** Vercel KV 또는 Upstash Redis 기반 큐를 도입하고, 상태를 `Generating`으로 두고 Polling 처리 |
| **TR-002** | RLS와 Application 레벨 권한 분리 기준 모호 | A. Prisma로만 검증하고 RLS Disable <br> B. **Prisma는 Query Builder로만 쓰고, Supabase JWT 연동 RLS 강제** | **옵션 B:** B2B SaaS의 멀티테넌트 데이터 유출 방지를 위해 DB 단의 RLS를 최종 방어선으로 채택 |
| **TR-003** | Serverless 환경의 브라우저 바이너리/폰트 용량 초과 | A. Vercel에서 직접 PDF 렌더링 <br> B. **프론트엔드(Client-side) PDF 생성 (html2pdf 등)** <br> C. 별도 PDF 전용 마이크로서비스 구축 | **옵션 B:** 백엔드/Vercel 부하를 없애기 위해 클라이언트 단에서 브라우저 내장 렌더링 기능을 활용하여 PDF 생성 |
| **TR-004** | AI 요약 등 Long-running Task에 Vercel 런타임 종료됨 | A. Vercel Pro 업그레이드(최대 300s) <br> B. **AI 응답 Streaming 처리 적용 (Vercel AI SDK)** | **옵션 B:** 긴 작업은 Streaming 방식으로 청크(Chunk)를 내려보내 Vercel Timeout을 우회하고 사용자 체감 시간 단축 |
| **TR-005** | SQLite는 JSONB, Enum 등 PostgreSQL 고유 기능 미지원 | A. 로컬도 Dockerized PostgreSQL 강제 사용 <br> B. Prisma 스키마를 최저 스펙으로 하향 평준화 | **옵션 A:** 개발/운영 환경 불일치로 인한 장애 방지를 위해 로컬 개발자 환경에 `docker-compose.yml` 기반 PostgreSQL 강제 |

## 5. 권장 아키텍처/운영 방향
위의 분석을 토대로, Sprint 2 및 이후 확장성을 고려한 아키텍처 운영 방향은 다음과 같습니다.
1. **비동기 및 분산 처리 지향:** 무거운 작업(AI, PDF, Bulk)은 결코 동기 API로 열어두지 않는다.
2. **보안의 계층화 (Defense in Depth):** 애플리케이션 레벨(Prisma Middlewares)과 인프라 레벨(Supabase RLS)의 역할을 명확히 분리한다. (Prisma는 비즈니스 로직 검증, RLS는 테넌트 격리)
3. **환경 일치화:** 모든 개발자는 로컬에서 SQLite를 폐기하고 PostgreSQL Docker 환경을 사용한다.

## 6. Sprint 2 전 완료 필요 항목

### 표 3. Sprint 2 전 해결 필요 목록
| 식별 ID | 해결 필요 과제 | 담당/주체 | 완료 기한 (목표) | 상태 |
|---|---|---|---|---|
| **TR-005** | 로컬 개발 환경용 `docker-compose.yml` (PostgreSQL) 배포 | System Architect | Sprint 1 1주차 | 미결 |
| **TR-002** | Supabase JWT - Prisma 연동 RLS 가이드라인 작성 | DB Admin | Sprint 1 2주차 | 미결 |
| **TR-004** | Next.js API Route에서의 Streaming 응답 보일러플레이트 작성 | Backend Lead | Sprint 1 2주차 | 미결 |

## 7. 후속 문서화 필요 항목

### 표 4. 후속 문서화 추천 표
| 문서 카테고리 | 생성 필요 문서명 | 목적 및 반영 내용 |
|---|---|---|
| 인프라/환경 | `ENV-001_local_dev_setup.md` | TR-005 대응. Docker 기반 PostgreSQL 셋업 및 초기 시드 주입 가이드 |
| 백엔드/보안 | `COM-SEC-001_supabase_rls_policy.md` | TR-002 대응. 각 테이블별 RLS 정책 문법 및 적용 가이드 |
| 프론트/유틸 | `UI-UTIL-001_pdf_export_guide.md` | TR-003 대응. Client-side PDF 렌더링 컴포넌트 사용 가이드 |

## 8. 우선순위 및 영향도

### 표 5. 항목별 영향도 표
| 리스크 ID | 심각도(Severity) | 발생 가능성(Prob.) | 영향도 및 비즈니스 충격 | 우선순위 |
|---|:---:|:---:|---|:---:|
| **TR-005 (DB 이식성)** | High | High | 개발 환경과 상용 간 100% 충돌. 마이그레이션 파탄. | **P0 (최고)** |
| **TR-002 (RLS 충돌)** | Critical| Medium | 쿼리 에러 시 원인 파악 불가능 및 타 고객사 데이터 노출 위험 | **P0 (최고)** |
| **TR-004 (Vercel Timeout)**| High | High | 리포트 생성 버튼 클릭 시 100% 504 Gateway Timeout 발생 | **P1** |
| **TR-001 (Rate Limit)** | Medium | Medium | 동시 다발적 감사 종료 시 생성 지연 발생 (일부 사용자 경험 저하) | **P1** |
| **TR-003 (PDF 폰트)** | High | Medium | 공식 산출물 출력 불가로 앱 효용가치 하락 | **P2** |

## 9. Definition of Done
- [x] Sprint 2 이전에 해결해야 할 기술 부채 및 리스크를 5가지 핵심 항목으로 도출하였는가?
- [x] 각 리스크별 문제 상황, 선택지, 권장 결정안이 명확히 대비되었는가?
- [x] 시스템 및 아키텍처 레벨에서의 결정 사항을 후속 액션 아이템(문서화, 설정)으로 연결했는가?
- [x] 팀 단위의 의사결정을 촉구할 수 있도록 위험도 및 우선순위가 명시되었는가?
