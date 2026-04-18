 # PRO ILI SMART - SRS v1.0 Tech Stack Transition Plan

## 1. 개요 (Overview)
본 문서는 PRO ILI SMART 시스템의 소프트웨어 요구사항 명세서(SRS)를 최신 단일 통합 프레임워크(Next.js 풀스택) 관점으로 전면 개정하기 위한 실행 계획(Plan)을 정의합니다. 
MVP(Minimum Viable Product) 단계에서 초기 인프라 구축 비용을 $0에 수렴하게 만들고, 개발 복잡도를 최소화하기 위해 지정된 7가지 기술 제약사항(C-TEC-001 ~ 007)을 SRS에 반영함과 동시에, 잠재적 기술 리스크를 아키텍처적으로 완전히 해소(Resolved)한 확정안을 제시합니다.

---

## 2. 기술 스택 제약사항(Constraints) 적용 계획

SRS-v0.5.md 의 **1.2.3 Constraints / Assumptions** 및 **6.8 Appendix C**, **3. System Context** 영역에 아래의 기술 스택을 확정적 제약사항으로 명시합니다.

| ID | 적용 대상 스택 | 반영할 문서 위치 및 수정 방향 |
|:---|:---|:---|
| **C-TEC-001** | Next.js (App Router) | 시스템 아키텍처 다이어그램 및 제약사항에 명시. 프론트엔드/백엔드 분리 구조 제거. 단일 통합 풀스택 프레임워크 강제. |
| **C-TEC-002** | Server Actions / Route Handlers | API Overview 및 Component Diagram 수정. RESTful API 서버 구성 대신 Server Actions와 Route Handlers를 이용한 내부 데이터 처리 로직 명시. |
| **C-TEC-003** | Prisma + SQLite (Local) / Supabase (Prod) | 데이터베이스 환경 명시. 로컬 개발 환경의 편의성(SQLite)과 프로덕션 환경의 확장성(Supabase PostgreSQL)을 동시에 충족하는 Prisma ORM 도입. |
| **C-TEC-004** | Tailwind CSS + shadcn/ui | 비기능 요구사항(NFR) 중 사용성(Usability) 및 UI 제약에 명시. AI 코드 생성의 일관성 확보를 위한 디자인 시스템 고정. |
| **C-TEC-005** | Vercel AI SDK | 별도의 Python(FastAPI 등) AI 오케스트레이션 서버 제거. Next.js 내부에서 AI SDK를 통해 파이프라인을 구축하도록 아키텍처 다이어그램 수정. |
| **C-TEC-006** | Google Gemini API 기본 연동 | 외부 연동 시스템(External Systems)에 Gemini API(Multimodal) 명시. STT/Vision 처리를 Gemini로 통합하여 비용 및 인프라 축소. |
| **C-TEC-007** | Vercel Git Push 자동 배포 | CI/CD 아키텍처 수정. 별도의 복잡한 파이프라인 툴 없이 Vercel 플랫폼을 통한 인프라 및 배포 자동화 명시. |

---

## 3. 기술적 제약 극복 및 아키텍처 확정안 (Resolved Architecture)

잠재적 리스크 요소를 사전에 차단하기 위해 설계 단계에서 확정된 기술적 해결 방안입니다. 이 방안을 통해 기술 스택 전환에 따른 위험을 '안전(Green)' 상태로 관리합니다.

### ✅ 해결안 1. Prisma Client Extensions를 활용한 DB 호환성 확보
* **현황:** 로컬 SQLite(JSON 미지원)와 운영계 PostgreSQL(JSONB 지원) 간의 데이터 타입 불일치.
* **확정 솔루션:** 
  - Prisma의 `client extensions` 기능을 사용하여 SQLite 환경에서도 JSON 데이터를 자동으로 직렬화/역직렬화하는 미들웨어 로직을 필수 적용합니다.
  - 이를 통해 개발자는 환경에 관계없이 동일한 객체 지향적 코드를 작성할 수 있으며, 배포 시 데이터 깨짐 현상을 원천 차단합니다.

