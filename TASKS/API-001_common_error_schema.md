# API-001_common_error_schema: 공통 에러 스키마 명세

## 1. 문서 목적

* **필요성**: PRO ILI SMART 플랫폼에서 발생하는 모든 API, Server Action, 비동기 작업의 에러 응답을 표준화하여, 프론트엔드, 백엔드, QA, 운영팀이 동일한 언어와 구조로 오류를 해석하고 대응할 수 있도록 한다.
* **Sprint 1 선행 필요성**: Auth, RBAC, Bulk Import, STT, Audit 등 핵심 모듈이 동시다발적으로 개발되는 Sprint 1에서 에러 규격이 파편화되면 디버깅과 프론트엔드 연동에 막대한 병목이 발생하므로 최우선으로 확정해야 한다.
* **후속 문서 기준**: 본 문서는 향후 작성될 모든 개별 API 명세서(`API-AUDIT_REPORT`, `API-BULK_IMPORT`, `API-STT_CAPTURE` 등)의 에러 응답 스펙 상위 기준으로 작용한다.

---

## 2. 공통 에러 설계 원칙

| 원칙 | 설명 | 적용 이유 | 관련 문서 | Sprint 우선순위 |
|---|---|---|---|---|
| **일관성** | 모든 에러 응답은 동일한 JSON 구조(`success`, `error` 객체 등)를 갖는다. | 프론트엔드의 공통 에러 핸들러 구축 용이 | 전체 아키텍처 | Sprint 1 |
| **보안 정보 최소 노출** | 스택 트레이스, DB 쿼리, 내부 인프라 구조 등은 절대 사용자 응답으로 반환하지 않는다. | 시스템 취약점 노출 방지 | `COM-AUTH_v1.md` | Sprint 1 |
| **예측 가능성** | HTTP 상태 코드와 비즈니스 에러 코드를 분리하되, 상호 맵핑 규칙을 엄격히 준수한다. | 클라이언트 측의 분기 처리 및 예외 상황 대응력 강화 | - | Sprint 1 |
| **운영 추적 가능성** | 모든 에러 응답에 고유한 `trace_id`를 부여하여 Audit Log 및 서버 로그와 연결한다. | CS 인입 시 빠른 원인 파악 및 규제/감사 대응 | `DATA-AUDIT_LOG_v1.md` | Sprint 1 |
| **사용자 안전성** | 사용자에게는 "무엇이 문제인지, 어떻게 해야 하는지" 행동 지향적인 메시지만 노출한다. | 사용자 혼란 방지 및 운영팀 문의 인입 감소 | `00_PRD_v1.md` | Sprint 1 |
| **테스트 가능성** | 에러 코드를 문자열 키로 제공하여 QA 자동화 스크립트에서 Assertion이 가능하게 한다. | E2E/API 테스트 자동화 시 문자열(메시지) 변경에 따른 테스트 깨짐 방지 | 향후 TEST 문서 | Sprint 1 |
| **재시도 판단 가능성**| 타임아웃, 네트워크 오류 등 일시적 장애와 권한/입력 오류 등 영구적 실패를 구분할 수 있게 한다. | 프론트엔드의 재시도 팝업, 백엔드 큐의 자동 재시도 판단 기준 제공 | - | Sprint 1 |

---

## 3. 적용 범위 정의

| 대상 | 설명 | Sprint 적용 시점 | 적용 여부 | 비고 |
|---|---|---|:---:|---|
| **REST API 응답** | 클라이언트-서버 간 모든 동기식 HTTP JSON 응답 | Sprint 1 | O | 전면 적용 |
| **Server Action 결과**| Next.js 환경 등에서의 백엔드 액션 응답 규격 | Sprint 1 | O | API 응답과 동일 구조 차용 |
| **비동기 Job 상태** | Bulk Import, Report 생성 등 비동기 폴링 결과의 실패 상세 내역 | Sprint 1 | O | DB Status 필드에 JSON 직렬화 저장 |
| **외부 연동 오류** | STT, 향후 ERP 연동 등 외부 API 실패 시 내부 래핑 응답 | Sprint 1 | O | 외부 에러 원문을 내부 에러 코드로 감싸서 반환 |
| **인프라 레벨 장애** | 502/504 등 API Gateway/LB 단의 에러 응답 | Sprint 1 | 세미-적용 | 인프라 설정으로 최대한 유사한 구조 반환하도록 설정 |

