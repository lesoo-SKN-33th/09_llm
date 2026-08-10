# 03. OpenAI API — 이론 요약

> 대상 노트북: `01_chat_api_conversation_streaming`, `02_tts`, `03_stt`, `04_embeddings`, `05_moderation`, `06_function_calling`, `07_openai_api_review_voice_faq_assistant`

## 1. Chat Completions API — 대화, 스트리밍, 토큰/비용

### 메시지와 요청 값
Chat Completions API는 대화 이력인 `messages`를 입력으로 받아 assistant 메시지 하나를 반환한다. 서버가 호출 사이의 대화를 자동으로 기억하지 않으므로 **애플리케이션이 이전 메시지를 순서대로 다시 보내야 한다**.

- `system`/`user`/`assistant` 역할 구분은 02장과 동일
- 주요 값: `model`, `messages`, `stream`(True면 부분 응답 청크 수신)
- 신규 프로젝트는 Responses API가 기본 권장 경로지만, 기존 `messages` 기반 서비스 이해를 위해 Chat Completions 구조도 함께 학습한다.

### 대화 이력 누적
반복 대화에서는 사용자 입력을 `user` 메시지로 추가하고, 모델의 완성 응답을 `assistant` 메시지로 다시 추가해야 다음 질문에서 모델이 직전 대화를 함께 참조한다.

### 스트리밍
- `stream=True`는 응답을 청크 반복자로 바꾼다. 각 청크의 `delta.content`를 순서대로 받는다.
- 화면 출력만 하면 다음 턴에 넣을 완성 응답이 없으므로, 조각을 리스트에 모아 `"".join()`으로 합쳐 `assistant` 메시지로 저장해야 한다.

### 토큰과 비용
- 토큰은 모델이 처리하는 텍스트 단위이며 요청·응답 토큰 수가 비용에 영향을 준다.
- `tiktoken`으로 요청 전 토큰 수를 근사할 수 있지만(`o200k_base` 인코딩 등), **실제 청구는 응답의 `response.usage`(`prompt_tokens`, `completion_tokens`, `total_tokens`)를 우선 사용**한다.
- 비용 = (입력 토큰 / 1M × 입력 단가) + (출력 토큰 / 1M × 출력 단가). 가격·처리 등급은 바뀌므로 실행 전 공식 Pricing 문서를 확인한다.

### Responses API로의 전환
Chat Completions는 계속 지원되지만 OpenAI는 신규 프로젝트에 **Responses API**를 권장한다.

- `input`으로 요청, `output_text`로 결과 텍스트 획득
- `previous_response_id`로 이전 응답을 이어 멀티턴 구성
- `reasoning.effort`로 추론 수준 조정
- 텍스트·이미지·도구 호출·멀티턴 상태를 하나의 인터페이스에서 처리

## 2. Text-to-Speech(TTS)

TTS는 입력 텍스트를 음성으로 변환하는 기능이다(안내 방송, 접근성 읽기, 콘텐츠 내레이션 등).

| 용도 | 모델 |
|---|---|
| 단순 파일 TTS(저지연) | `tts-1` |
| 단순 파일 TTS(고품질 비교) | `tts-1-hd` |
| 톤·억양·속도 등 지시형 TTS | `gpt-4o-mini-tts` |
| Chat Completions의 audio in/out | `gpt-audio-1.5` |
| 실시간 음성 에이전트 | `gpt-realtime-2.1` (Realtime API) |

- `voice`로 말투·음색 선택(예: `nova`, `ash`).
- `client.audio.speech.with_streaming_response.create(model, voice, input, response_format, speed)` → `stream_to_file(path)`로 MP3 저장.
- `speed`는 통상 0.25~4.0 범위에서 조정 가능.
- 비용은 입력 문자 수 기준(예: `tts-1`은 100만 자당 $15). 짧은 문장도 반복 실행이 누적되면 비용이 커진다.
- **생성 음성이 AI로 만들어졌음을 사용자에게 명확히 알려야 한다.**

