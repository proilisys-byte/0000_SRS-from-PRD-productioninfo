# T2-001_VISION_AI_SPEC.md

## 1. Vision AI 파이프라인 아키텍처

본 시스템은 현장 카메라 촬영 이미지를 분석하여 제조 현장 불량, 측정값, 작업 상태를 텍스트 및 구조화된 JSON 데이터로 자동 추출합니다.

### A. STT 파이프라인과의 분리 경계 (DEC-003 준수)
- **공유하는 것**: 최종 데이터 병합을 위한 응답 JSON 골격, 처리 이력 `audit_log` 연계 정책, Vercel Serverless 타임아웃 방어 및 Gemini API Rate Limit 지수 백오프 로직.
- **분리하는 것**: 
  - **엔드포인트**: `/api/stt`와 분리된 전용 `/api/vision` 라우트 핸들러 사용.
  - **입력 전처리**: Audio 청크 처리가 아닌 Image 다운사이징 및 Base64 인코딩.
  - **프롬프트**: 음성 명령 문맥이 아닌 '시각적 결함 탐지 및 분류'에 특화된 프롬프트 체인.

### B. 전체 데이터 흐름
1. **카메라 촬영 (Client)**: 브라우저 API를 통한 원본 캡처.
2. **이미지 전처리 (Client)**: 해상도/용량 제한(Canvas API) 및 EXIF 제거.
3. **API 호출 (Client -> Server)**: `/api/vision`으로 Base64 데이터 전송.
4. **스토리지 백업 (Server)**: Supabase Storage에 원본/압축본 저장 후 로그 ID 발급.
5. **AI 분석 (Server -> Gemini)**: `streamObject` 또는 `generateObject`로 Vision API 호출.
6. **구조화 완료 (Server)**: Zod 스키마 기반 JSON 검증.
7. **데이터 영속화 (Server)**: `audit_data_entries` (entry_type='vision')에 Insert.

---

## 2. 이미지 입력 처리 명세

### A. 지원 포맷 및 제약
- **지원 포맷**: `image/jpeg`, `image/png`, `image/webp`. 
  - *HEIC 제외 이유*: 모바일 웹 브라우저 네이티브 렌더링 미지원 및 서버 단 변환 리소스 소모 방지.
- **최대 파일 크기**: 네트워크 비용 및 API 페이로드 최적화를 위해 원본 관계없이 **2MB** 이하로 제한.
- **해상도**: 최대 가로/세로 1920px (Gemini Vision 권장 해상도 기반 다운사이징).

### B. 클라이언트 사이드 이미지 전처리
- **Canvas API 활용 다운사이징 및 EXIF 제거**:
```javascript
function resizeAndCleanImage(file, maxSize = 1920) {
  return new Promise((resolve) => {
    const reader = new FileReader();
    reader.onload = (e) => {
      const img = new Image();
      img.onload = () => {
        const canvas = document.createElement('canvas');
        let { width, height } = img;
        if (width > maxSize || height > maxSize) {
          const ratio = Math.min(maxSize / width, maxSize / height);
          width *= ratio; height *= ratio;
        }
        canvas.width = width; canvas.height = height;
        const ctx = canvas.getContext('2d');
        // 그리기 (이 과정에서 EXIF 위치 정보 자동 탈락)
        ctx.drawImage(img, 0, 0, width, height);
        // WebP 변환
        resolve(canvas.toDataURL('image/webp', 0.8));
      };
      img.src = e.target.result;
    };
    reader.readAsDataURL(file);
  });
}
```

### C. Base64 인코딩 vs Signed URL 비교
- **결정**: **Base64 인코딩 (Vercel API Payload 직접 전송)**
- **이유**: 실시간 처리(Zero-UI)가 핵심이므로, Supabase Storage 업로드 대기 시간 및 Signed URL 생성 오버헤드를 생략하고, 전처리로 1MB 미만 압축된 Base64 텍스트를 Vercel 함수에 즉시 전송하여 Latency를 최소화합니다. 단, 비동기로 백그라운드 Storage 업로드는 병행합니다.

---

## 3. Gemini Vision 프롬프트 설계

### 3-1. System Prompt (불변 컨텍스트)
```text
System: 당신은 반도체 소부장 제조 공정 현장의 정밀 품질 검사를 담당하는 전문 Vision AI입니다.
현장 작업자가 촬영한 이미지를 분석하여 공정 상태, 불량 유형, 그리고 시각적으로 확인 가능한 측정값을 객관적으로 추출하세요.

[규칙]
1. 주어진 JSON 스키마에 정확히 맞추어 응답을 생성해야 합니다. 마크다운이나 부가 설명은 절대 금지합니다.
2. 발견된 모든 객체와 결함을 `detected_items` 배열에 포함하십시오.
3. 이미지 품질이 나쁘거나 결함 판단이 모호한 경우(confidence_score < 0.7), `needs_review`를 반드시 `true`로 설정하고 `overall_result`를 `needs_review`로 지정하세요.
4. 환각(Hallucination) 방지: 사진에 명확히 보이지 않는 결함이나 숫자를 유추하여 적지 마십시오.
```

### 3-2. User Prompt 템플릿
```typescript
const userPrompt = `
[컨텍스트 정보]
- 현재 세션 ID: ${sessionId}
- 타겟 공정 코드: ${processCode}
- 기대 검사 항목(BOM/매뉴얼 기준): ${expectedItems.join(', ')}