---

## 4. 공통 에러 응답 구조 및 스키마

PRO ILI SMART 플랫폼의 모든 백엔드 응답(Route Handler 및 Server Actions)은 예측 가능성과 클라이언트 측의 일관된 에러 처리를 보장하기 위해 아래의 Response Envelope 규격을 따릅니다.

### 4.1. 공통 에러 응답 필드 정의

| 필드명 | 설명 | 타입 | 필수 여부 | 사용자 노출 여부 | 예시 | 비고 |
|---|---|---|:---:|:---:|---|---|
| `success` | 요청의 성공 여부 (에러 시 항상 `false`) | `boolean` | 필수 | X (로직용) | `false` | - |
| `error` | 에러 상세 정보를 담는 객체 래퍼 | `object` | 필수 | - | - | 하위 필드 포함 |
| `error.code` | 비즈니스 에러 코드 (문자열) | `string` | 필수 | X (로직용) | `"AUTH_401_EXPIRED"` | 다국어/프론트엔드 분기 키 |
| `error.message` | 사용자 친화적인 에러 설명 메시지 | `string` | 필수 | O | `"로그인 세션이 만료되었습니다. 다시 로그인해주세요."`| 그대로 UI 토스트에 노출 가능해야 함 |
| `error.details` | 폼 입력 에러 등 필드 단위 상세 내역 (옵션) | `array` | 선택 | O (폼 에러 표시용)| `[{"field": "email", "issue": "형식 오류"}]` | Validation 실패 시 필수 |
| `error.trace_id` | 서버에서 생성한 해당 요청/에러의 고유 추적 ID | `string` | 필수 | O (에러 화면 표시) | `"tr_123456789"` | 고객센터 문의 시 첨부 유도 |
| `error.timestamp` | 에러 발생 서버 시간 (ISO 8601) | `string` | 필수 | X (로그용) | `"2026-04-24T15:20:00Z"` | - |

**[JSON 에러 응답 예시]**
```json
{
  "success": false,
  "error": {
    "code": "VAL_400_INVALID_FIELD",
    "message": "입력하신 정보에 오류가 있습니다. 붉은색으로 표시된 항목을 확인해주세요.",
    "details": [
      { "field": "template_id", "issue": "필수 항목입니다." }
    ],
    "trace_id": "tr_09a8b7c6d5",
    "timestamp": "2026-04-24T15:20:00Z"
  }
}
```

### 4.2. Zod 스키마 정의 (`src/lib/validations/api-response.ts`)
```typescript
import { z } from 'zod';

export const ErrorResponseSchema = z.object({
  success: z.literal(false),
  error: z.object({
    code: z.string(),
    message: z.string(),
    field: z.string().optional(),
    details: z.array(z.any()).optional(),
    trace_id: z.string().optional(),
    timestamp: z.string().optional(),
  }),
});

export const SuccessResponseSchema = <T extends z.ZodTypeAny>(dataSchema: T) =>
  z.object({
    success: z.literal(true),
    data: dataSchema,
    meta: z.record(z.any()).optional(),
  });

export type ApiErrorResponse = z.infer<typeof ErrorResponseSchema>;
export type ApiSuccessResponse<T> = {
  success: true;
  data: T;
  meta?: Record<string, any>;
};
```

---

## 5. 에러 분류 체계 및 상태 코드

### 5.1. 에러 분류 체계

