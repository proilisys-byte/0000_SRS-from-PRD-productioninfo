# TEST-S1_DEMO_v1.md

## Sprint 1 데모 시연용 E2E 검증 시나리오

본 문서는 `08_DECISION_LOG_v1.md`의 합의 사항에 기반하여, Sprint 1 종료 시 경영진 및 이해관계자에게 "완료(Done)"를 증명하기 위한 E2E 데모 시나리오입니다.

### 시나리오 1: STT 현장 입력 → DB 적재
* **Given (준비)**: User 계정으로 모바일 뷰(현장 입력 화면)에 로그인하여 진행 중인 Audit 세션에 진입한 상태.
* **When (실행)**: 마이크 버튼을 클릭하고 "기어 조립 공정 백 개 완료, 불량 없음" 이라고 발성한 후 전송 버튼을 누른다.
* **Then (검증)**: 
  1. 클라이언트 화면에 구조화된 데이터(공정명: 기어 조립, 수량: 100)가 표시된다.
  2. DB의 `audit_data_entries` 테이블에 해당 내용이 JSON 포맷으로 저장된 것을 확인한다.

### 시나리오 2: Bulk Import → 기준정보 적재
* **Given (준비)**: Admin 계정으로 데스크톱 뷰의 `/dashboard/bulk-import` 화면에 진입한 상태.
* **When (실행)**: 5종의 제품 마스터가 기재된 정상 CSV 템플릿 파일을 업로드하고 '일괄 등록' 버튼을 누른다.
* **Then (검증)**:
  1. UI에 "성공적으로 업로드되었습니다" 메시지가 노출된다.
  2. DB의 `bulk_import_batches` 테이블 상태가 `DONE`으로 변경되고, 관련 기준정보 테이블에 5건의 레코드가 생성된 것을 확인한다.

### 시나리오 3: Smart Audit PDF 생성 (10분 이내)
* **Given (준비)**: 이미 20건 이상의 현장 입력 데이터(`audit_data_entries`)가 누적된 세션의 상세 조회 화면에 Admin이 접속한 상태.
* **When (실행)**: 'ISO 9001 보고서 생성' 버튼을 클릭한다. (클릭 시점 시간 기록)
* **Then (검증)**:
  1. 화면에 Streaming 방식의 진행률 또는 생성 텍스트가 노출되며 Vercel 60초 타임아웃 없이 유지된다.
  2. 클릭 시점으로부터 **10분 이내**에 브라우저에서 최종 매핑된 결과물이 PDF 형태(한글 폰트 정상 출력)로 다운로드된다.

### 시나리오 4: RBAC 권한 분리 검증
* **Given (준비)**: User 계정(현장 작업자)으로 로그인 완료된 상태.
* **When (실행)**: 주소창에 Admin 전용 라우트인 `/dashboard/bulk-import`를 강제로 입력하여 이동을 시도한다.
* **Then (검증)**:
  1. LNB(사이드바)에 관리자 메뉴가 표시되지 않음을 확인한다.
  2. 강제 이동 시 HTTP 권한 거부가 발생하며 `/dashboard/unauthorized` 또는 메인 화면으로 리다이렉트 된다.

### 시나리오 5: Insert-only Audit Log 무결성 확인
* **Given (준비)**: 데모 환경의 DB 클라이언트(DBeaver, Supabase Studio 등)에 Service Role로 접속된 상태.
* **When (실행)**: 시나리오 1~3을 거치며 쌓인 `audit_log` 테이블의 특정 레코드를 대상으로 `UPDATE audit_log SET action = 'MODIFIED' WHERE id = '...';` 쿼리를 실행한다.
* **Then (검증)**:
  1. DB 엔진에서 커스텀 예외(`COMPLIANCE LOCKUP: Modification or deletion of audit_log records is strictly prohibited.`)가 발생하며 쿼리가 거부됨을 시연 참석자에게 시각적으로 증명한다.
