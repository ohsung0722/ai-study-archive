# Multimodal
blog: multimodal_blog/llm_day2_blog.md
blog_title: [LLM 2일차] 보고, 그리고, 듣고, 말하는 AI: OpenAI 멀티모달 API
summary: 텍스트, 이미지, 음성 등 서로 다른 형태의 데이터를 함께 입력받고 처리하거나 출력할 수 있도록 만든 AI 기술. LLM 모델을 중심으로 Vision, 이미지 생성, TTS, STT 등의 모델이나 API를 연결하여 구현이 가능.

<!--
개념 노트 형식
- "## 개념 지도": "### 단계 이름"마다
    related: 관련 면접 질문 id (쉼표 구분)
    핵심: 이 단계를 한 문장으로
    본문 (자유 마크다운: 설명, 예, 직관)
    막힌 점: 다음 단계가 나온 이유 (마지막 단계는 생략 가능)
- "## 한 장 요약": 자유 마크다운
- "## 이해 확인": "### Q. 질문"마다 id, type(개념 | 만약에 | 생각해보기 | 연결 | OX | 계산), level(기본 | 심화), related,
  필요하면 문제 본문, 그리고 "#### 풀이" 아래에 답
-->

## 개념 지도

### 글만 아는 LLM
related: llm-request-flow, llm-next-token-prompt, llm-chat-messages, mm-multimodal-definition
핵심: 초창기의 LLM은 텍스트를 입력 받아 다음 토큰을 예측하고 텍스트를 출력하는 모델로, 이미지나 음성을 직접 이해하거나 처리할 수 없었다. 
- **기존 LLM의 입출력**: `client.responses.create(model=..., input="...")`처럼 텍스트를 입력하고, `output_text`를 통해 생성된 텍스트 데이터를 받는 방식.
- **대화 기록**: `{"role": ..., "content": "문자열"}` 형태의 리스트로 관리하며, content에는 텍스트만 포함.
- **한계**: 실제 서비스에서는 '이 사진이 뭐야?', '내 말을 영어로 통역해줘', '이 속담을 그림으로 그려줘' 처럼 텍스트 뿐 아니라 이미지와 음성 등 다양한 형태의 데이터를 처리해야 함.
- **직관적인 비유**: LLM이 두뇌라면 Vision은 눈, STT는 귀, 이미지 생성은 손, TTS는 입에 해당. 기존 LLM에 이러한 기능을 연결하거나 모델 자체를 확장하면 다양한 형태의 데이터를 처리할 수 있음.
막힌 점: 텍스트뿐 아니라 이미지, 음성 등 다양한 형태의 데이터를 처리하는 AI를 무엇이라고 부르고, 이를 구현하려면 어떤 기술이 필요할까?

### Multimodal: 입력과 출력의 형태부터 정한다
related: mm-multimodal-definition, mm-api-map
핵심: Multimodal은 텍스트, 이미지, 음성 등 다양한 형태의 데이터를 입력받아 처리하거나 생성하는 AI 기술. 구현하려는 기능에 따라 입력과 출력의 형태를 먼저 정하고, 적합한 모델과 API를 선택해야 함. 
- **Unimodal**: 한 가지 형태의 데이터만 처리 (ex. GPT-3.5의 text to text).
- **Multimodal**: 텍스트, 이미지, 영상, 음성 등을 입력이나 출력으로 처리할 수 있는 모델
- **주의할 점**: 모든 Multimodal 모델이 모든 형태의 데이터를 이해하고 생성하는 것은 아님. 하나의 모델이 여러 형태의 데이터를 직접 처리할 수도 있지만, 기능별로 서로 다른 모델이나 API를 연결해 구현할 수도 있음.
- 주요 API와 역할:

| API | 입력 → 출력 | 주요 기능 |
|---|---|---|
| Responses API | 텍스트 + 이미지 → 텍스트 | Vision(이미지 이해 및 분석) |
| Images API | 텍스트 (+ 이미지) → 이미지 | Image generate, edit |
| Audio API | 오디오 ↔ 텍스트 | STT(받아쓰기), 번역, TTS(읽어주기) |
| Realtime API | 실시간 오디오 · 텍스트 | 스트림으로 바로 음성 응답 (소개만) |

막힌 점: 모델은 이미지나 음성을 어떻게 이해할까? 텍스트와 달리 이미지와 음성은 그대로 토큰화하기 어려우므로, 모델이 처리할 수 있는 숫자 표현으로 변환하는 과정이 필요. 

### Image to Text: 이미지를 조각내어 이해하고 텍스트로 변환
related: mm-image-to-text, mm-cross-attention, mm-vlm-connect, mm-image-tokens
핵심: Vision Encoder가 이미지를 여러 조각(Patch)으로 나누어 벡터로 변환하고, LLM이 이미지 정보를 참고하여 텍스트를 생성하는 방식. 모델 구조에 따라 Cross-Attention을 사용하거나 이미지 벡터를 LLM의 입력에 직접 연결(Self-Attention)하기도 함.
- **학습**: 대량의 이미지와 텍스트 데이터를 학습하면서 이미지 속 객체의 위치, 형태, 색상 및 객체 간 관계를 언어와 연결하는 방법을 학습
- **이미지 분할(Patch Embedding)**: ViT 처럼 이미지를 일정한 크기의 Patch로 나누고, 각 Patch를 벡터로 변환. 
    - 예를 들어, 224×224를 16×16으로 자르면 14 × 14 = **196개**의 Patch 생성. 
    - 각 Patch는 벡터로 변환되어 이미지 토큰처럼 사용되며, 위치 정보를 함께 활용해 이미지의 공간적 구조 파악. 
- **Cross-Attention**: 
    -  `Query`: 사용자 질문 + 지금까지 생성된 텍스트의 Hidden State 
    - `Key`: 이미지 Patch 벡터 기반 검색용 이름표
    - `Value`: 이미지 Patch 벡터 기반 실제 시각 정보 
    - 동작 예시 : 'dog'을 쓸 때는 개 영역 Patch, 'a truck'은 트럭 영역 Patch에 높은 attention 부여. 'next to'와 같은 관계어는 두 객체 영역에 고르게 걸림.
- **전체 생성 흐름**: 
    - 1단계(이미지 인코딩): 이미지를 패치 벡터(Key, Value)로 변환해 메모리에 고정. (최초 1회만 실행)
    - 2단계(질문 처리/Prefill): 사용자 질문(ex. '이 이미지 설명해줘')을 한 번에 읽어 의도 파악
    - 3단계(단어별 반복 생성): 새 단어를 하나 뱉을 때마다 디코더 층 내부에서 Self-Attention(지금까지 쓴 문맥 정리) -> Cross-Attention(이미지의 필요한 부분 참조) -> FFN 사이클이 `<END>` 토큰 나올 때까지 반복. 
- 쉬운 예: 바다, 산, 해, 하늘을 찾고 → "하늘 중간에 해"처럼 관계를 붙이고 → 문장 완성
- 보충: 해상도가 두 배면 Patch 개수가 4배(448×448 → 784개)라 토큰과 비용이 커진다. 실제 VLM은 CLIP ViT에서 추출한 이미지를 projection Layer로 LLM의 입력 차원에 맞게 변환한 뒤 텍스트 토큰과 함께 입력하여 처리.
막힌 점: 이미지 정보를 활용하더라도 모델이 항상 정확한 설명을 생성하는 것은 아님. LLM은 이미지 정보를 참고해 다음 토큰을 예측하므로, 실제 이미지에 없는 객체나 상황을 그럴듯하게 설명하는 Hallucination 발생 가능