| 에러 카테고리 | 설명 | 대표 상황 | HTTP 상태코드 | 로그 중요도 |
|---|---|---|:---:|:---:|
| **Authentication (AUTH)** | 사용자 식별 불가 | 토큰 만료, 토큰 없음, 로그인 실패 | 401 | Medium |
| **Authorization (FORB)** | 사용자 식별은 되나 권한 부족 | Tenant 침범 시도, Admin 기능 일반 접근 | 403 | **High** |
| **Validation (VAL)** | 클라이언트 입력값 형식/필수값 오류 | 이메일 형식 오류, 빈 파일 업로드 | 400 | Low |
| **Business Rule (BIZ)** | 비즈니스 로직 및 정책 위반 | 반려된 Audit 수정 시도, 중복 제출 | 422 / 400 | Medium |
| **Not Found (NOTF)** | 대상 리소스가 존재하지 않음 | 삭제된 템플릿 조회, 잘못된 ID 파라미터 | 404 | Low |
| **Conflict (CONF)** | 동시성 문제 또는 상태 충돌 | 이미 승인된 리포트 중복 승인 시도 | 409 | Medium |
| **External (EXT)** | 외부 시스템/API 통신 실패 | STT 엔진 타임아웃, 결제 연동 장애 | 502 / 504 | High |
| **Rate Limit (RATE)** | 허용된 요청 횟수 초과 | 초당 API 호출 제한 초과 (DDoS 방어) | 429 | High |
| **System/Internal (SYS)** | 백엔드 인프라/DB/코드 장애 | DB 커넥션 풀 부족, NullReference | 500 | **Critical**|

### 5.2. 에러 코드 네이밍 규칙

| 규칙 항목 | 설명 | 예시 |
|---|---|---|
| **도메인 접두어** | 오류가 발생한 분류 카테고리 대문자 (3~4자) | `AUTH_`, `VAL_`, `BIZ_` |
| **HTTP 매핑 번호** | 도메인 접두어 뒤에 해당 에러의 HTTP 상태코드 포함 | `AUTH_401_`, `BIZ_422_` |
| **상세 명칭** | 오류의 구체적 원인을 영문 대문자 스네이크 케이스로 작성 | `_TOKEN_EXPIRED` |

*(예: `AUTH_401_TOKEN_EXPIRED`, `FORB_403_TENANT_MISMATCH`, `EXT_502_STT_TIMEOUT`)*

### 5.3. 에러 코드 전체 정의표 (예시)

| 카테고리 | HTTP 상태 | 코드명 (snake_case) | 메시지 | 발생 시나리오 |
| :--- | :---: | :--- | :--- | :--- |
| **AUTH** | 401 | `AUTH_401_UNAUTHORIZED_ACCESS` | "인증 토큰이 누락되었거나 유효하지 않습니다." | 로그인 미인증 접근, 토큰 만료 |
| **AUTH** | 403 | `FORB_403_INSUFFICIENT_PERMISSIONS` | "해당 작업을 수행할 권한이 없습니다." | 일반 사용자의 Admin 전용 엔드포인트 접근 |
| **AUTH** | 403 | `FORB_403_MFA_REQUIRED` | "다중 인증(MFA) 설정이 필요합니다." | MFA 미완료 관리자의 대시보드 접근 시도 |
| **VALIDATION** | 400 | `VAL_400_VALIDATION_FAILED` | "입력 데이터 형식이 올바르지 않습니다." | Zod 파싱 실패, 필수 필드 누락 |
| **BUSINESS** | 409 | `CONF_409_RESOURCE_CONFLICT` | "이미 존재하는 데이터입니다." | 제품코드/공정코드 중복 등록 시도 |
| **BUSINESS** | 422 | `BIZ_422_INVALID_STATE_TRANSITION` | "현재 상태에서는 해당 작업을 수행할 수 없습니다." | COMPLETED 세션에 데이터 추가 시도 |
| **BUSINESS** | 403 | `BIZ_403_LOCKUP_UNMET` | "컴플라이언스 락업 조건이 충족되지 않았습니다." | PIPA 미동의 상태에서 STT 진입 시도 |
| **EXTERNAL** | 429 | `RATE_429_EXTERNAL_RATE_LIMIT` | "외부 서비스 요청 한도를 초과했습니다." | Gemini API RPM 초과 (429) |
| **EXTERNAL** | 502 | `EXT_502_EXTERNAL_SERVICE_ERROR` | "외부 서비스 연결에 실패했습니다." | Gemini API 500 오류 또는 Supabase 장애 |
| **FILE** | 400 | `VAL_400_INVALID_FILE_FORMAT` | "지원하지 않는 파일 형식입니다." | CSV/Excel 외 파일 업로드, 잘못된 템플릿 |
| **FILE** | 413 | `VAL_413_PAYLOAD_TOO_LARGE` | "파일 크기가 허용치를 초과했습니다." | 오디오/CSV 파일 10MB/20MB 초과 |
| **SYSTEM** | 500 | `SYS_500_INTERNAL_SERVER_ERROR` | "서버 내부 오류가 발생했습니다." | 예기치 못한 코드 레벨 예외 발생 |
| **SYSTEM** | 504 | `EXT_504_GATEWAY_TIMEOUT` | "작업 처리 시간이 초과되었습니다." | Vercel Serverless Function 60초 타임아웃 |