이 이미지를 바탕으로 기대 검사 항목의 정상 조립 여부 및 표면 불량(크랙, 스크래치, 이물질 등) 유무를 분석해 주십시오.
`;
```

### 3-3. 출력 JSON 스키마 (Zod)
```typescript
import { z } from 'zod';

export const VisionAnalysisSchema = z.object({
  analysis_type: z.literal('visual_inspection'),
  process_code: z.string(),
  detected_items: z.array(z.object({
    item_type: z.enum(['defect', 'measurement', 'work_status']),
    label: z.string(),
    location: z.string().describe('결함/객체의 화면 상 대략적 위치'),
    severity: z.enum(['critical', 'major', 'minor', 'ok']),
    measured_value: z.string().optional().describe('게이지나 디스플레이에서 읽을 수 있는 텍스트/숫자'),
    confidence_score: z.number().min(0).max(1.0)
  })),
  overall_result: z.enum(['pass', 'fail', 'needs_review']),
  needs_review: z.boolean(),
  raw_description: z.string().describe('이미지에 대한 AI의 전반적인 요약 설명 (내부 로깅용)')
});
```

---

## 4. 카메라 촬영 UI 명세 (T2-002)

### A. 브라우저 API 및 UI 흐름
- **WebRTC API**: `navigator.mediaDevices.getUserMedia({ video: { facingMode: "environment" } })`를 활용하여 디바이스 후면 카메라 호출.
- **오버레이 가이드**:
  - 화면 중앙 십자선(Crosshair) 및 투명 가이드 박스 표출.
  - 조도 센서 또는 ImageCapture API 데이터를 이용해 노출 부족 시 "플래시를 켜거나 밝은 곳으로 이동하세요" 경고 토스트 렌더링.
- **플로우**: `실시간 스트림 뷰` → `[촬영 버튼]` → `정지된 화면 미리보기(Preview)` → `[재촬영] / [분석 전송]`

### B. 하드웨어 스펙 임시 대응 (추상 레이어)
- Edge 단말 장비(스마트 글래스 등) 미확정 상태이므로, 카메라 모듈을 `CameraProvider` 인터페이스로 추상화합니다.
- 현재는 `WebBrowserCameraAdapter`만 구현하고, 추후 하드웨어 확정 시 `NativeDeviceCameraAdapter`로 쉽게 교체할 수 있는 의존성 주입(DI) 패턴 적용.

---

## 5. STT와의 복합 입력 처리

### A. 데이터 병합 전략 (Session 묶음)
- 동일한 `session_id` 하에 오디오 입력과 이미지 입력이 순차적/동시적으로 발생할 수 있습니다.
- **출처 구분**: `audit_data_entries` 테이블의 `entry_type`을 `stt`, `vision`, `manual`로 명확히 구분하여 Insert합니다.
- **우선순위 (Conflict Resolution)**: 
  - STT(작업자 음성)가 "온도 정상"이라고 했으나 Vision AI가 "게이지 120도(초과)"로 읽은 경우, **Vision의 시각적 증거를 우선(우위)**시킵니다.
  - 단, 두 결과가 충돌하면 시스템은 강제로 `needs_review = true` 플래그를 세션에 활성화하여 관리자의 수동 검토를 요구합니다.

---

## 6. Rate Limit 방어 (Vision 특화)

### A. 토큰 소비량 추정 공식
- Gemini 1.5 Pro Multimodal은 이미지를 타일 방식으로 분할하여 토큰을 계산합니다.
- 1080p 기준 전처리(WebP) 이미지 전송 시 장당 약 **258 토큰**이 소비되는 것으로 시스템 내 자체 제한(Budget)을 산정합니다.

### B. STT 공유 제한 및 우선순위
- 분당 15 RPM (무료 티어 기준) 제한을 STT API와 공유합니다.
- **우선순위 큐**: 큐 매니저에서 STT(실시간 작업 기록) 요청을 Priority 1로, Vision(정적 이미지 분석) 요청을 Priority 2로 배정하여 지수 백오프 시 Vision 요청을 더 오래 대기시킵니다.
- **Fallback UI**: HTTP 429 반복 발생 시(3회 이상), 로딩 스피너를 중단하고 "현재 분석 서버가 혼잡합니다. 사진은 안전하게 저장되었으니 검사 결과를 수동으로 입력해 주세요." 메시지와 텍스트 폼 노출.

---

## 7. 성능 및 품질 기준

- **목표 응답 시간**: 이미지 Base64 전송 시작부터 JSON 응답 렌더링까지 **8초 이내 (p95)** 달성. 초과 시 낙관적 UI(Optimistic UI) 렌더링으로 지연 체감 완화.
- **정확도 로깅**: 관리자가 대시보드에서 `needs_review` 판정 결과를 뒤집었을 경우(예: AI는 Fail, 사람은 Pass), 해당 이미지와 프롬프트, 응답 JSON을 `ai_feedback_loops` 테이블에 저장.
- **개선 사이클**: Sprint 주기로 축적된 피드백 오답 노트를 기반으로 System Prompt의 Few-shot 예제를 업데이트(Prompt Engineering 주기적 수행).