### Vision의 한계: 그럴듯함과 정확함은 다르다
related: mm-vision-limits, mm-vision-hallucination
핵심: Vision은 학습한 패턴으로 가장 그럴듯한 답을 낼 뿐이라, 전문 판독이나 읽기 어려운 글자에는 약하고 없는 것을 말하기도 한다.
- **의학적 분석**: CT, X-ray 등의 의료 영상을 분석할 수는 있지만, 일반적인 vision 모델은 전문적인 의료 판독을 대체하기는 어려움.
- **비라틴 문자 인식**: 한글 등 비라틴 문자는 모델과 학습 데이터에 따라 영어보다 인식 정확도가 낮을 수 있음
- **읽기 어려운 텍스트**: 크기가 작거나 기울어지고 흐릿한 글씨는 정확한 인식이 어려움
- **Hallucination(환각)**: 이미지에 존재하지 않는 객체나 상황을 실제로 존재하는 것처럼 설명하는 현상. 예를 들어, 식탁 이미지 분석할 때, 실제 이미지에는 포크가 없지만 학습된 데이터에서는 식탁과 포크가 자주 등장했던 것을 기반으로 포크가 있다고 설명할 수 있음.
막힌 점: 한계를 알고 사용해야 함. 실제 API 요청에 이미지를 어떤 형태로 전달하고 모델의 응답을 받아오는가.


### 이미지 입력: content 배열, URL과 Base64
related: mm-message-structure, mm-url-vs-base64, mm-base64
핵심: Vision 모델에 이미지를 전달하려면 `content`를 문자열 대신 배열로 만들어 텍스트와 이미지를 함께 입력해야 함. 이미지는 URL 또는 Base64로 인코딩한 데이터의 형태로 전달 가능.
-  `content`구조: `[{"type": "input_text", ...}, {"type": "input_image", "image_url": ...}]`
- **content type**은 `input_text`, `input_image`, `input_audio`, `input_file` 네 가지. 
- **role**: 메시지의 역할을 구분하는 속성
    - `system`(신형 API에서는 `developer`) : 모델의 동작 방식과 지침 설정
    - `user`: 사용자 입력
    - `assistant`: 모델의 응답
    - `tool`: 도구 실행
- 인터넷 그림은 URL을 넘기면 쉽게 전달 가능.
- 내 컴퓨터 사진은 **Base64**(알파벳, 숫자, `+`, `/` 64글자)로 인코딩한 뒤 `"data:image/jpeg;base64,"` 뒤에 붙이기.
- 코드 예시
```py
response = client.responses.create(
    model="gpt-4.1",
    input=[
        {
            "role": "user",
            "content": [
                {
                    "type": "input_text",
                    "text": "이 사진에 무엇이 있나요?"
                },
                {
                    "type": "input_image",
                    "image_url": "https://example.com/image.jpg"
                }
            ]
        }
    ]
)

print(response.output_text)
```
막힌 점: 텍스트를 입력하여 새로운 이미지를 생성하는 방식은 어떻게 하지?

### Text to Image: 노이즈를 제거하며 이미지 생성
related: mm-text-to-image, mm-diffusion, mm-latent-space, mm-generate-vs-edit
핵심: Diffusion 모델은 텍스트 프롬프트를 벡터로 변환하고, 무작위 노이즈에서 시작해 프롬프트의 정보를 참고하면서 노이즈를 단계적으로 제거하여 이미지를 생성하는 방식.
- **이미지 생성 과정**
    1. Text Encoder: 입력된 프롬프트를 모델이 이해할 수 있는 벡터로 변환
    2. Noise 생성: 무작위 노이즈를 초기값으로 사용
    3. Denoising: U-Net 등의 모델이 프롬프트 정보를 참고하여 노이즈를 예측하고 단계적으로 제거하는 과정 반복
    4. Latent Space: 이미지의 픽셀 공간보다 작은 잠재 공간에서 Denoising을 수행하여 연산량 절감
    5. VAE Decoder: 최종 잠재 표현을 실제 픽셀 이미지로 복원
- **학습**: 원본 이미지에 단계적으로 노이즈를 추가하고, 모델이 추가된 노이즈를 예측하도록 학습
- **생성**: 학습 과정과 반대로 무작위 노이즈에서 시작해 예측한 노이즈를 단계적으로 제거
- **Latent Space(잠재 공간)**: 원본 이미지를 압축하여 표현한 저차원 공간으로, 픽셀 공간에서 직접 연산하는 것보다 효율적으로 이미지 처리 가능.
    - 예를 들어 Stable Diffusion의 대표적인 VAE 구조는 이미지의 가로와 세로를 각각 1/8로 축소, 4개 채널의 잠재 표현 사용
    - Latent Space에서 Denoising을 완료한 후 VAE Decoder를 통해 최종 이미지 복원
- **Generate**는 텍스트 프롬프트를 바탕으로 이미지 생성, **Edit**은 기존 이미지와 프롬프트를 기반으로 이미지의 특정 요소를 수정하거나 변형 
막힌 점: 실제 Images API에서 사용할 모델과 입력 조건 및 생성된 이미지의 반환 형식은?


### 이미지 생성 규칙, 모델 선택, Base64로 디코딩
related: mm-image-rules-model, mm-b64-json
핵심: Images API를 사용할 때는 이미지 형식과 크기, 콘텐츠 정책 등을 확인하고, 생성 품질과 비용을 고려해 적합한 모델을 선택해야 함. 생성된 이미지는 Base64 문자열로 반환되므로, 디코딩하여 이미지 파일로 저장
- 지원 형식: PNG, JPEG, WEBP, GIF / 크기 제한 50MB (모델마다 다름) / 콘텐츠 정책(저작권 침해나 성적인 요소 제한)
- 이미지 생성 및 저장
```py
import base64

result = client.images.generate(
    model="gpt-image-1-mini",
    prompt="고양이가 우주복을 입고 달 위에 서 있는 모습",
    size="1024x1024"
)

image_base64 = result.data[0].b64_json
image_bytes = base64.b64decode(image_base64)

with open("generated.png", "wb") as f:
    f.write(image_bytes)
```
- `images.generate(model, prompt, size="1024x1024")`: 프롬프트를 전달하여 이미지 생성 요청
- `result.data[0].b64_json`: 생성된 이미지의 Base64 인코딩 문자열
- `base64.b64decode()`: Base64 문자열을 원본 이미지의 바이너리 데이터로 복원
- `open(...,'wb')`: 복원된 바이너리 데이터를 파일로 저장
막힌 점: 이미지 생성 말고 이미 만들어진 이미지의 일부를 수정하는 기능은 없나?

### Image Editing: 기존 이미지를 참고해 수정
related: mm-generate-vs-edit, mm-edit-exp
핵심: `images.edit()`는 원본 이미지와 프롬프트를 함께 입력받아 원하는 부분을 수정하는 기능. 원본 이미지의 구도, 스타일 등의 특징을 참고하면서 특정 요소를 변경할 수 있음.
- **이미지 입력**: `image=open("elephant.png", "rb")` 처럼 파일을 바이너리 읽기 모드(rb)로 열어 이미지 데이터 전달
- **프롬프트**: 수정할 대상과 변경할 내용을 구체적으로 작성
- **원본 유지**: 프롬프트에서 변경을 요청하지 않은 요소는 가능한 원본을 유지하도록 생성되지만, 모든 요소가 정확하게 보존되는 것은 아님
- **예시**: '캐릭터를 코끼리에서 호랑이로 바꿔줘' -> 코끼리가 호랑이로 변경되지만, 그 외 기존 그림체 대부분은 유지
- generate로 "호랑이 신사"를 다시 그리면 시작 노이즈가 달라서(Seed가 달라진다고 표현) 옷, 소품, 화풍이 같다는 보장이 없다.
- 결과 형태는 generate와 같다. `b64_json`을 디코딩해 `"wb"`로 저장
막힌 점: 이제 이미지가 아닌 소리를 문자로 바꿔보기