### 5.4. HTTP 상태 코드 매핑 기준

| HTTP 상태코드 | 의미 | 대표 비즈니스 코드 예시 | 프론트엔드 처리 방향 |
|:---:|---|---|---|
| **400** | Bad Request | `VAL_400_INVALID_FIELD` | 폼 에러 하이라이팅, 토스트 알림 |
| **401** | Unauthorized | `AUTH_401_EXPIRED` | 즉시 로그아웃 처리 및 로그인 페이지 리다이렉트 |
| **403** | Forbidden | `FORB_403_TENANT_MISMATCH` | "권한 없음" 에러 페이지 노출, 뒤로 가기 |
| **404** | Not Found | `NOTF_404_RESOURCE_NOT_FOUND` | 404 페이지 노출 또는 빈 상태 UI |
| **409** | Conflict | `CONF_409_ALREADY_APPROVED` | "이미 처리되었습니다" 메시지 노출 후 목록 새로고침 |
| **422** | Unprocessable Entity | `BIZ_422_INVALID_TRANSITION` | 원인 설명 후 사용자의 논리적 수정 유도 |
| **429** | Too Many Requests | `RATE_429_TOO_MANY_REQUESTS` | 재시도 타이머(백오프) 노출 |
| **500** | Internal Server Error | `SYS_500_UNEXPECTED` | 에러 바운더리 노출, `trace_id`를 포함한 "관리자 문의" 유도 |
| **502 / 504** | Bad Gateway / Timeout | `EXT_504_STT_TIMEOUT` | 외부 시스템 장애 알림 및 재시도 버튼 제공 |

---

## 6. 메시지 처리 및 로깅 정책

### 6.1. 사용자 메시지 vs 내부 메시지 분리 기준

| 구분 | 목적 | 포함 가능한 정보 | 포함 금지 정보 (보안 원칙) |
|---|---|---|---|
| **사용자 메시지** (`error.message`) | 사용자에게 상황을 알리고 다음 행동(재시도, 수정, 대기 등)을 안내 | 어떤 동작이 안 되었는지, 무엇을 수정해야 하는지 | DB 테이블명, SQL 문법, 널 포인터 에러, 서버 IP, 사내망 주소 |
| **내부 운영 메시지** (Server Log, Datadog 등) | 개발자 및 운영자의 디버깅 및 원인 추적 | 스택 트레이스, 관련 사용자/Tenant ID, 파라미터 원문 | 개인 식별 정보 원문 (PIPA), 패스워드 평문, 토큰 원문 |

### 6.2. 감사 로그 및 추적 연계

* **`trace_id` 무조건 응답**: 클라이언트가 에러 조우 시 화면에 노출하여, CS 인입 시 백엔드 로그와 즉시 매칭 가능토록 함.
* **AuditLog 연계 1 (권한 위반)**: 403 (FORB) 에러 발생 시, `DATA-AUDIT_LOG` 테이블에 `SECURITY_VIOLATION` 이벤트로 적재 (PIPA/ISO 심사 대응).
* **AuditLog 연계 2 (고위험 실패)**: Admin의 Bulk Import, 권한 변경 등 고위험 작업 실패 시 원인과 함께 `FAILED` 로깅.
* **로깅 민감 정보 마스킹**: 내부 로깅 시에도 사용자의 패스워드, 토큰 원문 등은 마스킹 처리.
* **로직**: `handleRouteError` 내에서 예외가 `SYSTEM` 카테고리(500, 502, 504)이거나 심각한 보안 침해(AUTH-403)인 경우, 응답 반환과 무관하게 `waitUntil()` 또는 백그라운드 태스크를 활용하여 `audit_log` 테이블에 에러 로그를 비동기 Insert 합니다.

### 6.3. 공통 에러 핸들러 유틸리티 코드 (`src/lib/errors.ts`)

