---
name: Feature Task
about: SRS 기반의 구체적인 개발 태스크 명세
title: "[Feature] API-007: 공통 에러 코드 체계 정의 (400/401/403/404/409/429/500)"
labels: 'feature, backend, api, priority:high, sprint:0'
assignees: ''
---

## 🎯 Summary
- 기능명: [API-007] 공통 에러 코드 체계 정의 (400/401/403/404/409/429/500)
- 목적: 시스템 전체 API에서 사용할 통일된 에러 응답 구조와 에러 코드 체계를 정의한다. 모든 API 엔드포인트가 일관된 에러 포맷을 반환하도록 하여 프론트엔드 개발자와 외부 연동 시 예측 가능한 에러 처리가 가능하게 한다. 이 태스크는 선행 의존성이 없으며, 전체 API 스펙(API-001~006) 및 Input Validation(API-008)의 기반이 된다.

## 🔗 References (Spec & Context)
> 💡 AI Agent & Dev Note: 작업 시작 전 아래 문서를 반드시 먼저 Read/Evaluate 할 것.
- SRS 문서: [`05_SRS_v1.md#§4.2.4.4`] — API 보안 (Rate Limiting, Input Validation)
- SRS 문서: [`05_SRS_v1.md#§3.3`] — API & Data Interaction Overview
- SRS 문서: [`05_SRS_v1.md#REQ-NF-SEC-API-001`] — Rate Limiting 초과 시 429 반환 100%
- SRS 문서: [`05_SRS_v1.md#REQ-NF-SEC-API-004`] — 스키마 위반 요청 차단 100%
- SRS 문서: [`05_SRS_v1.md#§1.2.3`] — 기술 제약사항 (C-TEC-002: Server Actions / Route Handlers)
- 태스크 목록: [`TASKS/06_TASK_LIST_v1.md#Epic2`] — API Spec 태스크 목록
- 프로젝트 컨텍스트: [`PROJECT_CONTEXT.md`] — 디렉토리 구조, 환경 변수, 코딩 컨벤션

## 📁 Implementation Paths
- `src/types/common.ts` — ErrorResponse 인터페이스 + 에러 코드 enum 정의
- `src/lib/errors/error-factory.ts` — 에러 생성 유틸리티 함수
- `src/lib/errors/error-handler.ts` — 에러 핸들링 미들웨어
- `src/lib/errors/error-logger.ts` — 에러 로깅 (민감정보 마스킹)
- `docs/error-catalog.md` — 전체 에러 코드 카탈로그 문서

## ✅ Task Breakdown (실행 계획)
- [ ] 공통 에러 응답 인터페이스(ErrorResponse) TypeScript 타입 정의
  ```typescript
  interface ErrorResponse {
    success: false;
    error: {
      code: string;        // 예: "AUTH_001", "VALIDATION_002"
      httpStatus: number;   // 예: 400, 401, 403 ...
      message: string;      // 사용자 표시용 메시지
      detail?: string;      // 개발자용 상세 메시지
      timestamp: string;    // ISO 8601
      path?: string;        // 요청 경로
      traceId?: string;     // 분산 추적용 ID
    };
  }
  ```
- [ ] HTTP 상태 코드별 에러 코드 enum 체계 정의
  - `400` Bad Request: VALIDATION_*, INVALID_INPUT, MISSING_FIELD
  - `401` Unauthorized: AUTH_EXPIRED, AUTH_INVALID, AUTH_REQUIRED
  - `403` Forbidden: AUTHZ_DENIED, RBAC_INSUFFICIENT, RESOURCE_LOCKED
  - `404` Not Found: RESOURCE_NOT_FOUND, ENTITY_NOT_FOUND
  - `409` Conflict: DUPLICATE_ENTRY, VERSION_CONFLICT, STATE_CONFLICT
  - `429` Too Many Requests: RATE_LIMIT_EXCEEDED
  - `500` Internal Server Error: INTERNAL_ERROR, SERVICE_UNAVAILABLE, AI_ENGINE_ERROR
  - `503` Service Unavailable: EXTERNAL_SERVICE_DOWN, CIRCUIT_BREAKER_OPEN