### 소리도 숫자다: 파형과 스펙트로그램
related: mm-audio-api, mm-sound-representation
핵심: 소리는 시간에 따른 진폭을 일정한 간격으로 측정한 숫자 데이터로 표현할 수 있음. 음성 모델은 이러한 파형을 직접 처리하거나, 시간에 따른 주파수 성분을 나타내는 스펙트로그램으로 변환해 음성의 특징을 학습.
- Audio API 주요 기능
| 기능 | 입력 -> 출력 | 설명 |
|---|---|---|
| TTS(Text-to-Speech) | 텍스트 → 음성 | 입력된 텍스트를 자연스러운 음성으로 변환 |
| Transcription(STT) | 음성 → 텍스트 | 음성에서 발화 내용을 인식하여 텍스트로 변환 |
| Translation | 음성 -> 번역된 텍스트 | 음성을 인식하고 다른 언어로 변환 |
- 파형: 16kHz로 녹음하면 1초에 숫자 16,000개. 이 상태로는 무슨 소리인지 파악하기 어려움.
- 스펙트로그램: 가로 = 시간, 세로 = 주파수, 밝기 = 세기. 
    - 자음 같은 잡음은 세로로 길쭉한 띠, 모음은 가로로 몇 줄 밝은 띠를 남기고, 모음마다 줄의 높이가 달라 "아"와 "이"가 구분된다.
- 소리를 이미지화 했으니 Vision처럼 조각내 Encoder에 넣을 수 있음
막힌 점: TTS부터 자세히 살펴보기

### TTS: 텍스트를 음성으로 변환
related: mm-tts
핵심: TTS는 입력된 텍스트를 자연스러운 음성으로 변환하는 기술. Audio API의 `audio.speech.create`를 사용하면 텍스트를 음성으로 생성하고, 목소리와 말투, 속도 등 조절 가능
- 음성 생성 예시
```py
speech = client.audio.speech.create(
    model="gpt-4o-mini-tts",
    voice="alloy",
    input="안녕하세요, 좋은 날씨예요!",
    instructions="밝고 친근한 말투로 말해줘.",
    speed=1.0,
    response_format="mp3"
)

speech.write_to_file("speech.mp3")
```
- 이미지 생성 API와 달리 Base64 문자열을 디코딩할 필요 없이 파일로 저장 가능. 
- `instructions`(최대 4096자): "차분한 뉴스 앵커처럼", "신난 아이처럼". `speed` 0.25~4.0: 60초 글이 0.25면 4분, 4.0이면 15초
- input도 최대 4096자, voice는 alloy, coral, nova, marin, cedar 등, response_format은 mp3, opus, aac, flac, wav, pcm
막힌 점: STT 자세히 살펴보기.

### Transcription(STT): 음성을 텍스트로 변환
related: mm-transcription-exp, mm-audio-file-size
핵심: STT는 음성 파일을 입력받아 발화 내용을 텍스트로 변환하는 기술. Audio API의 `audio.transcriptions.create()`를 사용하며, 음성 파일을 바이너리 모드(rb)로 열어 전달하면 인식된 텍스트를 반환
- 음성 인식 예시
```py
with open("speech.mp3", "rb") as audio:
    transcription = client.audio.transcriptions.create(
        model="gpt-4o-mini-transcribe",
        file=audio
    )

print(transcription.text)
```
- 문장 부호는 억양에서 추측해야 하는 정보이기 때문에 정확도 떨어짐
- 파일 한도 25MB(OpenAI 문서, 강의 자료에는 없음). CD 음질 WAV는 1분 약 10MB라 2분 반, 16kHz 모노는 1분 1.83MB라 약 14분, MP3 64kbps는 약 55분
막힌 점: 음성을 다른 언어의 텍스트로 변환하는 Translation 기능은 어떻게 하는걸까

### Audio Translation: 음성을 다른 언어로 번역
related: mm-whisper-translation-exp
핵심: Audio Translation은 음성을 입력받아 다른 언어의 텍스트로 변환하는 기능. OpenAI의 `audio.translations.create()`는 whisper-1 모델을 사용하며, 다양한 언어의 음성을 영어 텍스트로 변환 가능. 
- 평가: "조금 도태된 모델", 슬라이드 제목은 "Whisper는… 조금 실전에서 쓰기 어렵긴 하다"
- 실시간 번역 모델(gpt-realtime-translate 등)은 2배에서 6배까지 비싸다.
- 출력 언어: OpenAI 문서 기준으로 Audio Translation은 여러 언어 음성을 **영어로만** 번역
막힌 점: 번역 전용 모델이 약하다면, 대안은 없을까?

### Transcription + LLM: 음성 인식과 번역 분리
related: mm-transcription-llm, mm-cascade-vs-realtime
핵심: STT 모델로 음성을 텍스트로 변환한 후 LLM에 전달해 번역하는 방식. 각각 다른 모델에 맡기기 때문에 번역 언어와 말투 등 자유롭게 지정 가능
- 구현 예시
```py
with open("speech.mp3", "rb") as audio:
    transcription = client.audio.transcriptions.create(
        model="gpt-4o-mini-transcribe",
        file=audio
    )

response = client.responses.create(
    model="gpt-4.1-mini",
    instructions="너는 한영 번역가이다. 자연스러운 영어로 번역하고 번역문만 출력해.",
    input=transcription.text
)

print(response.output_text)
```
- **한계**: 
    - 처리시간과 API 비용 증가
    - 오류 전파: STT에서 잘못 인식한 내용을 LLM이 그대로 번역하거나 오류가 드러나지 않게 감출 수 있음 
    - 음성 정보 손실: 음성을 텍스트로 변환하는 과정에서 억양, 감정 등 비언어적 정보 손실
막힌 점: 다양한 앱을 예시로 하여 아키텍쳐 생각해보기


### Multimodal Application: LLM과 여러 모델 연결하기
related: mm-proverb-quiz-exp, mm-artist-exp, mm-voice-apps-exp
핵심: LLM을 중심으로 Vision, 이미지 생성, STT, TTS 등의 모델을 연결해 Multimodal 어플리케이션 구현해보기
- **속담 퀴즈**: 속담 100개 중 무작위 → Images API → 사용자 답 → 정답 여부·점수(`gr.State`) → 다음 문제. "글자 없이 그려줘"라고 안 하면 정답이 그림에 글자로 박힐 수 있다. "우물 안 개구리"는 쉽고 "모르면 약이요, 아는 게 병"은 거의 불가능하다.
- **그림판 Artist**: ImageEditor composite → Base64 → Vision(I2T)으로 설명과 화가용 프롬프트 → Images API(T2I) → 파일로 저장해 출력
- **통역기**: 마이크 → Transcription → LLM 번역(Dropdown 언어를 instructions에) → TTS → 글과 소리
- **영어회화 퀴즈**: 녹음 → Transcription → LLM이 점수·피드백·모범 답안(JSON) → 모범 답안 TTS(speed 0.75 정도)
- 자율 프로젝트 모델은 gpt-4o-mini, gpt-image-1-mini, gpt-4o-mini-tts, gpt-4o-mini-transcribe 넷으로 고정했다.
- 다음 이야기: 오늘 "나중에"로 미룬 Tool calling(Responses API에서 도구로 이미지 만들기)과, LLM이 다른 모델과 도구를 직접 골라 부르는 Agent 쪽으로 이어진다.