```typescript
import { z } from 'zod';
import { NextResponse } from 'next/server';

export class AppError extends Error {
  constructor(
    public code: string,
    public message: string,
    public statusCode: number = 400,
    public field?: string,
    public details?: any[]
  ) {
    super(message);
    this.name = 'AppError';
  }
}

function generateTraceId() {
  return `tr_${Math.random().toString(36).substring(2, 11)}`;
}

export function handleRouteError(error: unknown) {
  const traceId = generateTraceId();
  const timestamp = new Date().toISOString();

  // 1. 커스텀 비즈니스 에러
  if (error instanceof AppError) {
    // 권한 오류 시 감사 로깅 추가
    if (error.statusCode === 403) {
        console.warn(`[SECURITY_VIOLATION] trace: ${traceId}`, error);
        // TODO: audit_log 비동기 기록 로직 추가
    }
    return NextResponse.json({
      success: false,
      error: {
        code: error.code,
        message: error.message,
        field: error.field,
        details: error.details,
        trace_id: traceId,
        timestamp
      }
    }, { status: error.statusCode });
  }

  // 2. Zod 검증 에러
  if (error instanceof z.ZodError) {
    return NextResponse.json({
      success: false,
      error: {
        code: 'VAL_400_VALIDATION_FAILED',
        message: '입력 데이터 형식이 올바르지 않습니다. 붉은색으로 표시된 항목을 확인해주세요.',
        details: error.errors,
        trace_id: traceId,
        timestamp
      }
    }, { status: 400 });
  }

  // 3. 외부 API 429 에러 대응
  if (typeof error === 'object' && error !== null && 'status' in error) {
    const apiError = error as any;
    if (apiError.status === 429) {
      return NextResponse.json({
        success: false,
        error: { 
            code: 'RATE_429_EXTERNAL_RATE_LIMIT', 
            message: '외부 서비스 요청 한도를 초과했습니다.',
            trace_id: traceId,
            timestamp
        }
      }, { status: 429 });
    }
  }

  // 4. 기타 서버 내부 에러 (SYSTEM 로깅 연동)
  console.error(`[SYSTEM_ERROR] trace: ${traceId}`, error);
  // TODO: audit_log 비동기 기록 로직 추가
  
  return NextResponse.json({
    success: false,
    error: {
      code: 'SYS_500_INTERNAL_SERVER_ERROR',
      message: '서버 내부 오류가 발생했습니다. 지속될 경우 관리자에게 문의하세요.',
      trace_id: traceId,
      timestamp
    }
  }, { status: 500 });
}
```

---

## 7. 기능별 대표 에러 시나리오 및 재시도 정책

### 7.1. 기능별 대표 에러 시나리오

| 기능 영역 | 시나리오 | 에러 카테고리 | 사용자 메시지 방향 | 운영 대응 |
|---|---|---|---|---|
| **COM-AUTH** | 타 Tenant 리소스 접근 시도 | `FORB_403` | "해당 자원에 대한 접근 권한이 없습니다." | 비정상 접근 시도 모니터링 알람 (보안 위반) |
| **ADM-BULK** | CSV 포맷 위반 또는 필수 컬럼 누락 | `VAL_400` | "파일 형식이 잘못되었습니다. 템플릿을 확인해주세요." | 정상적인 사용자 에러, 모니터링 제외 |
| **ADM-BULK** | 대용량 파싱 중 메모리 부족 장애 | `SYS_500` | "시스템 처리 중 오류가 발생했습니다." | 인프라 경고, 컨테이너 스케일업 검토 |
| **F1-AUDIT** | 필수 점검 항목 누락 후 Review 제출 | `BIZ_422` | "모든 필수 점검 항목을 완료해야 제출할 수 있습니다." | 정상적인 로직 차단 |
| **STT 연동** | STT API 서버 응답 지연 (Timeout) | `EXT_504` | "음성 인식 서버 응답이 지연되고 있습니다. 수동으로 입력해주세요." | 외부 서비스 헬스체크 |
| **Report** | 승인되지 않은 Audit의 리포트 다운로드 시도 | `BIZ_422` / `FORB_403` | "최종 승인된 리포트만 다운로드할 수 있습니다." | 비정상 요청 패턴 감지 |

### 7.2. 재시도 및 복구 정책

