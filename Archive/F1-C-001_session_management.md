# F1-C-001_session_management.md

## 1. 세션 상태 정의 (State Machine)

Smart Audit 세션은 데이터 무결성 보장을 위해 엄격한 상태 전이(State Machine)를 가집니다.

| 상태 (Status) | 진입 조건 | 허용되는 액션 (상태 전이 트리거) | 금지 액션 | RBAC 권한 제약 |
| :--- | :--- | :--- | :--- | :--- |
| **DRAFT** | `createAuditSession` 실행 후 초기 상태 | 세션 시작(`start_session` → `IN_PROGRESS`), 기본 정보(이름, 날짜) 수정, 세션 삭제 | 감사 엔트리(데이터) 추가, 리포트 생성 | Admin, User(본인 할당분) |
| **IN_PROGRESS**| 현장에서 "감사 시작" 버튼 클릭 | 데이터(STT/Vision/수동) 계속 추가, 세션 완료(`complete_session` → `PENDING_REVIEW`) | 세션 기본 정보 수정 금지, 삭제 금지 | Admin, User(본인 할당분) |
| **PENDING_REVIEW** | "감사 완료" 버튼 클릭 후 AI 매핑 중/검토 대기 상태 | AI 매핑 엔진 비동기 실행, 관리자의 데이터 수정/반려(`reject` → `IN_PROGRESS`), 최종 승인(`approve` → `COMPLETED`) | 작업자의 신규 데이터 추가 금지 | Admin 전용 (User는 보기만 가능) |
| **COMPLETED** | 관리자가 최종 승인 완료 및 PDF 리포트 생성 완료 | 보관소 이관(`archive` → `ARCHIVED`) | **모든 데이터 및 상태 수정 절대 금지** | Admin 전용 |
| **ARCHIVED** | 보관/폐기 주기에 따라 논리적 보관 처리 시 | 읽기 전용 조회만 가능 (전이 불가) | 복구 불가, 수정 불가, 삭제(Soft/Hard 모두) 불가 | Admin 전용 |

---

## 2. 세션 생성 Command 명세

Next.js Server Action을 활용하여 새로운 세션을 DB에 원자적으로(Atomic) 등록합니다.

### A. Zod 검증 스키마
```typescript
import { z } from 'zod';

export const CreateAuditSessionSchema = z.object({
  name: z.string().min(5).max(100),
  auditType: z.enum(['ISO_9001', 'CUSTOMER_AUDIT', 'INTERNAL_AUDIT']),
  scheduledDate: z.string().datetime(),
  assigneeId: z.string().uuid(),
  templateId: z.string().uuid().optional(),
});
```

### B. DB 트랜잭션 및 로직
```typescript
'use server';

import { redirect } from 'next/navigation';
import { prisma } from '@/lib/prisma';
import { CreateAuditSessionSchema } from '@/lib/validations';
import { getUserAuth } from '@/lib/auth';

export async function createAuditSession(formData: FormData) {
  const user = await getUserAuth();
  if (!user || user.role !== 'admin') throw new Error('Unauthorized');

  const parsed = CreateAuditSessionSchema.parse(Object.fromEntries(formData));

  const session = await prisma.$transaction(async (tx) => {
    // 1. 세션 생성
    const newSession = await tx.audit_sessions.create({
      data: {
        name: parsed.name,
        audit_type: parsed.auditType,
        scheduled_date: new Date(parsed.scheduledDate),
        assignee_id: parsed.assigneeId,
        template_id: parsed.templateId,
        status: 'DRAFT',
        created_by: user.id
      }
    });

    // 2. Audit Log 원자적 기록
    await tx.audit_log.create({
      data: {
        action_type: 'SESSION_CREATED',
        user_id: user.id,
        target_table: 'audit_sessions',
        target_id: newSession.id,
        status: 'PASS'
      }
    });

    return newSession;
  });

  // 생성 완료 후 DRAFT 세션의 상세(준비) 화면으로 리다이렉트
  redirect(`/audit/workspace/${session.id}`);
}
```

---

## 3. 세션 데이터 수집 Command 명세

현장 작업자가 STT나 수동 조작으로 데이터를 기입할 때 호출되는 엔드포인트입니다.

### `POST /api/sessions/{session_id}/entries`
- **Request Body**:
```json
{
  "entry_type": "stt", // 'stt' | 'vision' | 'manual'
  "raw_input": "1번 라인 온도 120도로 확인됨",
  "parsed_json": { "equipment": "line_1", "temperature": 120 },
  "confidence_score": 0.95
}
```
- **상태 방어 로직**:
  - `session_id`로 DB를 조회하여 `status !== 'IN_PROGRESS'`일 경우 HTTP 422 반환.
