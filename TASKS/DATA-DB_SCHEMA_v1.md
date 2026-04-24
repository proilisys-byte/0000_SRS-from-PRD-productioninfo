# DATA-DB_SCHEMA_v1.md

## 1. 테이블 정의 (Supabase PostgreSQL 기준)

시스템의 근간이 되는 데이터베이스 스키마는 Multi-Tenancy와 Role-Based Access Control(RBAC), 그리고 무결성 보장을 위한 구조로 설계되었습니다.

| 테이블명 | 컬럼명 | 타입 | 제약조건 | 설명 |
| :--- | :--- | :--- | :--- | :--- |
| **`tenants`** | `id` | UUID | PK, DEFAULT uuid_generate_v4() | 테넌트(고객사) 고유 ID |
| | `name` | TEXT | NOT NULL | 고객사명 |
| | `created_at` | TIMESTAMPTZ | NOT NULL, DEFAULT NOW() | 생성일시 |
| **`rbac_roles`** | `id` | TEXT | PK | 권한 식별자 (예: 'admin', 'user') |
| | `description` | TEXT | | 권한 설명 |
| **`users`** | `id` | UUID | PK (Supabase Auth 연동) | 사용자 고유 ID |
| | `tenant_id` | UUID | FK (tenants.id), NOT NULL | 소속 고객사 |
| | `role_id` | TEXT | FK (rbac_roles.id), NOT NULL | 소속 권한 |
| | `email` | TEXT | NOT NULL, UNIQUE | 이메일 |
| | `created_at` | TIMESTAMPTZ | NOT NULL, DEFAULT NOW() | 생성일시 |
| **`audit_sessions`**| `id` | UUID | PK, DEFAULT uuid_generate_v4() | Audit 세션 ID |
| | `tenant_id` | UUID | FK (tenants.id), NOT NULL | 소속 고객사 |
| | `user_id` | UUID | FK (users.id), NOT NULL | 생성 사용자 |
| | `status` | TEXT | NOT NULL, DEFAULT 'IN_PROGRESS' | 세션 상태 |
| | `start_time` | TIMESTAMPTZ | NOT NULL, DEFAULT NOW() | 세션 시작일시 |
| | `end_time` | TIMESTAMPTZ | | 세션 종료일시 |
| **`audit_data_entries`** | `id` | UUID | PK, DEFAULT uuid_generate_v4() | 수집 데이터 엔트리 고유 ID |
| | `session_id` | UUID | FK (audit_sessions.id), NOT NULL | 연관된 세션 |
| | `raw_data` | JSONB | NOT NULL | STT에서 변환된 원본 데이터 |
| | `mapped_data` | JSONB | | ISO 양식 등에 맞춰 매핑된 데이터 |
| | `created_at` | TIMESTAMPTZ | NOT NULL, DEFAULT NOW() | 생성일시 |
| **`audit_log`** <br/>*(Insert-only)* | `id` | UUID | PK, DEFAULT uuid_generate_v4() | 로그 고유 ID |
| | `table_name` | TEXT | NOT NULL | 변경이 발생한 테이블명 |
| | `record_id` | UUID | NOT NULL | 변경된 레코드의 PK |
| | `action` | TEXT | NOT NULL | INSERT / UPDATE / DELETE |
| | `old_data` | JSONB | | 변경 전 데이터 |
| | `new_data` | JSONB | | 변경 후 데이터 |
| | `changed_by` | UUID | FK (users.id) | 변경 주체(사용자) |
| | `created_at` | TIMESTAMPTZ | NOT NULL, DEFAULT NOW() | 기록 발생일시 |
| **`bulk_import_batches`**| `id` | UUID | PK, DEFAULT uuid_generate_v4() | 배치 작업 고유 ID |
| | `tenant_id` | UUID | FK (tenants.id), NOT NULL | 소속 고객사 |
| | `uploaded_by` | UUID | FK (users.id), NOT NULL | 업로드 담당자 |
| | `file_name` | TEXT | NOT NULL | 업로드된 CSV 파일명 |
| | `status` | TEXT | NOT NULL, DEFAULT 'PENDING' | 처리 상태 (PENDING/DONE/FAIL) |
| | `total_rows` | INT | NOT NULL, DEFAULT 0 | 전체 처리 대상 행 수 |
| | `processed_rows`| INT | NOT NULL, DEFAULT 0 | 현재 처리된 행 수 |
| | `created_at` | TIMESTAMPTZ | NOT NULL, DEFAULT NOW() | 생성일시 |


## 2. Insert-only Audit Log 무결성 보장 전략

가장 강력한 데이터 무결성 보장을 위해 애플리케이션 계층(Prisma)의 통제를 넘어, PostgreSQL 데이터베이스 레벨에 트리거와 RLS(Row Level Security)를 결합하여 `audit_log` 테이블의 수정/삭제를 원천 차단합니다. 