| 오류 유형 | 사용자 재시도 가능 여부 | 시스템 자동 재시도 여부 | 운영자 개입 필요 여부 | UI 안내 방식 |
|---|:---:|:---:|:---:|---|
| **입력값/비즈니스 오류 (400, 422)** | O (수정 후) | X | X | 붉은색 필드 강조, 에러 토스트 |
| **권한/인증 오류 (401, 403)** | O (로그인 후) | X | X (Admin 문의 유도) | 접근 차단 페이지, 관리자 문의 안내 |
| **외부 통신 지연 (502, 504)** | O (즉시) | O (비동기 큐 한정, 최대 3회) | X | "일시적 지연. 잠시 후 재시도" 알림 |
| **시스템 장애 (500)** | X | X | **O (긴급)** | 에러 바운더리, Trace ID 복사 버튼 제공 |

---

## 8. 구현 패턴 가이드

### 8.1. Route Handler 적용 패턴

```typescript
// app/api/example/route.ts
import { NextRequest, NextResponse } from 'next/server';
import { handleRouteError, AppError } from '@/lib/errors';
import { z } from 'zod';

const RequestSchema = z.object({
  title: z.string().min(1),
});

export async function POST(req: NextRequest) {
  try {
    const body = await req.json();
    const data = RequestSchema.parse(body);

    if (data.title === 'restricted') {
      throw new AppError('BIZ_422_INVALID_STATE_TRANSITION', '허용되지 않은 타이틀입니다.', 422);
    }

    return NextResponse.json({ success: true, data: { id: 1, ...data } });
  } catch (error) {
    return handleRouteError(error);
  }
}
```

### 8.2. Server Action 적용 패턴

Server Action은 `NextResponse` 대신 평문 객체를 리턴해야 `useFormState` 등과 정상 연동됩니다.

```typescript
// app/actions/exampleAction.ts
'use server';

import { z } from 'zod';
import { AppError } from '@/lib/errors';

export async function exampleAction(prevState: any, formData: FormData) {
  try {
    const title = formData.get('title');
    if (!title) throw new AppError('VAL_400_VALIDATION_FAILED', '타이틀이 필요합니다.', 400);

    return { success: true, data: { title } };
  } catch (error) {
    const traceId = `tr_${Math.random().toString(36).substring(2, 11)}`;
    const timestamp = new Date().toISOString();
    
    if (error instanceof AppError) {
      return { success: false, error: { code: error.code, message: error.message, trace_id: traceId, timestamp } };
    }
    return { success: false, error: { code: 'SYS_500_INTERNAL_SERVER_ERROR', message: '오류가 발생했습니다.', trace_id: traceId, timestamp } };
  }
}
```

### 8.3. FE 클라이언트 에러 처리 가이드 (`apiClient` 및 Toast)

```typescript
// lib/apiClient.ts
export async function apiClient<T>(url: string, options?: RequestInit): Promise<T> {
  const res = await fetch(url, options);
  const json = await res.json();

  if (!res.ok || !json.success) {
    const errorMsg = json.error?.message || '알 수 없는 오류가 발생했습니다.';
    const errorCode = json.error?.code || 'UNKNOWN_ERROR';
    const traceId = json.error?.trace_id || 'UNKNOWN_TRACE';
    // 에러를 던져 클라이언트 측 catch 블록이나 ErrorBoundary에서 잡도록 함
    throw new Error(`[${errorCode}] ${errorMsg} (Trace ID: ${traceId})`);
  }

  return json.data;
}
```

```tsx
// React 성분 예시
import { toast } from 'sonner';
import { apiClient } from '@/lib/apiClient';

async function handleSubmit() {
  try {
    await apiClient('/api/example', { method: 'POST', body: JSON.stringify({...}) });
    toast.success('저장되었습니다.');
  } catch (error: any) {
    // 공통 파싱된 에러 메시지를 토스트로 바로 노출
    // 에러 메시지는 'error.message' 포맷에 맞춰 출력
    toast.error(error.message);
  }
}
```

---

## 9. 프론트엔드 / QA / 운영 영향 및 우선순위

