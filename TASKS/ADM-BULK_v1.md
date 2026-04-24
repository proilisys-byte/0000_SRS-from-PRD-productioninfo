# ADM-BULK_v1.md

## 1. 지원 파일 포맷 및 템플릿 정의

시스템 초기화 및 대량 업데이트를 위해 CSV/Excel 파일 업로드를 지원합니다. 기본 포맷은 **CSV (UTF-8)**로 제한합니다.

### 마스터별 템플릿 규격

#### A. 제품 마스터 (Product Master)
| 영문 컬럼명 | 한국어 설명 | 데이터 타입 | 필수 여부 | 유효성 규칙 |
| :--- | :--- | :--- | :---: | :--- |
| `product_code` | 제품코드 | String | Y | 영문대문자+숫자, 고유값 |
| `product_name` | 제품명 | String | Y | 최대 100자 |
| `specification` | 규격 | String | N | - |
| `unit` | 단위 | String | Y | EA, BOX, KG, M 등 |
| `client_code` | 고객사코드 | String | N | 영문대문자+숫자 |
| `is_active` | 활성여부 | Boolean | Y | TRUE / FALSE (기본: TRUE) |

#### B. 공정 마스터 (Process Master)
| 영문 컬럼명 | 한국어 설명 | 데이터 타입 | 필수 여부 | 유효성 규칙 |
| :--- | :--- | :--- | :---: | :--- |
| `process_code` | 공정코드 | String | Y | 영문대문자+숫자, 고유값 |
| `process_name` | 공정명 | String | Y | 최대 100자 |
| `line_code` | 담당라인 | String | Y | 사전에 정의된 라인코드 일치 |
| `cycle_time_sec` | 사이클타임(초) | Number | Y | 0 초과 양수 정수 |
| `defect_codes` | 불량유형코드 목록 | String | N | 파이프(`\|`) 또는 쉼표(`,`) 분리 |

#### C. BOM (Bill of Materials)
| 영문 컬럼명 | 한국어 설명 | 데이터 타입 | 필수 여부 | 유효성 규칙 |
| :--- | :--- | :--- | :---: | :--- |
| `parent_item_code` | 상위제품코드 | String | Y | 제품 마스터에 존재하는 코드 |
| `child_item_code` | 하위부품코드 | String | Y | 제품 마스터에 존재하는 코드 |
| `quantity` | 수량 | Number | Y | 0 초과 양수 (소수점 허용) |
| `unit` | 단위 | String | Y | EA, KG 등 |
| `valid_from` | 유효시작일 | Date | Y | YYYY-MM-DD 형식 |
| `valid_to` | 유효종료일 | Date | N | YYYY-MM-DD 형식, valid_from 이후일 것 |

#### D. 불량 유형 (Defect Master)
| 영문 컬럼명 | 한국어 설명 | 데이터 타입 | 필수 여부 | 유효성 규칙 |
| :--- | :--- | :--- | :---: | :--- |
| `defect_code` | 불량코드 | String | Y | 영문대문자+숫자, 고유값 |
| `defect_name` | 불량명 | String | Y | 최대 100자 |
| `category` | 카테고리(4M) | String | Y | MAN, MACHINE, MATERIAL, METHOD 중 1 |
| `severity` | 심각도 | String | Y | S, A, B, C 중 1 |

---

## 2. 파일 업로드 API Route Handler 명세

### `POST /api/admin/bulk-import`
관리자가 파일을 업로드하고 처리 비동기 큐에 등록하는 엔드포인트입니다.

- **Request Body**: `multipart/form-data`
  - `file`: 업로드할 CSV 파일
  - `import_type`: `product` | `process` | `bom` | `defect`
- **Zod 검증**:
  - 파일 크기 ≤ 10MB
  - MIME 타입: `text/csv`, `application/vnd.ms-excel`
- **처리 흐름**:
  1. 권한 확인 (Admin Only).
  2. Supabase Storage의 `bulk_imports` 버킷에 파일 업로드 (경로: `tenant_id/import_type/timestamp.csv`).
  3. DB의 `bulk_import_jobs` 테이블에 Job 레코드 생성 (상태: `queued`).
- **Response**:
  ```json
  {
    "success": true,
    "data": {
      "job_id": "uuid-1234",
      "status": "queued",
      "total_rows": 0 // 초기 생성 시점엔 0
    }
  }
  ```

---

## 3. CSV 파싱 및 검증 엔진