## 3. Speech-to-Text(STT)

STT(ASR)는 음성을 텍스트로 전사하는 기능이다(회의록 초안, 자막, 음성 검색 등).

| 용도 | 모델 |
|---|---|
| 일반 파일 전사 | `gpt-transcribe` |
| 단어·구간 타임스탬프, 영어 번역 | `whisper-1` |
| 화자 구분 | `gpt-4o-transcribe-diarize` |
| 연속 입력(실시간) | Realtime transcription |

- 입력 파일은 25MB 이하, 지원 형식은 mp3/mp4/mpeg/mpga/m4a/wav/webm.
- `client.audio.transcriptions.create(model="gpt-transcribe", file=f)` → `transcription.text`.
- **문맥 힌트로 정확도 향상**: `prompt`(녹음의 주제·상황을 자연어로 설명), `keywords`(고유명사·전문용어 목록), `languages`(ISO 639-1 코드의 입력 후보 언어). 힌트는 정답을 강제하지 않으므로 실제 발화와 결과를 반드시 비교해야 한다.
- 비용은 분당 과금(예: `gpt-transcribe`는 분당 $0.0045).

## 4. Embeddings — 의미 기반 검색

임베딩은 텍스트의 의미를 실수 벡터로 바꾸는 표현 방식이다. 의미가 가까운 문장은 벡터 공간에서도 가까워지므로 검색·추천에 활용한다.

- `text-embedding-3-small`: 기본 1,536차원 / `text-embedding-3-large`: 기본 3,072차원. `dimensions`로 출력 차원을 줄일 수 있으며, 줄이면 저장·검색 비용과 검색 품질이 함께 달라질 수 있다.
- **MTEB 벤치마크**: 분류·군집화·검색·의미 유사도 등을 평가하는 벤치마크 모음. 리더보드는 데이터셋·구성에 따라 달라지므로 고정 순위를 보장하지 않는다. 서비스 언어·문서 길이·질의 유형과 유사한 검증 데이터로 직접 비교해야 한다.
- **검색 절차**: 문서와 질의를 같은 모델로 임베딩 → `cosine_similarity`가 높은 문서 반환. 질의와 문서는 반드시 같은 임베딩 모델을 사용해야 차원과 벡터 공간이 일치한다.
- 실전에서는 검색 대상 문서(제목+본문 등)를 하나의 문자열로 결합해 문서당 벡터 하나를 생성하고, `top_n`개를 유사도 순으로 반환하는 검색 함수를 구성한다.

## 5. Moderation — 콘텐츠 정책 검사

Moderation은 입력 텍스트·이미지를 정책 관련 범주로 검사하는 API이다. `omni-moderation-latest`는 텍스트와 이미지를 함께 검사하며 무료로 제공된다.

### 세 가지 신호
- `flagged`: 모델의 전체 판정(Boolean). 서비스 차단·검토 흐름은 이 값과 자체 정책을 함께 사용한다.
- `categories`: 범주별 Boolean 신호(어떤 위험 유형인지).
- `category_scores`: 범주별 연속 점수. **임계값은 서비스 정책과 검증 데이터에서 별도로 정한다.**

### 주요 범주
`harassment/threatening`, `hate/threatening`, `illicit/violent`, `self-harm`, `self-harm/intent`, `self-harm/instructions`, `sexual/minors`, `violence/graphic` 등.

### 이미지 검사
`{"type": "image_url", "image_url": {"url": ...}}` 형식으로 이미지 URL 또는 base64 데이터 URL을 입력할 수 있다.

### 결과 해석 시 주의
`flagged=True`는 위험 가능성이 감지되었다는 뜻이지, 사용자 제재나 법률 판단을 확정하는 결과가 아니다. 실제 서비스에서는 다음 중 하나를 정책으로 결정한다: 허용 / 차단 / 관리자 검토 / 지원 정보 안내.