- **Race Condition 방지**:
  - 오프라인 환경 등에서 한꺼번에 요청이 들어올 때 데이터 순서 꼬임을 막기 위해, 클라이언트에서 UUID 형태의 `idempotency_key`(또는 `client_timestamp`)를 부여하여 서버로 전송.
  - DB Insert 전 `idempotency_key` 중복 확인으로 멱등성 보장.

---

## 4. 세션 완료 및 리포트 생성 트리거 Command

작업자가 현장 감사를 마치고 "완료" 버튼을 눌렀을 때의 흐름입니다.

1. **완료 조건 검사**: 
   - `entry_type === 'stt'` 등의 데이터가 최소 1건 이상 존재하는지 확인.
2. **상태 전환**: 
   - `IN_PROGRESS` → `PENDING_REVIEW`로 DB 상태 업데이트. (이 순간부터 클라이언트의 엔트리 추가 API는 HTTP 422 거부 시작)
3. **매핑 엔진 비동기 호출**:
   - `waitUntil()` (또는 Upstash 큐)를 통해 AI 매핑 엔진 워커 트리거.
   - 워커는 해당 세션의 모든 `entries`를 수집하여 하나의 통합 Report JSON 생성.
4. **결과 연계**:
   - AI 매핑이 성공하면 상태가 `PENDING_REVIEW` 유지 및 관리자에게 "리뷰 요청" 알림.
   - 관리자가 최종 검토 후 `COMPLETED`로 변경하면 즉시 PDF 생성 서버리스 함수가 트리거됨.

---

## 5. 세션 목록 조회 Query 명세 (UI-011 연동)

무한 스크롤(Infinite Scroll) 환경에서 대용량 세션을 지연 없이 조회하기 위한 Cursor Pagination 기반 쿼리 명세입니다.

### A. 필터 파라미터 스키마 (Zod)
```typescript
export const SessionQuerySchema = z.object({
  status: z.enum(['DRAFT', 'IN_PROGRESS', 'PENDING_REVIEW', 'COMPLETED', 'ARCHIVED', 'ALL']).default('ALL'),
  auditType: z.string().optional(),
  assigneeId: z.string().uuid().optional(),
  cursor: z.string().optional(), // Prisma의 경우 대상 레코드의 ID (UUID)
  limit: z.number().min(5).max(100).default(20),
});
```

### B. Prisma 쿼리 예시
```typescript
const { status, assigneeId, cursor, limit } = SessionQuerySchema.parse(reqQuery);

const queryArgs: any = {
  take: limit + 1, // 다음 페이지 유무 확인을 위해 하나 더 가져옴
  orderBy: { updated_at: 'desc' },
  where: {
    ...(status !== 'ALL' && { status }),
    ...(assigneeId && { assignee_id: assigneeId }),
  }
};

if (cursor) {
  queryArgs.cursor = { id: cursor };
  queryArgs.skip = 1; // 기준 cursor 레코드는 제외하고 다음부터 반환
}

const sessions = await prisma.audit_sessions.findMany(queryArgs);

let nextCursor: string | undefined = undefined;
if (sessions.length > limit) {
  const nextItem = sessions.pop(); // 초과 조회한 1건 제거
  nextCursor = nextItem!.id;
}

return {
  items: sessions,
  nextCursor,
  hasMore: nextCursor !== undefined
};
```

---

## 6. 세션 불변성 보장 (무결성)

컴플라이언스 준수를 위해 완료된 세션 데이터의 조작을 원천 봉쇄합니다.

### A. RLS(Row Level Security) 적용 SQL
Supabase 단에서 쿼리 실행을 거부하는 방어벽을 구축합니다.
```sql
-- audit_data_entries (세션 데이터) 테이블 수정/삭제 차단
CREATE POLICY "Block update on entries if session completed" ON audit_data_entries
FOR UPDATE
USING (
  (SELECT status FROM audit_sessions WHERE id = audit_data_entries.session_id) NOT IN ('COMPLETED', 'ARCHIVED')
);

CREATE POLICY "Block delete strictly" ON audit_data_entries
FOR DELETE
USING (false); -- 데이터 삭제는 상태 무관 영구 금지 (Insert-only)
```

### B. 앱 레벨 선검사
Server Action이나 Route Handler의 최상단에서 DB RLS 도달 전에 미리 쳐내는 로직.
```typescript
const session = await prisma.audit_sessions.findUnique({ select: { status: true }});
if (['COMPLETED', 'ARCHIVED'].includes(session.status)) {
  throw new AppError('INVALID_STATE_TRANSITION', '완료된 세션은 수정할 수 없습니다.', 422);
}
```

### C. 삭제 금지 정책
- 프로젝트 내에 `DELETE FROM audit_sessions` 코드는 존재해서는 안 됩니다.
- 오기입으로 인한 취소 처리는 `status = 'ARCHIVED'` 상태 업데이트로만 수행(Soft Delete의 변형)하여 DB에 영구 보존합니다.