```sql
-- 1. UPDATE 및 DELETE 차단을 위한 트리거 함수 정의
CREATE OR REPLACE FUNCTION prevent_audit_log_modification()
RETURNS TRIGGER AS $$
BEGIN
  RAISE EXCEPTION 'COMPLIANCE LOCKUP: Modification or deletion of audit_log records is strictly prohibited.';
END;
$$ LANGUAGE plpgsql;

-- 2. 트리거 부착
CREATE TRIGGER trg_prevent_audit_log_update_delete
BEFORE UPDATE OR DELETE ON audit_log
FOR EACH ROW
EXECUTE FUNCTION prevent_audit_log_modification();

-- 3. Insert-only 강제를 위한 RLS 정책 설정
ALTER TABLE audit_log ENABLE ROW LEVEL SECURITY;

-- Service Role 및 권한 있는 시스템/사용자만 INSERT 허용
CREATE POLICY "Allow insert for authenticated users" 
ON audit_log 
FOR INSERT TO authenticated 
WITH CHECK (true);

-- 조회는 허용하되 (관리자용 등), UPDATE/DELETE 정책은 절대 생성하지 않음
CREATE POLICY "Allow select for tenant admins" 
ON audit_log 
FOR SELECT TO authenticated 
USING (
  -- [검토 필요: 실제 tenant_id 조인 기반 격리 로직 반영 필요]
  true 
);
```

## 3. Supabase RLS vs Prisma Extension 충돌 해소 방안

Prisma Client는 기본적으로 Database Connection Pool을 통해 모든 쿼리를 실행하므로, 브라우저/클라이언트의 JWT를 기반으로 작동하는 Supabase RLS 정책과 구조적 충돌이 발생할 수 있습니다. 이를 우회하고 권한을 분리하기 위한 방안은 다음과 같습니다.

* **권한 분리 방식**:
  * **User JWT 토큰**: 클라이언트에서 직접 Supabase Edge/Storage에 접근하거나, Next.js Server Actions에서 유저 식별을 할 때 사용합니다.
  * **Service Role Key**: Next.js Server 내부에서 작동하는 Prisma Client는 `Service Role Key`가 적용된 Connection String을 사용하여 RLS를 우회(Bypass)합니다. 

* **Insert-only 정책 접근 구조**:
  * Prisma Client는 Service Role로 연결되어 DB 레벨의 읽기/쓰기를 자유롭게 수행하지만, **`audit_log` 테이블에 걸린 UPDATE/DELETE 트리거는 Service Role이라 할지라도 강제로 예외(Exception)를 발생**시켜 데이터를 보호합니다.
  * 권한 통제는 Prisma Extension 미들웨어에서 `update`, `delete` 쿼리 시도를 애플리케이션 계층에서 1차 차단하고, 뚫리더라도 DB 트리거가 2차로 차단하는 **Defense-in-depth(심층 방어)** 구조를 채택합니다.


## 4. SQLite(로컬) ↔ PostgreSQL(Supabase) 이식성 전략

초기 로컬 개발 편의성(SQLite)과 프로덕션 환경(PostgreSQL) 간의 호환성을 유지하기 위한 전략입니다.

* **타입 불일치 대응 방법**:
  * **Boolean & DateTime**: Prisma가 SQLite와 PostgreSQL의 차이를 추상화해주므로 표준 `Boolean`, `DateTime` 타입을 그대로 사용합니다.
  * **JSON 타입**: SQLite는 네이티브 JSON을 미지원하므로 `String` 텍스트로 저장해야 합니다. 이를 극복하기 위해 `prisma-json-types-plugin`을 적용합니다.

* **`prisma-json-types-plugin` 적용 판단 및 이유**:
  * **적용 필수**: SQLite 환경에서 `String`으로 저장된 데이터를 코드 상에서는 강력한 타입의 TypeScript 객체로 직렬화/역직렬화하기 위해 필수적으로 적용합니다.

* **Prisma schema.prisma 예시 코드**:
```prisma
datasource db {
  // 환경 변수에 따라 provider 동적 적용 (Prisma ^4.0 이상 지원 기능 활용, 미지원 시 빌드 스크립트로 분기 처리)
  provider = env("DATABASE_PROVIDER") // "sqlite" 또는 "postgresql"
  url      = env("DATABASE_URL")
}

generator client {
  provider = "prisma-client-js"
}

generator json {
  provider = "prisma-json-types-generator"
}

model audit_data_entries {
  id          String   @id @default(uuid())
  session_id  String
  
  /// [AuditMappedData] 
  mapped_data String   // SQLite 호환을 위해 Json 대신 String 타입 선언 후 플러그인 주석 처리
  
  created_at  DateTime @default(now())
}
```
*(참고: `[검토 필요: Prisma 버전에 따른 동적 provider 지원 여부]` - Prisma 최신 버전에서는 다중 provider 지정 기능이 제거되었으므로, 실무에서는 로컬에서도 Docker 기반 PostgreSQL을 사용하는 것이 가장 강력한 이식성 전략입니다.)*

## 5. 인덱스 전략

조회 성능을 보장하기 위해 B-Tree 기반의 인덱스를 선언합니다.

1. **`users` 테이블**: `CREATE INDEX idx_users_tenant ON users(tenant_id, email);` (테넌트 별 유저 조회)
2. **`audit_sessions` 테이블**: `CREATE INDEX idx_sessions_tenant_date ON audit_sessions(tenant_id, created_at DESC);` (세션 리스트 최근순 정렬 조회)
3. **`audit_data_entries` 테이블**: `CREATE INDEX idx_entries_session ON audit_data_entries(session_id);` (특정 세션의 데이터 일괄 로드)
4. **`audit_log` 테이블**: `CREATE INDEX idx_auditlog_record ON audit_log(table_name, record_id);` (특정 레코드의 이력 추적)
