---
name: Feature Task
about: SRS 기반의 구체적인 개발 태스크 명세
title: "[Feature] ZUI-C-001: 음성 STT 변환 — Gemini Multimodal API 연동 (Vercel AI SDK)"
labels: 'feature, backend, ai, edge, priority:critical, sprint:1'
assignees: ''
requires_human_review: true
---

## 🎯 Summary
- 기능명: [ZUI-C-001] 음성 STT 변환 — Gemini Multimodal API 연동 (Vercel AI SDK)
- 목적: 제조 현장(80dB+ 소음 환경)에서 현장 반장(오반장)이 발화하는 음성을 실시간으로 텍스트로 변환(STT)하여 구조화된 불량 접수 데이터로 자동 입력한다. Gemini 1.5 Multimodal API를 Vercel AI SDK를 통해 연동하며, 노이즈 캔슬링 전처리 후 한국어 음성 인식을 수행한다. Zero-UI 수집기의 핵심 모듈로서, 키보드/터치 없이 음성만으로 데이터 입력이 완료되는 UX를 실현한다.

## 🔗 References (Spec & Context)
> 💡 AI Agent & Dev Note: 작업 시작 전 아래 문서를 반드시 먼저 Read/Evaluate 할 것.
- SRS 문서: [`05_SRS_v1.md#REQ-FUNC-011`] — 음성 기반 불량 접수 입력 (Online)
- SRS 문서: [`05_SRS_v1.md#REQ-FUNC-015`] — 인식 실패 시 수동 입력 Fallback
- SRS 문서: [`05_SRS_v1.md#REQ-NF-002`] — p95 latency ≤ 3s (음성 인식)
- SRS 문서: [`05_SRS_v1.md#§4.3.3`] — Zero-UI 수집기 시퀀스 다이어그램
- SRS 문서: [`05_SRS_v1.md#§3.4.1`] — 컴포넌트 다이어그램 (Edge→Cloud 연동)
- SRS 문서: [`05_SRS_v1.md#§1.2.3`] — 기술 제약 (C-TEC-005: Vercel AI SDK, C-TEC-006: Gemini Multimodal)
- SRS 문서: [`05_SRS_v1.md#REQ-NF-PRIV-003`] — Edge 가명처리 (피치 비가역 변환)
- SRS 문서: [`05_SRS_v1.md#REQ-NF-PRIV-007`] — STT 벤더 DPA 계약 및 Zero-Retention
- 측정 프로토콜: [`05_SRS_v1.md#Appendix A.2`] — STT 정확도 측정 (WER ≤ 8%, CSR ≥ 92%)
- 태스크 목록: [`TASKS/06_TASK_LIST_v1.md#Epic7`] — Zero-UI 수집기 태스크 목록
- 프로젝트 컨텍스트: [`PROJECT_CONTEXT.md`] — 디렉토리 구조, 환경 변수, 코딩 컨벤션

> 🔍 **requires_human_review: true** — H 복잡도. AI Multimodal 연동 설계 리뷰 필수.

## 📁 Implementation Paths
- `src/app/actions/edge/transcribeVoice.ts` — STT Server Action
- `src/lib/ai/stt-engine.ts` — Gemini Multimodal STT 엔진
- `src/lib/validations/edge.ts` — 오디오 입력 검증 스키마
- `src/types/dto/edge.ts` — Edge/STT DTO 타입 정의
- `src/types/stt.ts` — STTResult 인터페이스 정의

## ✅ Task Breakdown (실행 계획)
- [ ] Gemini Multimodal API 연동 설정
  - `@ai-sdk/google` 패키지 설치 및 API Key 환경 변수 설정
  - Gemini 1.5 Flash Multimodal 모델 선택 (음성+텍스트 동시 처리)
  - Vercel AI SDK `generateText()` 기반 STT 처리 파이프라인 구현
- [ ] 음성 데이터 전처리 파이프라인
  - 오디오 포맷 검증 (지원 포맷: WAV, WebM, OGG, MP3)
  - 오디오 길이 검증 (최소 0.5초 ~ 최대 60초)
  - 파일 크기 제한 검증 (≤ 4.5MB, Vercel 서버리스 페이로드 제약)
  - 대용량 파일(> 4.5MB) 감지 시 Supabase Storage Direct Upload 유도 (ZUI-C-004 연동)
- [ ] STT 변환 Server Action 구현 (`transcribeVoice.ts`)
  - Gemini Multimodal 프롬프트: "다음 한국어 공장 현장 음성을 텍스트로 변환하세요. 불량 접수 데이터 형식으로 구조화하세요."
  - 출력 구조 Zod 검증:
    ```typescript
    interface STTResult {
      raw_text: string;          // 원본 전사 텍스트
      structured_data: {
        defect_type?: string;    // 불량 유형
        production_line?: string; // 라인 번호
        part_number?: string;     // 부품 번호
        severity?: string;        // 심각도
        description: string;      // 상세 설명
      };
      confidence: number;        // 0.0 ~ 1.0 신뢰도
      processing_time_ms: number;
    }
    ```
- [ ] 인식 신뢰도 기반 분기 처리
  - confidence ≥ 0.92: 정상 결과 반환 + 확인 프롬프트
  - 0.70 ≤ confidence < 0.92: 경고 표시 + 결과 반환 + 수정 유도
  - confidence < 0.70: 인식 실패 → 재시도 카운터 증가
  - 3회 연속 실패: 수동 입력 Fallback UI 전환 이벤트 발행 (ZUI-C-006 연동)