## 한 장 요약

### 한 문장 흐름
글만 아는 LLM → 입출력 형태로 API 고르기 → 그림은 조각내 읽고 Cross-Attention으로 쓰면서 다시 본다(그럴듯함이지 정확함은 아님) → content 배열에 URL이나 Base64로 싣기 → 글에서 그림은 잠재 공간에서 노이즈 걷어 내기 → 싼 모델 고르고 b64_json 디코딩 → edit는 원본 참고 → Gradio Image·Gallery·ImageEditor → 소리는 파형에서 스펙트로그램으로 → TTS로 말하기, Transcription으로 받아 적기 → whisper 번역은 약하니 받아쓰기 + LLM → gr.Audio → LLM을 가운데 둔 앱

### 핵심 숫자와 식
| 무엇 | 값 또는 식 | 비고 |
|---|---|---|
| 조각 수 | (224 ÷ 16)² = 196 | 448×448이면 784 (4배) |
| Base64 크기 | `⌈n / 3⌉ × 4` 글자 ≈ 원본 × 4/3 | dog.jpg 163,759 → 218,348 |
| 잠재 공간 | H/8 × W/8 × 4, 픽셀의 1/48 | 1024² → 3,145,728 대 65,536 |
| 노이즈 섞기 | `x_t = √ᾱ_t · x_0 + √(1 − ᾱ_t) · ε` | t = 999에서 원본 약 1% |
| 이미지 출력 단가 | gpt-image-1-mini $8 / 1M 토큰 | 2는 $30, 1.5는 $32 |
| TTS 단가 | gpt-4o-mini-tts $12 / 1M 오디오 토큰 | gpt-audio $64 |
| TTS 재생 시간 | 기본 시간 ÷ speed | 60초 → 0.25면 240초, 4.0이면 15초 |
| 전사 단가 | gpt-4o-mini-transcribe $0.003/분 | gpt-4o-transcribe·Whisper 약 $0.006 |
| WAV 크기 | 샘플링 × 채널 × 2바이트 × 초 | 16kHz 모노 1분 1.83MB |
| 전사 파일 한도 | 25MB | WAV CD 음질 2.5분, 16kHz 모노 14분, MP3 64kbps 55분 |

### 헷갈리는 용어
| 용어 | 뜻 |
|---|---|
| 이해 / 생성 | 그림·소리 → 글 (Vision, STT) / 글 → 그림·소리 (Images API, TTS) |
| Vision Encoder / Text Decoder | 조각 벡터로 무엇이 어디 있는지 정리 / 한 단어씩 쓰며 조각을 다시 봄 |
| Self-Attention / Cross-Attention | 한 시퀀스 안에서 서로 봄 / 글(Q)이 이미지 조각(K, V)을 봄 |
| URL / Base64 / file_id | 서버가 주소에서 가져감 / 바이트를 글자로 적어 요청에 실음 / 먼저 올리고 id로 가리킴 |
| `b64_json` / `write_to_file` | 이미지 결과(디코딩 필요) / TTS 결과를 바로 파일로 |
| `"rb"` / `"wb"` | 파일을 바이트 그대로 읽기(edit, 전사) / 바이트로 쓰기(생성 그림 저장) |
| generate / edit | 프롬프트만으로 새로 / 원본을 참고해 고침 |
| Transcription / Translation | 같은 언어로 받아 적기 / 다른 언어로 옮겨 적기 |
| background / layers / composite | 원본 / 덧그린 레이어 / 합친 결과 |

### 실습 앱 파이프라인
| 앱 | 흐름 | 쓰는 API |
|---|---|---|
| 속담 퀴즈 | 속담 → 그림 → 사용자 답 → 채점·점수 → 다음 | Images |
| 그림판 Artist | 손그림 → 설명·프롬프트 → 작품 | Responses(Vision) → Images |
| 통역기 | 말 → 글 → 번역 → 소리 | Audio(STT) → Responses → Audio(TTS) |
| 영어회화 퀴즈 | 문제 → 녹음 → 글 → 채점·모범 답안 → 발음 | Audio(STT) → Responses → Audio(TTS) |

## 이해 확인

### Q. 강의에서 "멀티모달 모델 하나가 다 해 준다"고 생각하면 헷갈린다고 했습니다. 이해하는 모델과 만들어 내는 모델이 따로 있는 이유는 무엇일까요?
id: mmc-understand-vs-generate
type: 개념
level: 기본
related: mm-multimodal-definition, mm-api-map

#### 풀이
두 일은 **정보가 흐르는 방향과 학습 방식이 반대**이기 때문입니다.
- 이해(Vision, 받아쓰기)는 수십만 개의 픽셀이나 샘플을 압축해 짧은 글로 만든다. 이미지와 설명을 짝지어 다음 단어를 맞히게 학습한다.
- 생성(이미지, TTS)은 짧은 글에서 수백만 개의 픽셀이나 긴 파형을 만들어 낸다. Diffusion은 노이즈를 걷어 내는 법을 배운다.
- 그래서 API도 나뉜다. 그림을 보고 글로 답하면 Responses API, 그림을 받으면 Images API, 소리는 Audio API다. 앱을 만들 때는 입력과 출력의 형태부터 정하고 맞는 API를 고른다.

### Q. Vision Encoder가 사진을 16×16 조각으로 자르는 것은 모델이 이미지를 "읽는" 데 어떤 의미가 있나요?
id: mmc-patch-as-words
type: 개념
level: 기본
related: mm-image-to-text, mm-image-tokens

#### 풀이
이미지를 **단어처럼 다룰 수 있는 토큰의 줄**로 바꾸는 단계입니다. 224×224 사진이 196개 조각 벡터가 되면, Transformer는 글의 단어를 다루듯 이 벡터들을 서로 비교하고(Self-Attention), Decoder는 글을 쓰면서 이 벡터들을 참고(Cross-Attention)할 수 있습니다.
- 픽셀 하나는 의미가 거의 없지만, 16×16 조각에는 선, 질감, 색 같은 지역 정보가 담긴다.
- 픽셀 5만 개를 토큰으로 쓰면 서로 비교하는 계산이 감당할 수 없이 커진다. 196개면 다룰 만하다.
- 조각이 곧 토큰이라, 해상도가 비용과 직결된다는 것도 여기서 나온다.

### Q. Text Decoder가 "a dog sitting next to a bicycle and a truck"을 쓸 때 "next to"는 개와 자전거 양쪽을 보고 "and"는 주목이 고르게 퍼집니다. Cross-Attention이 어떻게 계산되기에 이런 모양이 나올까요?
id: mmc-cross-attn-pattern
type: 개념
level: 심화
related: mm-cross-attention, nlp-cross-attention

#### 풀이
지금까지 쓴 문장 상태가 **Query**, 이미지 조각 벡터들이 **Key와 Value**입니다. Query와 각 Key의 유사도를 softmax로 확률처럼 바꾼 뒤, 그 비율대로 Value를 섞어 다음 단어를 고르는 데 씁니다.
- "dog"을 쓰려는 상태의 Query는 개 조각의 Key와 잘 맞아 그쪽 가중치가 커진다.
- "next to"는 관계를 말하는 단어라 두 물체의 정보가 다 필요하고, 가중치가 개와 자전거 두 곳에 나뉜다.
- "a", "and"는 이미지의 특정 부분과 관련이 없어 모든 Key와 비슷하게 맞고, softmax 결과가 고르게 퍼진다.
- 블로그 그림은 이 흐름을 보여 주려고 손으로 칠한 예시이며 실제 모델 값이 아니다.