- [ ] 도메인별 에러 코드 네임스페이스 체계 설계 (AUTH_*, AUDIT_*, NC_*, LEAN_*, EDGE_*, AI_*)
- [ ] 공통 에러 생성 유틸리티 함수 구현 (`createErrorResponse`, `createValidationError`)
- [ ] Next.js Server Action / Route Handler에서의 에러 핸들링 미들웨어 구현
- [ ] 에러 로깅 통합 (민감정보 마스킹 포함, REQ-NF-SEC-LOG-003 선반영)
- [ ] 에러 코드 레지스트리 문서 작성 (전체 에러 코드 카탈로그)
- [ ] 단위 테스트: 각 HTTP 상태 코드별 에러 응답 포맷 검증

## 🧪 Acceptance Criteria (BDD/GWT)

**Scenario 1: Validation 에러 표준 응답**
- Given: 필수 필드(site_id)가 누락된 API 요청이 수신됨
- When: Server Action에서 Zod 검증이 실패함
- Then: 400 상태 코드와 함께 `{ success: false, error: { code: "VALIDATION_001", httpStatus: 400, message: "필수 항목이 누락되었습니다.", detail: "site_id is required" } }` 형태의 표준 에러 응답을 반환한다.

**Scenario 2: 인증 만료 에러**
- Given: JWT 토큰이 만료된 상태에서 API 요청이 수신됨
- When: 인증 미들웨어에서 토큰 검증이 실패함
- Then: 401 상태 코드와 `code: "AUTH_EXPIRED"` 에러를 반환하며, 응답 본문에 토큰이나 세션 정보가 포함되지 않는다.

**Scenario 3: RBAC 권한 부족**
- Given: User 역할의 사용자가 Admin 전용 API를 호출함
- When: RBAC 검증에서 권한 부족이 감지됨
- Then: 403 상태 코드와 `code: "AUTHZ_DENIED"` 에러를 반환한다.

**Scenario 4: Rate Limit 초과**
- Given: 특정 API Key가 분당 허용 횟수를 초과함
- When: Rate Limiter에서 초과가 감지됨
- Then: 429 상태 코드와 `code: "RATE_LIMIT_EXCEEDED"` 에러를 반환하며, `Retry-After` 헤더가 포함된다.

**Scenario 5: 에러 로깅 시 민감정보 마스킹**
- Given: 요청 body에 비밀번호 필드가 포함된 API 호출에서 에러가 발생함
- When: 에러가 로깅됨
- Then: 로그에 비밀번호 필드는 `"***"` 로 마스킹되어 기록되며, 평문 노출이 0건이다.

## ⚙️ Technical & Non-Functional Constraints
- 프레임워크: Next.js App Router Server Actions / Route Handlers (C-TEC-001, C-TEC-002)
- 검증: Zod 스키마 기반 Input Validation (REQ-NF-SEC-API-004)
- Rate Limiting: 엔드포인트·API Key별 제한, 429 반환 100% (REQ-NF-SEC-API-001)
- 로깅: 개인식별정보·인증정보 자동 마스킹 (REQ-NF-SEC-LOG-003)
- 국제화: 에러 메시지는 한국어 기본, 향후 다국어 확장 가능 구조
- 타입 안전: TypeScript strict 모드에서 타입 안전한 에러 코드 관리

## 🏁 Definition of Done (DoD)
- [ ] 모든 Acceptance Criteria를 충족하는가?
- [ ] 전체 에러 코드 카탈로그 문서가 작성되었는가?
- [ ] ErrorResponse 타입이 TypeScript로 정의되고, export 가능한가?
- [ ] 에러 생성 유틸리티 함수가 구현되고 단위 테스트가 통과하는가?
- [ ] 에러 로깅 시 민감정보 마스킹이 동작하는가?
- [ ] ESLint / 정적 분석 경고가 없는가?
- [ ] API 명세서(OpenAPI)에 에러 스키마가 반영 가능한 구조인가?

## 🚧 Dependencies & Blockers
- **Depends on:** None (선행 의존성 없음)
- **Blocks:** API-008 (OpenAPI Input Validation 명세), INT-C-005 (멱등성 키 미들웨어), INFRA-005 (Rate Limiting 구현), TEST-AUTH-004 (Rate Limiting 429 반환 테스트), TEST-AUTH-005 (Input Validation 차단 테스트), 전체 API-001~006 DTO 정의 시 에러 응답 포맷 참조