### 통합 경로
Responses API 생성 요청의 최상위 `moderation`에 모델을 지정하면 입력과 생성 출력의 moderation 신호를 함께 받을 수 있다. `response.moderation.input`/`.output`은 오류 객체가 될 수 있으므로 `type == "error"` 여부를 먼저 확인해야 한다. 스트리밍에서는 moderation 점수가 부분 출력 조각이 아니라 전체 출력이 준비된 뒤 도착한다.

## 6. Function Calling — 도구 호출

Function Calling은 모델이 함수 이름과 JSON 인자를 선택해 애플리케이션에 전달하고, 애플리케이션이 이를 검사·실행한 뒤 결과를 모델에 돌려주어 최종 자연어 응답을 만드는 기능이다. **모델은 함수를 직접 실행하지 않는다.**

### 처리 순서
1. 애플리케이션이 함수 이름·설명·JSON Schema를 모델에 전달
2. 모델이 함수 이름과 JSON 인자를 반환(`tool_calls` / `function_call`)
3. 애플리케이션이 **허용된 함수만** 실행 (모델이 만든 인자는 신뢰하지 않고 검증)
4. 함수 실행 결과를 같은 호출 ID와 함께 모델에 전달
5. 모델이 도구 결과를 근거로 최종 텍스트 생성

### strict 스키마 권장 설정
- 모든 속성을 `required`에 포함
- `additionalProperties: false`
- 반환된 이름·인자를 실행 전 재검사

### Chat Completions vs Responses API 비교
| 구분 | Chat Completions | Responses API |
|---|---|---|
| 함수 정의 위치 | `function` 안에 중첩 | `name`, `description`, `parameters`를 최상위에 직접 작성 |
| 호출 요청 확인 | `message.tool_calls` | `response.output`의 `function_call` 항목 |
| 결과 전달 | `role="tool"`, `tool_call_id` 메시지 | `function_call_output` 항목, `call_id` |
| 최종 텍스트 | `choices[0].message.content` | `response.output_text` |

`gpt-5.6-luna`를 Chat Completions 함수 도구와 함께 쓸 때는 `reasoning_effort="none"`을 명시해야 호환된다.

## 7. 종합 파이프라인 — 음성 FAQ 상담 도우미

여러 API의 출력이 다음 API의 입력으로 이어지는 파이프라인 설계 예시.

```
질문 텍스트 → TTS(테스트 음성) → STT(전사문) → Moderation(입력 검사)
→ Embeddings(FAQ 의미 검색) → Responses API(근거 기반 답변 생성) → TTS(안내 음성)
```

- 각 API는 서로 다른 반환 형태를 가지므로 다음 단계에 맞게 변환해야 한다: 파일 경로(Speech) → 문자열(Transcription) → Boolean/점수(Moderation) → 숫자 벡터(Embeddings) → 텍스트(Responses).
- Moderation의 `flagged=True`이면 자동 답변 생성을 중단하고 사람 검토로 넘기는 정책을 적용할 수 있다(수업용 단순 정책이며, 실제 서비스는 별도 정책 설계 필요).
- 검색 증강 답변은 검색된 FAQ 범위 안에서만 답하도록 `instructions`로 제한하고, 근거가 없으면 담당자 확인이 필요하다고 답하게 한다.
- Cosine Similarity 점수가 높다고 해서 반드시 정답은 아니므로 실제 관련성도 함께 확인해야 한다.

## 8. 공통 원칙

- API 키(`OPENAI_API_KEY` 등)는 `.env`로 관리하고 출력하지 않는다.
- 실제 API 호출은 비용이 발생하므로 같은 셀을 불필요하게 반복 실행하지 않는다.
- 모델 이름·가격·지원 기능은 자주 바뀌므로 실행 전 공식 문서로 재확인한다.