### Q. 같은 Vision 모델에 224×224 대신 448×448로 이미지를 넣으면(조각 크기 16×16 유지) 무엇이 달라질까요?
id: mmc-higher-resolution
type: 만약에
level: 심화
related: mm-image-tokens, mm-vision-limits

#### 풀이
- **조각 수**: 28 × 28 = 784개로 4배가 된다. 이미지 토큰이 4배라 입력 비용과 시간이 늘어난다.
- **계산량**: Self-Attention은 조각 쌍마다 계산하니 784² ÷ 196² = 16배가 된다.
- **얻는 것**: 작은 글씨나 작은 물체가 더 많은 픽셀로 남아, 강의가 짚은 "작은 글씨, 흐린 글씨" 한계가 줄어든다.

그래서 사진 분위기를 묻는 일에는 낮은 해상도로 충분하고, 영수증 읽기처럼 세부가 중요한 일에만 높은 해상도를 쓰는 것이 맞습니다. 보충: OpenAI API의 `detail` 옵션이 이 선택을 맡습니다.

### Q. CLIP도 이미지와 글을 함께 다룹니다. CLIP과 이번에 쓴 Vision(Image to Text) 모델은 무엇이 같고 무엇이 다른가요?
id: mmc-clip-vs-vision
type: 연결
level: 심화
related: clip-overview, clip-zeroshot, vit-overview, mm-vlm-connect, mm-image-to-text

#### 풀이
**같은 점**: 둘 다 이미지를 ViT처럼 조각으로 잘라 벡터로 바꾸는 Encoder를 쓰고, 이미지와 언어의 관계를 대량의 이미지-설명 쌍으로 배운다.

**다른 점**은 출력입니다.
- CLIP은 이미지와 문장을 각각 벡터 하나로 만들어 **얼마나 맞는지 점수**를 낸다. 그래서 후보 문장 중 고르기(zero-shot 분류, 검색)는 하지만 문장을 새로 쓰지는 못한다.
- Vision 모델은 Encoder 뒤에 **Text Decoder(언어 모델)**가 있어 질문에 맞춰 문장을 한 단어씩 생성한다.
- 둘은 이어진다. 보충: LLaVA 같은 VLM은 CLIP의 ViT를 이미지 인코더로 그대로 가져와 projection으로 LLM에 붙인다. CLIP이 이미 이미지 벡터를 언어와 맞춰 놓았기 때문에 연결이 쉽다.

### Q. "Vision 모델은 이미지 속 글자를 OCR처럼 정확하게 읽어 낸다."
id: mmc-ox-vision-ocr
type: OX
level: 기본
related: mm-vision-limits, mm-vision-hallucination

#### 풀이
**X.** Vision은 학습한 패턴으로 **가장 그럴듯한 답**을 내는 생성 모델입니다. 강의에서도 한글 같은 비라틴 문자, 작은 글씨, 기울어지거나 흐린 글씨는 성능이 낮을 수 있다고 짚었습니다. 읽지 못한 부분을 그럴듯한 글자로 채워 넣을 수도 있어서, 정확도가 중요한 글자 인식은 결과를 따로 검증해야 합니다.

### Q. 한글 영수증 사진을 찍으면 품목과 금액을 정리해 주는 앱을 Vision으로 만들려고 합니다. 블로그에서 본 한계를 생각하면 어떻게 설계해야 할까요?
id: mmc-receipt-app
type: 생각해보기
level: 심화
related: mm-vision-limits, mm-vision-hallucination, mm-image-tokens

#### 풀이
정답이 하나는 아니지만, 한계 세 가지와 환각을 각각 막는 방향으로 생각해 볼 수 있습니다.
1. **입력 품질**: 한글, 작은 글씨, 기울어진 글씨가 약점이다. 촬영 안내(평평하게, 밝게)를 넣고, 영수증 부분만 잘라 크게 보낸다. 해상도는 높게 쓴다.
2. **출력 형식**: "품목, 수량, 금액을 JSON으로, 읽을 수 없는 칸은 null"처럼 형식과 모를 때의 행동을 정한다. 그럴듯하게 채우는 환각을 줄이려는 것이다.
3. **코드로 검증**: 금액 합계가 영수증의 합계와 맞는지 계산해 보고, 안 맞으면 사용자에게 확인을 요청한다.
4. **사람 확인**: 결과를 수정 가능한 표로 보여 주고 사용자가 고친 뒤 저장한다.

핵심은 Vision의 답을 정답이 아니라 **초안**으로 다루는 것입니다.

### Q. 멀티모달 입력에서 `content`가 문자열이 아니라 배열이 된 이유는 무엇인가요?
id: mmc-content-array
type: 개념
level: 기본
related: mm-message-structure, llm-chat-messages

#### 풀이
한 메시지 안에 **종류가 다른 여러 조각**을 순서대로 담아야 하기 때문입니다. "이 그림을 3줄 내로 설명해줘"라는 글과 사진 한 장은 둘 다 사용자가 한 번에 한 말입니다.
- 배열의 각 항목은 `type`(input_text, input_image, input_audio, input_file)으로 종류를 밝히고, 종류에 맞는 값을 담는다.
- 배열이라 순서가 있다. 그림 두 장을 넣고 "첫 번째와 두 번째의 차이"처럼 가리킬 수 있다.
- role/content 구조와 리스트로 대화를 잇는 방식은 1일차와 같다. 바뀐 것은 content 한 칸뿐이다.

### Q. 내 컴퓨터의 사진을 보낼 때 왜 Base64로 바꿔야 하고, 왜 하필 4/3배로 커지나요?
id: mmc-base64-why
type: 개념
level: 기본
related: mm-base64, mm-url-vs-base64

#### 풀이
요청은 **JSON, 즉 글자**로 보내는데 사진 파일은 아무 값이나 들어 있는 바이트라서 그대로 글자 사이에 넣을 수 없습니다. 그래서 바이트를 안전한 64개 글자(알파벳 대소문자, 숫자, `+`, `/`)로 다시 적습니다.
- 64 = 2⁶이라 글자 하나가 6비트를 나타낸다.
- 3바이트 = 24비트를 6비트씩 나누면 글자 4개. 그래서 3바이트마다 4글자, **4/3배**다.
- 압축이나 암호화가 아니라 표기만 바꾼 것이라 디코딩하면 원본과 한 바이트도 다르지 않다.

### Q. 다음 두 파일을 Base64로 바꾸면 각각 몇 글자가 될까요? (1) 블로그의 dog.jpg 163,759바이트 (2) 3,000,000바이트짜리 휴대폰 사진. 그리고 (1)을 `image_url`에 넣을 때 앞에 붙는 `"data:image/jpeg;base64,"`는 몇 글자인가요?
id: mmc-base64-calc
type: 계산
level: 기본
related: mm-base64, mm-url-vs-base64