### ✅ 해결안 2. Edge Runtime 및 Streaming 아키텍처 강제 적용
* **현황:** Vercel 서버리스의 페이로드 제한(4.5MB) 및 타임아웃(최대 5분) 제약.
* **확정 솔루션:**
  - **대용량 데이터:** 4.5MB를 초과하는 현장 데이터(음성/영상)는 `Supabase Storage SDK`를 사용하여 클라이언트에서 직접 업로드(Direct Upload) 하도록 시퀀스를 고정합니다.
  - **긴 작업 시간:** AI 리포트 생성 및 NC 분석 등 시간이 걸리는 작업은 `Next.js Edge Runtime`과 `Vercel AI SDK Streaming`을 결합하여 연결 끊김 없이 UI에 실시간 결과를 전달하도록 구현합니다.

### ✅ 해결안 3. GitHub Native 보안/품질 게이트 활용
* **현황:** 별도 CI/CD 파이프라인 미구축에 따른 보안 취약점 점검 누락 우려.
* **확정 솔루션:**
  - Vercel 배포와 별개로, GitHub 저장소 자체 기능인 `CodeQL(SAST)`, `Dependabot(의존성 스캔)`, `GitHub Actions(Unit Test)`를 기본 활성화합니다.
  - 별도의 복잡한 인프라 설정 없이도 PR(Pull Request) 단계에서 코드 품질과 보안을 자동으로 검증하는 'Zero-Config QA' 체계를 확립합니다.

---

## 4. 핵심 사용자 경험(MVP 가치 전달) 훼손 여부 검토

새로운 기술 스택(C-TEC-001 ~ 007) 적용이 MVP의 핵심 가치에 미치는 영향을 검토한 결과, **사용자 경험의 훼손 없이 오히려 가치가 극대화**되는 것으로 확인되었습니다.

| 핵심 사용자 경험 (MVP 가치) | 기존 아키텍처 방식 | 새로운 기술 스택 (Vercel/Supabase/Gemini) | 훼손 여부 및 검토 의견 |
| :--- | :--- | :--- | :---: |
| **1. Zero-UI 현장 수집** | 별도 Python AI 서버(STT/Vision) 거침 | **Vercel AI SDK + Gemini Multimodal** 직접 연동 | **[향상]** 응답 속도 단축 및 수집 마찰 제거 |
| **2. Smart Audit 자동화** | 생성 완료 시까지 로딩 스피너 대기 | **Vercel AI SDK UI Streaming** 적용 | **[향상]** 실시간 생성 과정 노출로 사용자 신뢰도 상승 |
| **3. 데이터 무결성 보장** | AWS S3 Object Lock 등 복잡한 설정 | **Supabase RLS + DB 해시 체인(WORM)** | **[유지]** 구현 복잡도는 낮추고 무결성은 동일 보장 |
| **4. 초저비용 도입 (ROI)** | AWS Fargate/RDS 등 고정비 발생 | **Vercel + Supabase Free Tier ($0)** | **[극대화]** 초기 인프라 비용 $0 달성으로 도입 장벽 제거 |
| **5. 유지보수 및 확장성** | 프론트/백엔드/AI 인프라 분절 관리 | **Next.js 단일 통합 코드베이스** | **[향상]** 소규모 팀의 운영 부하를 줄이고 개발 속도 가속 |

### 🎯 결론
요청하신 **Next.js 단일 풀스택 + Vercel/Supabase + Gemini API 생태계로의 전환**은 MVP가 달성하고자 하는 "비용 최소화, 유지보수 극단적 간소화, AI 중심의 마찰 없는 경험"이라는 핵심 목적에 가장 부합하는 아키텍처입니다. **확정된 기술 해결 방안을 통해 초기 리스크를 모두 해소하였으므로, 안정성과 속도를 모두 갖춘 상태로 개발에 착수할 수 있습니다.**