### 9.1. 각 파트별 영향
* **프론트엔드**: `error.code`를 기준으로 다국어 처리를 수행하며, `error.message`는 Fallback 용도로 사용한다. 500 에러 발생 시 반드시 글로벌 에러 바운더리에서 `trace_id`를 화면에 표기하여 캡처를 유도한다.
* **QA**: API/E2E 테스트 시나리오는 HTTP Status 확인과 더불어 `error.code` 문자열을 Assert의 핵심 조건으로 활용하여 테스트의 회복탄력성(Resilience)을 높인다.
* **운영 (Ops)**: Datadog/Sentry 등의 모니터링 도구에서 4xx 에러(특히 400, 404, 422)는 경고(Warn) 수준으로, 5xx 에러 및 403 권한 에러는 심각도 높음(Error/Critical)으로 분류하여 알림(Alert) 룰을 분리 설정한다.

### 9.2. Sprint 우선순위

| 기능/규칙 | Sprint | 이유 | 선행조건 |
|---|---|---|---|
| **기본 응답 스키마(코드, 메시지, Trace ID) 확정** | Sprint 1 | FE/BE 병렬 개발 시 규격 합의 필수 | - |
| **권한(401/403) 및 폼 검증(400/422) 규칙 확정** | Sprint 1 | Auth, Bulk Import, Audit Core의 필수 요소 | 스키마 확정 |
| **STT 및 비동기 작업 재시도(Timeout) 처리 룰** | Sprint 1 | STT 외부 의존성에 대한 데모 안정성 확보 | - |
| **Sentry/Datadog 인프라 연동** | Sprint 4 | 초기 개발 시에는 서버 콘솔 및 Audit DB로 커버 | 배포 파이프라인 |

---

## 10. 완료 기준 및 작성 가이드

### 10.1. 완료 기준 (Definition of Done)
* **문서 완료 조건**: 프론트엔드/백엔드/QA 파트에서 본 문서를 리뷰하고 코드 매핑 체계에 합의 서명함.
* **설계 완료 조건**: 백엔드 글로벌 Exception Handler 모듈에 본 규격이 인터페이스(`ErrorResponseDto`)로 정의됨.
* **구현 반영 조건**: Sprint 1의 모든 REST API가 본 문서의 JSON 규격 외에 다른 포맷(예: 평문 텍스트, HTML 등)을 반환하지 않음.
* **테스트 반영 조건**: 모든 API 통합 테스트에서 에러 발생 시 `success: false` 및 올바른 `error.code` 반환 여부를 검증함.

### 10.2. 다음 단계 작성 가이드
1. **`TEST-S1_ACCEPTANCE_v1.md`**: 전반적인 Sprint 1 인수 테스트 명세 작성 (HTTP 상태코드 및 에러 코드 Assert 연동)
2. **`API-BULK_IMPORT_v1.md`**: 파일 용량, 파싱, 무결성 오류 응답 명세 작성
3. **`UI-BULK_IMPORT_ADMIN_v1.md`**: 에러 목록 피드백 화면과 재업로드 가이드 UI 작성
4. **`API-AUDIT_REPORT_v1.md`**: 권한 미달, 템플릿 검증 에러 응답 및 권한 오류 체계 연동
5. **`UI-AUDIT_WORKSPACE_v1.md`**: 입력 피드백 및 검증 오류 UI 설계 작성

### 10.3. Opus 4.6 리뷰 포인트
1. 에러 분류 체계가 겹치거나 모호하지 않은가? (400 vs 422 경계 명확성)
2. 인증 실패(401)와 권한 부족(403)의 구분이 명확하며, 403의 보안 로깅이 설계되었는가?
3. `error.message`에 시스템 내부 에러 스택 트레이스 등의 노출 금지 조항이 확고한가?
4. `trace_id`를 통한 프론트-백-로그 간 추적성이 확보되었는가?
5. 시스템 오류(500)와 외부 의존 실패(502/504)가 적절히 구분되어 인프라 오탐을 방지하는가?
6. 클라이언트가 재시도를 논리적으로 판단할 수 있는 기준이 제공되었는가?
7. Sprint 1의 적용 범위가 Sentry 연동 등 과대하지 않게 적절히 통제되었는가?
8. 제공된 JSON 스키마와 Zod 정의가 개발 과정에서 즉각적으로 복사-붙여넣기 수준으로 활용될 수 있는가?