#### 풀이
Base64 글자 수는 `⌈바이트 ÷ 3⌉ × 4`입니다.
- (1) 163,759 ÷ 3 = 54,586.3… → 올림 **54,587묶음** × 4 = **218,348글자**. 블로그의 실측값과 같다. 마지막 묶음이 1바이트뿐이라 `=` 두 개로 채운다(163,759 = 3 × 54,586 + 1).
- (2) 3,000,000 ÷ 3 = 1,000,000묶음 × 4 = **4,000,000글자**. 1,000,000글자, 약 33%가 늘었다.
- 접두어 `data:image/jpeg;base64,`는 `data:`(5) + `image/jpeg`(10) + `;base64,`(8) = **23글자**. 그래서 실제로 들어가는 문자열은 218,348 + 23 = 218,371글자다.

### Q. Diffusion 모델이 노이즈를 한 번에 없애지 않고, "섞인 노이즈 맞히기"를 배운 뒤 여러 단계에 걸쳐 조금씩 빼는 이유는 무엇인가요?
id: mmc-diffusion-steps
type: 개념
level: 심화
related: mm-diffusion, mm-text-to-image

#### 풀이
완전한 노이즈에서 한 번에 완성된 그림을 맞히는 것은 너무 어려운 문제지만, **"지금 상태에서 노이즈를 조금 덜어 내기"는 쉬운 문제**이기 때문입니다.
- 학습: 깨끗한 그림에 `x_t = √ᾱ_t · x_0 + √(1 − ᾱ_t) · ε`로 노이즈를 섞고, 모델이 ε를 맞히게 한다. 섞은 노이즈를 우리가 알고 있으니 정답이 있는 문제가 된다.
- t가 작으면 원본이 많이 남아 쉽고, t = 999면 원본 신호가 1% 정도라 거의 순수한 노이즈다. 모델은 모든 단계의 상황을 배운다.
- 생성: 순수한 노이즈에서 시작해 예측한 노이즈를 조금씩 빼면, 매 단계가 학습 때 본 상황과 비슷해진다.
- 프롬프트는 각 단계에서 "어느 방향으로 걷어 낼지"를 정한다. 대가는 단계 수만큼의 계산이다.

### Q. 잠재 공간을 쓰지 않고 1024×1024 픽셀 공간에서 바로 노이즈 제거를 반복한다면 어떻게 될까요?
id: mmc-no-latent
type: 만약에
level: 심화
related: mm-latent-space, mm-text-to-image

#### 풀이
- 한 단계에서 다듬을 숫자가 1024 × 1024 × 3 = **3,145,728개**다. 잠재 공간(128 × 128 × 4 = 65,536개)의 **48배**다.
- 이 단계를 수십 번 반복하니, 단계마다 48배 넘는 계산이 그대로 곱해진다. U-Net 안의 Attention처럼 크기의 제곱으로 늘어나는 연산이 있으면 차이는 더 커진다.
- 메모리도 그만큼 더 필요하다.

그림의 큰 구조(무엇이 어디에)는 작은 공간에서도 충분히 표현되고, 세부 픽셀 복원은 VAE Decoder가 한 번만 하면 됩니다. "작은 스케치북에서 먼저 그린다"는 선택이 생성 속도를 현실적으로 만든 것입니다. 대신 VAE를 거치며 작은 글자 같은 아주 세밀한 부분이 흐려질 수 있습니다.

### Q. "`images.generate`는 PNG 파일을 돌려주므로 응답을 그대로 파일에 쓰면 된다."
id: mmc-ox-generate-png
type: OX
level: 기본
related: mm-b64-json, mm-base64

#### 풀이
**X.** 그림은 `result.data[0].b64_json`에 **Base64 문자열**로 담겨 옵니다. `base64.b64decode()`로 바이트로 되돌린 뒤 `"wb"` 모드로 써야 PNG 파일이 됩니다. 디코딩 없이 쓰면 파일 안에 글자가 들어가 이미지 뷰어가 열지 못합니다. 반면 TTS는 `write_to_file`로 바로 mp3가 저장되니, API마다 결과 모양이 다르다는 점을 기억해야 합니다.

### Q. 코끼리 신사 그림을 호랑이로 바꿀 때 `images.edit` 대신 `images.generate`에 "라떼를 마시는 호랑이 신사"라고 새로 그리게 하면 결과가 어떻게 달라질까요?
id: mmc-regenerate-vs-edit
type: 만약에
level: 기본
related: mm-generate-vs-edit, mm-edit-exp, mm-text-to-image

#### 풀이
호랑이 신사는 나오겠지만, **정장의 색과 모양, 라떼 잔, 그림체가 원래 그림과 같다는 보장이 없습니다.**
- generate는 원본을 모르고, 매번 다른 무작위 노이즈에서 시작한다. 같은 프롬프트로도 매번 다른 그림이 나온다.
- edit는 원본 그림을 함께 받아 참고하기 때문에, 강의 결과처럼 캐릭터만 바뀌고 정장, 라떼 잔, 그림체가 남았다.
- 그래서 "이 그림에서 이것만" 바꾸려면 edit, 새로운 아이디어를 여러 개 보고 싶으면 generate가 맞다.

### Q. Gradio에서 Images API로 만든 그림을 출력 gr.Image에 띄울 때, Base64 문자열이 아니라 파일 이름을 돌려주는 이유는 무엇인가요? 입력 gr.Image의 `type`은 어떻게 고르나요?
id: mmc-gradio-filepath
type: 개념
level: 기본
related: mm-gradio-image, mm-b64-json, mm-gradio-image-editor

#### 풀이
gr.Image가 보여 줄 수 있는 것은 이미지 배열, PIL 이미지, 파일 경로 같은 **이미지 형태**이고, Base64 문자열은 그냥 긴 글자일 뿐이기 때문입니다. 그래서 디코딩해 파일로 저장하고 그 이름을 돌려주면 컴포넌트가 파일을 열어 보여 줍니다.

입력 쪽 `type`은 **함수 안에서 이미지로 무엇을 할지**로 고릅니다.
- 딥러닝 모델에 바로 넣는다 → numpy
- Pillow로 흑백, 반전 같은 처리를 한다 → pil
- 파일을 열어 API에 보내야 한다 → filepath
- ImageEditor는 값이 딕셔너리라, 손그림이 필요하면 `composite`를 꺼낸다.

### Q. 음성 모델이 소리를 스펙트로그램으로 바꿔 "그림처럼" 다루는 것은 ViT나 CNN이 이미지를 다루는 방식과 어떻게 이어지나요?
id: mmc-spectrogram-image
type: 연결
level: 심화
related: mm-sound-representation, vit-patch-embedding, cnn-image-as-tensor

#### 풀이
스펙트로그램은 가로가 시간, 세로가 주파수, 값이 세기인 **2차원 숫자 표**라서, 이미지가 픽셀마다 밝기가 적힌 표인 것과 같은 모양입니다. 그래서 이미지용 도구를 거의 그대로 쓸 수 있습니다.
- ViT처럼 작은 조각으로 잘라 벡터의 줄로 만들어 Transformer Encoder에 넣을 수 있다. 블로그 표현대로 "Vision처럼 조각내서 Encoder에 넣을 수 있게 된 셈"이다.
- CNN 관점에서도 모음의 가로 줄, 자음의 세로 띠는 작은 필터가 찾을 수 있는 지역 패턴이다.
- 차이도 있다. 이미지의 가로세로는 둘 다 공간이지만, 스펙트로그램은 축마다 의미가 다르다(시간과 주파수). 세로로 옮기면 소리의 높이가 바뀌어 다른 소리가 된다. 그래서 "어디로 옮겨도 같은 패턴"이라는 가정이 주파수 축에서는 덜 맞는다.
- 보충: Whisper는 log-Mel 스펙트로그램을 Encoder에 넣고 Decoder가 글을 쓰는 구조라, Image to Text의 Encoder-Decoder와 같은 모양이다.