### A. 사용 라이브러리 선택
- **라이브러리**: `papaparse`
- **선택 근거**: 브라우저와 Node.js(Vercel Edge 포함) 환경 모두에서 빠르고 일관되게 작동하며, 대용량 파일 스트리밍 파싱을 네이티브로 지원하여 메모리 오버헤드를 줄일 수 있음.

### B. Row 단위 검증 로직
1. **필수 컬럼 검사**: 템플릿 헤더(컬럼명) 존재 여부 확인.
2. **타입 형식 검사**: `Zod` 스키마를 활용하여 각 Row 단위로 타입 및 정규식 검사 수행.
3. **참조 무결성 검사**: BOM 업로드 시 `parent_item_code`가 제품 마스터 테이블에 실재하는지 Supabase 쿼리로 확인 (벌크 조회를 통해 캐싱 후 비교하여 DB 쿼리 최소화).

### C. 에러 행 수집 방식 (선택 및 근거)
- **방식**: 에러 행 스킵 후 계속 진행 (Skip & Continue).
- **근거**: 수천 건의 데이터 중 한 건의 오타로 인해 전체 롤백(전체 중단)이 발생하면 사용성 및 현장 작업 효율이 극도로 저하됨. 올바른 행은 모두 DB에 적재하고, 실패한 행만 별도로 수집하여 보고하는 것이 대규모 마스터 데이터 업로드에 적합.

---

## 4. 처리 결과 조회 API

### `GET /api/admin/bulk-import/{job_id}`
클라이언트가 Polling 방식을 통해 현재 업로드 진행률과 완료 결과를 조회합니다.

- **Response**:
  ```json
  {
    "success": true,
    "data": {
      "status": "completed", // queued, processing, completed, failed
      "processed_rows": 1000,
      "success_count": 995,
      "error_count": 5,
      "errors": [
        { "row": 12, "column": "client_code", "message": "필수 입력값이 누락되었습니다." },
        { "row": 150, "column": "parent_item_code", "message": "존재하지 않는 제품코드입니다." }
      ]
    }
  }
  ```

---

## 5. UI 명세 (T1-005 직접 대응)

### A. 업로드 화면 컴포넌트 구성
1. **마스터 선택**: 탭(Tab) 또는 드롭다운으로 업로드할 마스터 유형(제품, 공정, BOM 등) 선택.
2. **템플릿 다운로드**: 우측 상단 'CSV 템플릿 다운로드' 버튼 제공.
3. **DropZone (shadcn/ui 기반)**: 파일을 끌어다 놓거나 클릭하여 탐색기 오픈 (오직 .csv 확장자만 허용).
4. **상태 모니터링**:
   - 파일 첨부 후 '업로드 시작' 클릭.
   - 프로그레스 바(Progress Bar) 표시 및 Polling 방식으로 1~3초 주기로 `job_id` 조회.
5. **에러 테이블 노출**: 업로드 완료 후 `error_count > 0`일 때, 하단에 Data Table로 행번호/컬럼/사유 표시.

---

## 6. Insert-only Audit Log 연동

Bulk Import 작업 단위(성공이든 실패든)로 `audit_log` 테이블에 반드시 이력을 남깁니다.
- **적재 시점**: Job 상태가 `completed` 또는 `failed`로 변경되는 시점.
- **적재 데이터**:
  - `action_type`: `BULK_IMPORT`
  - `tenant_id`: 업로더 테넌트
  - `user_id`: 관리자 ID
  - `target_table`: `products` 등 타겟 엔티티
  - `status`: `PASS` (일부 에러가 있어도 Job 자체 처리가 끝나면 PASS) 또는 `FAIL` (서버 다운 등)
  - `details`: JSON 형태 `{ "filename": "bom_2026.csv", "total": 1000, "success": 995, "error": 5, "job_id": "..." }`

---

## 7. 에러 피드백 화면 완료 기준

업로드 작업 종료 후 UI는 다음 3가지 상태로 분기하여 사용자 경험을 완료합니다.

1. **전체 성공 (Success)**:
   - 화면: "총 1,000건의 데이터가 성공적으로 등록되었습니다." 요약 모달 표시.
   - 액션: 리스트 새로고침.
2. **부분 오류 (Partial Success)**:
   - 화면: "995건 등록 완료. 5건 오류 발생." 알림 표시.
   - 액션: Data Table 옆에 **[오류 데이터만 CSV 다운로드]** 버튼 노출. 사용자가 수정한 후 해당 파일만 다시 업로드할 수 있도록 유도.
3. **전체 실패 (Fatal Error)**:
   - 화면: "파일 인코딩 오류 또는 지원하지 않는 형식입니다." 경고 모달 표시.
   - 액션: 표준 템플릿 재다운로드 버튼 활성화.
