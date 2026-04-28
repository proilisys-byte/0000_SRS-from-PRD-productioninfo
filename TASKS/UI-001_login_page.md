---
name: Feature Task
about: PRD&SRS 기반의 구체적인 개발 태스크 명세
title: "[UI] UI-001: Login Page"
labels: 'ui, spec, priority:high'
assignees: ''
---

## 🎯 Summary
- 기능명: [UI-001] Login Page
- 목적: 사용자 인증을 수행하는 첫 진입점 화면을 구축하여 사용자가 플랫폼에 접근하고, 로그인 후 적절한 권한의 대시보드로 라우팅되도록 한다.

## 🔗 References (Spec & Context)
> 작업 시작 전 아래 문서를 반드시 먼저 읽고 정합성을 유지할 것.
- PRD/SRS 문서: `00_PRD_v1.md`
- 공통 인증 체계: `COM-AUTH_v1.md`
- 관련 API: `COM-RH-002_auth_route_handler.md`

## 🧭 Scope
### In Scope
- 이메일/비밀번호 입력 폼 UI
- 로그인 성공 시 홈(또는 대시보드) 리다이렉트
- 로그인 실패 시 에러 메시지(Validation, 401 등) 표시
- 자동 로그인(Remember Me) 토글
- **비밀번호 정책 클라이언트 Validation** (SRS REQ-NF-SEC-AUTH-003)
- **연속 실패 시 계정 잠금 안내 UI** (SRS REQ-NF-SEC-AUTH-005)
- **유휴 30분 자동 로그아웃 처리** (SRS REQ-NF-SEC-AUTH-004)

### Out of Scope
- 비밀번호 찾기 기능 (추가 기획 대기)
- 다중 인증(MFA) 흐름 (Sprint 4 T4-003, `01_TASK_LIST_v2.md` 참조)

## 🧱 Preconditions
- 선행 완료 문서:
  - `COM-AUTH_v1.md`
- 선행 의존성:
  - `COM-RH-002_auth_route_handler.md` (Mock 또는 실제 API)
- 시작 가능 조건:
  - 로그인 폼 필드 구성과 API 응답 규격이 정의됨

## ✅ Task Breakdown (실행 계획)
- [ ] 레이아웃 및 폼 컴포넌트 마크업
- [ ] 클라이언트 사이드 Form Validation (빈 값, 이메일 형식, **비밀번호 정책**)
- [ ] 로그인 API(`POST /api/auth/login`) 연동 로직
- [ ] API 에러 응답 수신 시 UI 알림(Toast 또는 폼 에러 메시지) 처리
- [ ] 인증 성공 시 JWT/세션 기반 Auth Context 업데이트 및 `router.push` 처리
- [ ] **계정 잠금(429) 응답 시 남은 잠금 시간 표시 UI**
- [ ] **유휴 30분 감지 → 세션 만료 모달 → 로그인 리다이렉트 로직**

### 비밀번호 정책 Zod 스키마 (REQ-NF-SEC-AUTH-003)
```typescript
import { z } from 'zod';

export const PasswordSchema = z.string()
  .min(12, '비밀번호는 12자 이상이어야 합니다')
  .refine((pw) => {
    const checks = [
      /[A-Z]/.test(pw),   // 대문자
      /[a-z]/.test(pw),   // 소문자
      /[0-9]/.test(pw),   // 숫자
      /[^A-Za-z0-9]/.test(pw), // 특수문자
    ];
    return checks.filter(Boolean).length >= 3;
  }, '대·소문자, 숫자, 특수문자 중 3종 이상을 포함해야 합니다');

export const LoginFormSchema = z.object({
  email: z.string().email('올바른 이메일 형식을 입력하세요'),
  password: PasswordSchema,
});
```

## 🧪 Acceptance Criteria (BDD / GWT)
Scenario 1: 로그인 성공 및 이동
- Given: 이메일과 비밀번호를 정상 입력했다.
- When: 로그인 버튼을 클릭한다.
- Then: 인증 API 호출 성공 후 사용자의 Role에 맞는 대시보드(예: `UI-010` 또는 `UI-061`)로 리다이렉트된다.

Scenario 2: 로그인 실패 안내
- Given: 존재하지 않는 이메일이나 틀린 비밀번호를 입력했다.
- When: 로그인 버튼을 클릭한다.
- Then: 폼 제출이 완료된 후, "이메일 또는 비밀번호가 일치하지 않습니다"라는 에러 메시지가 화면에 노출된다.