### Q. TTS의 `voice`, `instructions`, `speed`는 각각 무엇을 바꾸나요? 영어회화 퀴즈의 모범 답안 발음에는 어떻게 쓸 수 있을까요?
id: mmc-tts-params
type: 개념
level: 기본
related: mm-tts, mm-voice-apps-exp

#### 풀이
- **voice**: 누구의 목소리인가. alloy, coral, nova 같은 정해진 목소리 중 고른다.
- **instructions**: 어떻게 말하는가. "차분한 뉴스 앵커처럼"처럼 말투, 감정, 억양을 글로 지시한다(최대 4096자).
- **speed**: 얼마나 빠르게. 0.25~4.0, 재생 시간은 기본 시간 ÷ speed다.

영어회화 퀴즈라면 같은 voice를 유지해 일관성을 주고, instructions로 "또박또박 발음하는 영어 선생님처럼"을 주고, speed를 0.75 정도로 늦추면 좋습니다. 60초 분량이면 60 ÷ 0.75 = 80초가 됩니다. 모델은 단가가 가장 낮은 gpt-4o-mini-tts(오디오 출력 1M 토큰당 $12)를 씁니다.

### Q. 1시간짜리 회의를 16kHz 모노 16비트 WAV로 녹음했습니다. 샘플 수, 파일 크기(MB = 1024 × 1024바이트), 25MB 한도로 나누면 몇 개 파일이 필요한지 구하세요. 같은 회의를 MP3 64kbps로 저장하면 몇 개인가요?
id: mmc-audio-size-calc
type: 계산
level: 심화
related: mm-audio-file-size, mm-sound-representation

#### 풀이
- 샘플 수: 16,000 × 3,600초 = **57,600,000개**
- 크기: 샘플 하나가 2바이트(16비트), 모노라 채널 1개 → 57,600,000 × 2 = 115,200,000바이트 ÷ 1,048,576 ≈ **109.9MB**
- 파일 수: 109.9 ÷ 25 ≈ 4.39 → **5개**. 한 파일에 약 13.7분(25 ÷ 1.83MB/분)씩 들어간다.
- MP3 64kbps: 64,000비트 ÷ 8 = 8,000바이트/초 × 3,600 = 28,800,000바이트 ≈ **27.5MB** → 25MB를 살짝 넘어 **2개**

참고로 CD 음질(44.1kHz 스테레오)이면 1분에 44,100 × 2 × 2 × 60 = 10,584,000바이트 ≈ 10.09MB라 1시간이 약 606MB, 25개로 나눠야 합니다. 인식에 필요한 만큼만 음질을 쓰는 것이 중요합니다.

### Q. 하루 1시간씩 20일 동안 회의를 받아 적는다면, gpt-4o-mini-transcribe와 gpt-4o-transcribe의 전사 비용은 각각 얼마인가요? 이 회의록을 LLM으로 영어 번역까지 한다면 비용 계산에 무엇이 더 들어가나요?
id: mmc-transcribe-cost
type: 계산
level: 기본
related: mm-transcription-exp, mm-transcription-llm, llm-token-cost

#### 풀이
총 길이는 60분 × 20일 = **1,200분**입니다.
- gpt-4o-mini-transcribe: 1,200 × $0.003 = **$3.6**
- gpt-4o-transcribe: 1,200 × 약 $0.006 = **약 $7.2** (두 배)
- 전사는 분 단위라 말이 빠르든 느리든 녹음 길이로만 정해진다.

번역을 붙이면 LLM 호출 비용이 따로 더해집니다. 이쪽은 1일차에서 본 대로 **토큰 단위**라, 받아 적은 글의 길이(입력 토큰), instructions의 길이, 번역문 길이(출력 토큰)에 단가를 곱해야 합니다. 한국어 입력은 같은 뜻의 영어보다 토큰을 더 쓴다는 점도 고려해야 합니다. 가격은 강의 자료 기준이며 바뀔 수 있습니다.

### Q. "TTS로 만든 음성을 다시 전사하면 원래 문장이 그대로 나온다."
id: mmc-ox-roundtrip
type: OX
level: 기본
related: mm-transcription-exp, mm-tts

#### 풀이
**X.** 블로그에서 "안녕하세요, 좋은 날씨예요!"를 TTS로 만든 뒤 전사하자 `'안녕하세요? 좋은 날씨예요?'`가 나왔습니다. 글자는 다 맞았지만 쉼표와 느낌표가 물음표로 바뀌었습니다. 문장부호는 소리에 직접 들어 있지 않고 억양으로 추측해야 하는 정보라서, 글 → 소리 → 글을 거치면 사라지거나 바뀔 수 있습니다. 전사 결과의 문장부호로 의문문을 판단하는 식의 로직은 위험합니다.

### Q. whisper-1에 한국어 음성을 넣고 영어 번역을 시켰더니 'Annyeonghaseyo. Oneuldo gungbuhagi…'처럼 로마자가 나왔습니다. 왜 이런 결과가 나왔을지 가능한 이유를 생각해 보세요.
id: mmc-whisper-romanize
type: 생각해보기
level: 심화
related: mm-whisper-translation-exp, mm-sound-representation

#### 풀이
모델 내부를 확인할 수는 없으니 가설로 생각해 봅니다.
1. **소리를 글자로 옮기는 습관이 이겼다**: 음성 모델의 기본 일은 들은 소리를 글자로 적는 것이다. 영어로 적으라는 조건이 약하게 작용하면, 뜻을 옮기는 대신 소리를 영어 알파벳으로 받아 적는 쪽(음역)이 더 쉬운 길이 된다.
2. **학습 데이터의 부족**: 한국어 음성과 영어 번역문이 짝지어진 데이터가 영어나 유럽 언어보다 적었다면, 번역보다 음역을 배우기 쉬웠을 것이다.
3. **받아쓰기 자체도 불안정하다**: "공부하기"가 "gungbuhagi"로 적혔다. 듣기 단계부터 흔들리면 뜻을 잡아 번역하기는 더 어렵다.

어느 쪽이든 결론은 같습니다. 한 모델에 "듣기 + 번역"을 함께 맡기면 어디서 틀렸는지 알 수 없습니다. 받아쓰기와 번역을 나누면 중간 결과(한국어 글)를 보고 문제를 찾을 수 있습니다.

### Q. 받아쓰기와 번역을 하나의 Translation 모델에 맡기지 않고, Transcription 모델과 LLM으로 나눠 맡기면 무엇이 좋아지나요?
id: mmc-split-roles
type: 개념
level: 기본
related: mm-transcription-llm, llm-instructions

#### 풀이
**각 모델이 가장 잘하는 일만 하게 되고, 번역 단계를 글로 조절할 수 있습니다.**
- 품질: whisper-1이 로마자로 적던 같은 음성이 `'Hello. It's a wonderfully pleasant day for studying today…'`로 자연스럽게 번역됐다. LLM은 이미 번역을 잘한다.
- 언어: Translation 모델은 영어로만 번역하지만(OpenAI 문서 기준), LLM은 instructions에 원하는 언어를 적으면 된다. 한국어로, 일본어로 번역하는 것도 같은 코드로 된다.
- 말투: "비즈니스 영어로", "초등학생도 이해할 수 있게", "고유명사는 원어 그대로"를 instructions에 넣는다. 1일차의 instructions가 그대로 쓰인다.
- 확인: 중간에 받아 적은 한국어가 남아 어디서 틀렸는지 추적할 수 있다.