- [ ] 연속 실패 카운터 관리 (세션 기반, 성공 시 리셋)
- [ ] 응답 시간 측정 및 메트릭 기록 (p95 ≤ 3s 검증용)
- [ ] 에러 처리
  - Gemini API 5xx/타임아웃: 지수 백오프 재시도 (최대 3회)
  - 네트워크 단절: 즉각 오류 알림 + 재시도 유도 (ZUI-C-005 연동)
  - 지원하지 않는 오디오 포맷: 400 에러 + 포맷 안내
- [ ] 단위 테스트 + STT 정확도 검증 테스트 (WER/CSR 측정)

## 🧪 Acceptance Criteria (BDD/GWT)

**Scenario 1: 정상 음성 인식 및 구조화**
- Given: 현장 반장이 "A라인 2번 부품 외관 불량, 스크래치 발견"이라고 발화한 음성 파일(WAV, 10초)이 주어짐
- When: `transcribeVoice` Server Action을 호출함
- Then: raw_text에 발화 내용이 전사되고, structured_data에 `{ production_line: "A라인", part_number: "2번", defect_type: "외관 불량", description: "스크래치 발견" }`이 추출되며, confidence ≥ 0.92이고, 응답 시간이 3초 이내(p95)이다.

**Scenario 2: 소음 환경 음성 인식**
- Given: 80dB 이상의 배경 소음이 포함된 음성 파일이 주어짐
- When: STT 변환을 수행함
- Then: WER(Word Error Rate) ≤ 8%, CSR(Command Success Rate) ≥ 92%를 달성한다.

**Scenario 3: 인식 실패 시 재시도 유도**
- Given: confidence < 0.70인 인식 결과가 발생함
- When: STT 결과가 반환됨
- Then: `{ success: false, retry_count: 1, message: "음성을 다시 말씀해 주세요" }` 형태의 재시도 유도 응답을 반환한다.

**Scenario 4: 3회 연속 실패 시 Fallback**
- Given: 동일 세션에서 3회 연속 confidence < 0.70이 발생함
- When: 3번째 실패가 감지됨
- Then: `{ fallback_required: true, message: "수동 입력 모드로 전환합니다" }` 이벤트가 발행되어 수동 입력 UI로 전환된다.

**Scenario 5: 대용량 오디오 파일 감지**
- Given: 5MB 크기의 WAV 파일이 업로드됨
- When: 파일 크기 검증이 수행됨
- Then: 413 Payload Too Large와 함께 "Supabase Storage Direct Upload를 사용하세요" 안내가 반환된다.

**Scenario 6: Gemini API 장애 시 재시도**
- Given: Gemini API가 503 에러를 반환함
- When: STT 요청이 실패함
- Then: 지수 백오프(1초→2초→4초)로 최대 3회 재시도하고, 모두 실패 시 503과 `code: "AI_ENGINE_ERROR"` 에러를 반환한다.

## ⚙️ Technical & Non-Functional Constraints
- 성능: p95 latency ≤ 3초 (REQ-NF-002). Gemini Flash 모델 사용으로 속도 최적화
- 정확도: WER ≤ 8%, CSR ≥ 92% (Appendix A.2 측정 프로토콜 기준)
- 비용: Gemini Free Tier (15 RPM, 1,500 RPD) 한도 내 (REQ-NF-018)
- 보안: 음성 데이터 로깅 시 원본 저장 금지, 전사 텍스트만 기록
- 개인정보: Edge-side 가명처리 후 Cloud 전송 필수 (REQ-NF-PRIV-003 — PRIV-003 태스크에서 구현)
- 페이로드: Vercel 서버리스 4.5MB 제한 (§3.6.2). 초과 시 Direct Upload 유도
- 프레임워크: Vercel AI SDK + Gemini 1.5 Multimodal (C-TEC-005, C-TEC-006)
- 재시도: 외부 서비스 실패 시 지수 백오프 + Jitter, 최대 3회 (REQ-NF-027)
- AI 제어: Gemini 출력 Zod 검증, 신뢰도 기반 분기 (REQ-NF-029)

## 🏁 Definition of Done (DoD)
- [ ] 모든 Acceptance Criteria를 충족하는가?
- [ ] Gemini Multimodal STT가 한국어 음성을 정상 인식하는가?
- [ ] 신뢰도 기반 3단계 분기(정상/경고/실패)가 동작하는가?
- [ ] 3회 연속 실패 시 Fallback 이벤트가 발행되는가?
- [ ] WER ≤ 8%, CSR ≥ 92% 테스트를 통과하는가?
- [ ] p95 응답 시간 ≤ 3초 벤치마크를 통과하는가?
- [ ] 단위 테스트 및 통합 테스트가 추가되었고 통과하는가?
- [ ] ESLint / 정적 분석 경고가 없는가?
- [ ] API 명세서(OpenAPI)가 최신화되었는가?

## 🚧 Dependencies & Blockers
- **Depends on:** DB-003 (RAW_DATA 테이블), API-003 (Edge Sync DTO), AUTH-C-001 (인증 기반)
- **Blocks:** ZUI-C-003 (Edge→Cloud 실시간 동기화), ZUI-C-006 (AI 인식 실패 Fallback), PRIV-001 (민감정보 동의 프롬프트), PRIV-003 (Edge-side 가명처리), EDGE-UI-001 (음성 입력 인터페이스), TEST-ZUI-001 (STT WER/CSR 검증), TEST-ZUI-004 (Fallback 전환 테스트)