Scenario 3: 클라이언트 밸리데이션
- Given: 비밀번호를 비워둔 채로
- When: 로그인 폼을 제출하려 한다.
- Then: API 요청이 전송되지 않고 입력 칸에 "비밀번호를 입력하세요" 오류가 표시된다.

Scenario 4: 비밀번호 정책 검증 (REQ-NF-SEC-AUTH-003)
- Given: 10자 이하이거나 대·소문자·숫자·특수문자 중 2종 이하로만 구성된 비밀번호를 입력했다.
- When: 로그인 폼을 제출하려 한다.
- Then: API 요청이 전송되지 않고 "비밀번호는 12자 이상, 3종 이상 문자 조합이 필요합니다" 오류가 표시된다.

Scenario 5: 계정 잠금 (REQ-NF-SEC-AUTH-005)
- Given: 동일 계정으로 5회 연속 잘못된 비밀번호를 입력하여 로그인에 실패했다.
- When: 6번째 로그인 시도를 한다.
- Then: API가 HTTP 429를 반환하고, 화면에 "계정이 잠겼습니다. 15분 후 다시 시도해주세요."라는 안내가 표시된다. 잠금 시간 카운트다운이 실시간으로 갱신된다.

Scenario 6: 유휴 자동 로그아웃 (REQ-NF-SEC-AUTH-004)
- Given: 로그인 성공 후 30분 이상 마우스·키보드 입력 없이 유휴 상태가 유지되었다.
- When: 30분 경과 후 아무 동작(페이지 이동, 클릭 등)을 시도한다.
- Then: 세션 만료 안내 모달이 표시되고 "확인" 클릭 시 로그인 페이지로 리다이렉트된다.

## 🔐 Technical / Domain Constraints
- 상태 전이 규칙:
  - 로그인 성공 -> App Shell 마운트 -> Dashboard 라우팅
- 공통 에러 스키마 연계:
  - `API-001`에서 정의된 `error_code`를 기반으로 다국어(또는 사용자 친화적) 에러 메시지로 매핑
  - 계정 잠금 시 `ACCOUNT_LOCKED` 에러 코드 + `retry_after_seconds` 필드 활용
- 보안 / 민감정보 처리:
  - 평문 비밀번호가 브라우저 콘솔 로그나 로컬 스토리지에 기록되지 않도록 주의
  - 비밀번호 입력 필드는 `autocomplete="current-password"` 설정
- 유휴 타임아웃:
  - `mousemove`, `keydown`, `click` 이벤트를 디바운스(30초 간격)로 감지
  - 마지막 활동 시점 + 30분 경과 시 세션 무효화 API 호출 후 리다이렉트

### SRS 트레이서빌리티

| SRS REQ ID | 요구사항 | 반영 위치 |
|:---|:---|:---|
| REQ-NF-SEC-AUTH-003 | 비밀번호 12자 이상, 3종 이상 문자 조합 | Scenario 4 + Zod 스키마 |
| REQ-NF-SEC-AUTH-004 | 유휴 30분 자동 로그아웃 | Scenario 6 + Task Breakdown |
| REQ-NF-SEC-AUTH-005 | 5회 연속 실패 시 15분 잠금 | Scenario 5 + Task Breakdown |

## 📦 Deliverables
- 산출 문서/코드/테스트/Mock/연계 결과:
  - `LoginPage.tsx` 및 폼 관련 Hook
  - 통합 UI 테스트 (Cypress/Playwright 등)

## 🏁 Definition of Done (DoD)
- [ ] PRD/SRS 및 관련 기존 TASK 문서와 충돌하지 않는가?
- [ ] Acceptance Criteria를 모두 반영했는가?
- [ ] 상태/권한/tenant-site/에러/감사로그 기준이 누락되지 않았는가?
- [ ] 관련 후속 문서 또는 구현 단계로 바로 넘길 수 있는가?
- [ ] 범위가 다른 카테고리 문서와 섞이지 않았는가?

## 🚧 Dependencies & Blockers
- Depends on:
  - `COM-RH-002_auth_route_handler.md`
- Blocks:
  - 사용자 진입을 요구하는 모든 클라이언트 페이지 테스트

## 📝 Notes for Dev / AI Agent
- 이 TASK의 핵심 초점:
  - 사용자 경험의 첫인상이므로, 에러 발생 시 부드럽고 명확한 피드백을 제공해야 한다.
- 관련 문서와의 경계:
  - API 통신부의 복잡한 로직 처리는 RH 명세에 따르며, UI는 응답에 기반한 '표시'에 집중한다.