### Q. 통역기 앱에서 전사 모델이 사람 이름 "김서준"을 "김서중"으로 잘못 받아 적었다면, 뒤의 LLM 번역과 TTS에서는 어떤 일이 생길까요?
id: mmc-error-propagation
type: 만약에
level: 심화
related: mm-transcription-llm, mm-cascade-vs-realtime, mm-voice-apps-exp

#### 풀이
- LLM은 받은 글만 보므로 "김서중"이 맞는 이름이라 믿고 **자연스럽게 번역**한다(Kim Seo-jung). 문장이 매끄러워서 오류가 오히려 감춰진다.
- TTS는 받은 번역문을 그대로 읽으니, 상대는 틀린 이름을 또렷한 목소리로 듣는다.
- 파이프라인의 약점인 **앞 단계 오류의 전파**다. 뒤 단계는 앞 단계 결과를 고칠 근거(원래 소리)를 갖고 있지 않다.

대응으로는 요구 조건에 있던 대로 **받아 적은 원문을 화면에 보여 줘서** 사용자가 확인하게 하고, 자주 나오는 고유명사 목록을 instructions에 넣어 LLM이 비슷한 이름을 고쳐 쓰게 하거나, 수정 후 다시 번역하는 버튼을 두는 방법이 있습니다.

### Q. 통역기를 Transcription → LLM → TTS 파이프라인으로 만드는 것과, 소리를 스트림으로 보내고 바로 음성으로 답을 받는 Realtime API로 만드는 것 중 무엇을 고르겠나요?
id: mmc-pipeline-vs-realtime
type: 생각해보기
level: 심화
related: mm-cascade-vs-realtime, mm-api-map, llm-stream

#### 풀이
쓰는 상황에 따라 다르고, 기준을 나눠 따져 보면 됩니다.
- **지연**: 파이프라인은 세 번의 호출을 차례로 기다린다. 마주 보고 대화하는 통역이라면 이 기다림이 크다. Realtime API는 스트림이라 짧다.
- **잃는 정보**: 파이프라인은 글로 바꾸는 순간 말투, 감정, 억양이 사라진다. 화난 말투도 TTS에서는 instructions가 정한 목소리로 나온다.
- **제어와 확인**: 파이프라인은 중간 글을 보여 주고 저장하고, 단계별로 모델과 instructions를 바꿀 수 있다. 오류가 어디서 났는지도 보인다.
- **비용**: 블로그 표에서 실시간 번역 모델은 2배에서 6배 비쌌다. 파이프라인은 각 단계에 가장 싼 모델을 고를 수 있다.

실습처럼 비용을 아끼고 결과를 확인하며 배우는 단계라면 파이프라인, 여행용 실시간 대화 통역처럼 속도와 자연스러움이 중요하면 Realtime 쪽이 맞습니다. 파이프라인의 지연은 LLM 답을 stream으로 받아 문장 단위로 TTS에 넘기는 식으로 줄일 수도 있습니다.

### Q. 이미지는 Base64 문자열로 오가는데, Gradio 마이크 녹음을 전사할 때는 파일을 `"rb"`로 열어 그대로 넘깁니다. 같은 바이너리인데 왜 다르게 보낼까요?
id: mmc-rb-vs-base64
type: 개념
level: 심화
related: mm-gradio-audio, mm-base64, mm-message-structure

#### 풀이
**어떤 형식의 요청에 싣느냐**가 다르기 때문입니다.
- Responses API의 이미지는 **JSON 메시지 안의 한 항목**이다. JSON은 글자만 담으니 Base64로 바꿔 넣는다. 메시지 구조 표의 `input_audio`도 같은 이유로 `"data": base64 문자열`이다.
- 전사 API는 `file=`로 **파일 자체를 받는** 형태다. 보충: 이런 업로드는 JSON이 아니라 파일 전송용 형식(multipart)으로 바이트를 그대로 보내서, 글자로 바꿀 필요도 4/3배로 커질 일도 없다. `images.edit`의 `image=open(..., "rb")`도 같은 방식이다.
- 그래서 Gradio에서는 `type="filepath"`로 녹음 파일 경로를 받아 `"rb"`로 열어 넘기면 끝이다.
- 반대 방향도 같다. 생성 이미지는 JSON 응답의 `b64_json`이라 디코딩이 필요하고, TTS는 오디오 바이트가 그대로 와서 `write_to_file`로 바로 저장된다.

### Q. 그림판 Artist에서 손그림을 바로 `images.edit`에 넣지 않고, Vision으로 글을 만든 뒤 Images API로 새로 그리게 했습니다. 두 방식은 결과가 어떻게 다를까요?
id: mmc-artist-design
type: 생각해보기
level: 심화
related: mm-artist-exp, mm-generate-vs-edit, mm-gradio-image-editor

#### 풀이
"원본을 얼마나 지킬 것인가"의 선택입니다.
- **edit에 바로 넣으면**: 원본을 참고하는 성질 때문에 손그림의 구도와 배치는 잘 지켜진다. 대신 서툰 선, 비뚤어진 비율도 남기 쉬워 "화가의 작품"이라기보다 "다듬은 손그림"에 가까울 수 있다.
- **Vision → 글 → generate**: 중간에 글만 남기 때문에 "언덕 위의 집과 해"처럼 **내용만** 전달된다. 서툰 선에서 완전히 벗어나 화풍이 살아난 그림이 나오지만, 원래 구도나 세부 배치는 달라질 수 있다.
- 글을 사이에 두는 장점이 하나 더 있다. 중간 프롬프트를 보여 주거나 "고흐 스타일로" 같은 화풍을 붙여 고칠 수 있고, 결과가 왜 그렇게 나왔는지 추적할 수 있다.
- 대가는 호출이 두 번(Vision + 생성)이라 시간과 비용이 더 든다는 점이다.

두 방식을 모두 만들어 Gallery에 나란히 띄워 비교하게 하는 것도 방법입니다.

### Q. 블로그는 "LLM을 중심에 두고 다른 모델을 입출력 장치처럼 붙이는 구조"가 많이 쓰이겠다고 정리했습니다. 이 구조에서 1일차에 배운 프롬프트 기술은 어디에 쓰이나요?
id: mmc-llm-hub
type: 연결
level: 기본
related: mm-transcription-llm, mm-voice-apps-exp, llm-instructions, llm-prompt-elements

#### 풀이
모든 모델 사이를 **글**이 오가고, 그 글을 다루는 곳이 LLM이라 프롬프트 기술이 거의 모든 연결 지점에 쓰입니다.
- **통역기**: 번역 말투와 대상 언어를 `instructions`로 정한다. Dropdown에서 고른 언어를 지시문에 끼워 넣는다.
- **영어회화 퀴즈**: 다섯 요소 중 **출력 형식**을 지정해 점수, 피드백, 모범 답안을 JSON으로 받으면 화면 컴포넌트에 나눠 뿌리기 쉽다.
- **그림판 Artist**: Vision에게 "화가에게 넘길 프롬프트를 써 줘"라고 시키는 것 자체가 프롬프트 설계다. 생성 모델이 읽을 글을 LLM이 쓰는 셈이다.
- **속담 퀴즈**: "속담의 장면을 글자 없이 그려줘"처럼 **제약**을 넣어야 정답이 그림에 글자로 박히지 않는다.

입출력 모델은 소리와 그림을 글로, 글을 소리와 그림으로 바꿔 주고, 무엇을 할지는 글로 된 지시가 정합니다.
