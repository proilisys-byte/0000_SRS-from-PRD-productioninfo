# API-AI_PIPELINE_v1.md

## 1. STT 파이프라인 명세 (T1-006)

### A. Gemini 1.5 Pro Multimodal Audio 입력 방식
* **지원 오디오 포맷**: WAV, MP3, M4A, FLAC
* **제약사항**: Vercel 및 브라우저 제한을 고려하여 오디오 파일 크기는 최대 20MB(약 10~15분)로 제한하며, 샘플링 레이트는 16kHz 이상을 권장합니다.
* **Vercel AI SDK `streamText` 코드 예시 (TypeScript)**:
```typescript
import { streamText } from 'ai';
import { google } from '@ai-sdk/google';

// app/api/stt/route.ts
export const runtime = 'edge';

export async function POST(req: Request) {
  const formData = await req.formData();
  const audioFile = formData.get('audio') as File;
  const audioBuffer = await audioFile.arrayBuffer();

  const result = await streamText({
    model: google('gemini-1.5-pro-latest'),
    messages: [
      {
        role: 'user',
        content: [
          { type: 'text', text: '다음 제조 현장 음성에서 공정명, 작업 수량, 불량 코드를 추출하여 JSON 형태로 반환해.' },
          { type: 'file', data: audioBuffer, mimeType: audioFile.type },
        ],
      },
    ],
  });
  
  return result.toDataStreamResponse();
}
```

### B. STT 출력 → 구조화 JSON 변환 프롬프트
* **프롬프트 전략**: 현장 작업자의 방언이나 부정확한 발음을 보정하기 위해 도메인 특화 지시어를 포함합니다.
* **출력 JSON 스키마 (Zod 정의)**:
```typescript
import { z } from 'zod';

export const SttOutputSchema = z.object({
  process_name: z.string().describe("공정명 (예: 압출, 조립, 도색)"),
  quantity: z.number().int().describe("작업 수량 또는 불량 수량"),
  defect_code: z.string().optional().describe("불량 발생 시 코드명 (예: D-001)"),
  notes: z.string().optional().describe("기타 현장 작업자 특이사항 메모")
});
```
* **Fallback 처리 로직**: Zod 검증 파싱 실패 또는 주요 식별자(process_name, quantity) 누락 시 클라이언트에서 에러 처리 후 "음성 인식이 불완전합니다. 수동으로 값을 입력해주세요" 배너와 함께 수동 입력 폼으로 전환합니다.

### C. STT 정확도 92% 목표 달성 검증 방법
* **Fallback율 측정**: `audit_data_entries` 테이블에 `is_fallback` (boolean) 플래그를 추가하여, 자동 저장에 실패하고 수동 수정한 비율을 모니터링합니다.
* **개선 사이클**: Fallback이 발생한 원본 음성과 수동 정정 텍스트를 수집하여 Gemini 프롬프트의 Few-shot 예시(Prompt Tuning)로 주 1회 반영합니다.

---

## 2. Smart Audit 매핑 엔진 명세 (T1-008)

### A. ISO 9001 템플릿 매핑 프롬프트
* **입력**: `audit_data_entries`에서 수집된 현장 데이터 JSON Array
* **출력 구조**: ISO 9001의 각 심사 섹션별 요약 텍스트 및 신뢰도 점수(0~100)를 포함하는 JSON.
* **신뢰도 산출 방법**: LLM 스스로 추론 과정(Chain of Thought)을 거쳐 데이터의 누락 여부를 판단하고 `confidence_score` 필드를 반환합니다. 80점 미만 시 UI에서 노란색 경고로 관리자 수동 검토를 유도합니다.

### B. Vercel 60초 타임아웃 방어 전략 (필수)
* **문제점**: Vercel Serverless 함수의 기본 타임아웃(최대 60초) 내에 대규모 매핑이 종료되지 않으면 에러가 발생합니다.
* **방어 전략**: `export const runtime = 'edge';`를 적용하고 HTTP Streaming 기반으로 응답을 쪼개어 타임아웃을 우회합니다.
* **`StreamingTextResponse` 패턴**:
```typescript
import { streamText } from 'ai';
import { google } from '@ai-sdk/google';

export const runtime = 'edge';

export async function POST(req: Request) {
  const { rawData } = await req.json();
  const result = await streamText({
    model: google('gemini-1.5-pro-latest'),
    prompt: `다음 데이터를 ISO 9001 양식에 맞게 매핑하라: ${JSON.stringify(rawData)}`,
  });
  return result.toDataStreamResponse();
}
```
* 클라이언트는 `useCompletion` 훅을 사용해 수신되는 텍스트 청크 단위로 진행률 UI를 표시하며, 타임아웃/연결 끊김 발생 시 멈춘 지점부터 Retry를 수행합니다.

### C. html2pdf 한글 폰트 처리
* 폰트 렌더링 지연으로 인한 글자 깨짐 방지를 위해 `Noto Sans KR` 폰트를 Base64 포맷으로 CSS 내부에 직접 임베딩(`@font-face`)합니다.
* PDF 생성 트리거 시 `document.fonts.ready` Promise가 완료될 때까지 대기 후 `html2pdf()`를 실행합니다.

---

## 3. Gemini Rate Limit 방어 전략

### A. Free Tier 한계 수치 명시
* **RPM** (요청 수/분): 15
* **RPD** (요청 수/일): 1,500
* **TPM** (토큰 수/분): 1,000,000

### B. 지수 백오프(Exponential Backoff) 구현
* HTTP 429(Too Many Requests) 발생 시 대기 시간을 2배씩 늘려 재시도합니다.
```typescript
export async function fetchWithBackoff(fn: () => Promise<any>, maxRetries = 3) {
  let attempt = 0;
  let delay = 2000; // 초기 대기시간 2초
  
  while (attempt < maxRetries) {
    try {
      return await fn();
    } catch (error: any) {
      // AI SDK 에러 처리 로직
      if (error?.response?.status === 429 || error?.status === 429) {
        attempt++;
        console.warn(`429 Rate Limit. Retrying in ${delay}ms... (Attempt ${attempt})`);
        await new Promise(resolve => setTimeout(resolve, delay));
        delay *= 2; // 배수 설정값 (x2)
      } else {
        throw error;
      }
    }
  }
  throw new Error("Max retries exceeded for Gemini API");
}
```

### C. 요청 큐잉 전략
* 동시다발적인 요청은 Vercel 환경에서 즉각 처리가 불가할 수 있으므로, 단기적으로는 클라이언트 측 분산 요청을 유도하고, 장기적으로(또는 오류 빈발 시) **Upstash QStash**를 도입하여 백그라운드 큐 기반 비동기 웹훅 처리 아키텍처로 전환합니다.
