# 멀티모달
blog: multimodal_blog/llm_day2_blog.md
blog_title: [LLM 2일차] 보고, 그리고, 듣고, 말하는 AI: OpenAI 멀티모달 API

## Q. 멀티모달 AI란 무엇이고, Unimodal AI와 어떻게 다른가요?
id: mm-multimodal-definition
type: 개념
level: 기본
tags: Multimodal, Unimodal, 입력, 출력
source: 1. 멀티모달이 뭔가

### 30초 답변
Unimodal AI는 한 종류의 데이터를 받아 한 종류의 결과를 내는 모델로, 글을 넣으면 글이 나오는 GPT-3.5가 대표적입니다. Multimodal AI는 텍스트, 이미지, 영상, 음성처럼 여러 종류의 데이터를 입력으로 받거나 출력으로 냅니다. 다만 멀티모달 모델 하나가 모든 일을 하는 것은 아니고, 입력을 이해하는 일과 이미지나 오디오를 만들어 내는 일은 서로 다른 모델과 도구가 맡습니다. 그래서 입력과 출력의 형태를 먼저 정하고 거기에 맞는 API를 고르는 것이 출발점입니다.

### 자세히
| | Unimodal | Multimodal |
|---|---|---|
| 입력 | 한 종류 (예: 텍스트) | 여러 종류 (텍스트, 이미지, 음성, 영상) |
| 출력 | 한 종류 | 텍스트, 이미지, 음성 등 |
| 예 | GPT-3.5: 글 → 글 | 사진 + "이거 뭐야?" → 설명, 글 → 그림, 말 → 글 |

- 강의에서 강조한 점: "멀티모달 모델 하나가 다 해 준다"고 생각하면 헷갈린다.
- **이해**(그림을 보고 글로 답하기, 소리를 받아 적기)와 **생성**(그림 그리기, 목소리 만들기)은 맡는 모델이 다르다.
- 실제 앱은 이런 모델 여러 개를 이어 붙여 만든다. 예: 통역기 = 받아쓰기 → LLM 번역 → TTS

### 꼬리질문
#### 이미지를 입력으로 받는 모델과 이미지를 출력하는 모델은 왜 따로 있나요?
하는 일의 방향이 반대이기 때문입니다. 이미지를 이해하는 모델은 픽셀을 압축해 의미(글)로 바꾸고, 생성 모델은 짧은 글에서 수백만 개의 픽셀을 만들어 냅니다. 학습 방법도 다릅니다. 이해 쪽은 이미지와 설명을 짝지어 다음 단어를 맞히게 하고, 생성 쪽(Diffusion)은 노이즈를 걷어 내는 법을 배웁니다.

## Q. 하고 싶은 일이 정해졌을 때 OpenAI의 어떤 API를 써야 하는지 어떻게 고르나요?
id: mm-api-map
type: 개념
level: 기본
tags: Responses API, Images API, Audio API, Realtime API
source: 1. 멀티모달이 뭔가

### 30초 답변
무엇을 넣고 무엇을 받고 싶은지로 고릅니다. 그림을 보고 글로 답하게 하려면 Responses API의 `content`에 이미지를 같이 넣고, 그림을 받고 싶으면 Images API, 소리를 글로 바꾸거나 글을 소리로 바꾸려면 Audio API를 씁니다. 실시간으로 소리를 주고받는 대화는 Realtime API가 맡습니다.

### 자세히
| API | 입력 → 출력 | 하는 일 | 실습 모델 |
|---|---|---|---|
| Responses API | 텍스트 + 이미지 → 텍스트 | Vision. `content`에 글과 그림을 같이 넣음 | gpt-4o-mini |
| Images API | 텍스트 (+ 이미지) → 이미지 | generate(새로 그리기), edit(고치기) | gpt-image-1-mini |
| Audio API | 오디오 ↔ 텍스트 | STT(받아쓰기, 번역), TTS(읽어 주기) | gpt-4o-mini-transcribe, gpt-4o-mini-tts |
| Realtime API | 실시간 오디오 · 텍스트 | 소리를 스트림으로 보내고 바로 음성으로 답을 받음. WebSocket이나 WebRTC로 연결 | 소개만 함 |

- 블로그 정리: 결국 "무엇을 넣고 무엇을 받고 싶은가"에 따라 고르는 문제다.
- 자율 프로젝트에서는 비용과 형평성 때문에 gpt-4o-mini, gpt-image-1-mini, gpt-4o-mini-tts, gpt-4o-mini-transcribe 네 개만 쓰기로 했다.

### 꼬리질문
#### 이미지를 Responses API로도 만들 수 있나요?
Responses API에 이미지 생성 도구를 붙이는 방법도 있다고 소개만 들었고, Tool calling을 배울 때 다시 다룬다고 했습니다. 이번 실습에서는 Images API로 만들었습니다.

## Q. Image to Text 모델은 이미지를 어떻게 읽고 글로 답을 만드나요?
id: mm-image-to-text
type: 개념
level: 기본
tags: Vision, Image to Text, Vision Encoder, Text Decoder, Patch
source: 2. Vision › Image to Text가 돌아가는 방식

### 30초 답변
학습할 때 엄청난 양의 이미지와 설명을 짝지어 보여 줘서, 모양, 색, 위치 같은 시각적 특징과 언어의 관계를 미리 배워 둔 상태입니다. 새 이미지가 들어오면 Vision Encoder가 이미지를 작은 조각으로 잘라 벡터로 바꾸고, 조각들을 서로 비교해 무엇이 어디에 있는지 정리합니다. 그다음 Text Decoder가 질문과 함께 이 정보를 보면서 답을 한 단어씩 씁니다.

### 자세히
| 부분 | 하는 일 |
|---|---|
| Vision Encoder | 사진을 조각내 벡터로 바꾸고, 어떤 정보가 어디에 있는지 정리 |
| Text Decoder | 문장을 한 단어씩 쓰면서 그때그때 이미지의 필요한 부분을 다시 찾아봄 |

- 조각내는 방식은 ViT와 같다. 224×224를 16×16으로 자르면 가로 14 × 세로 14 = **196개** 조각
- 조각마다 숫자 벡터가 되어 한 줄로 늘어서고, Encoder가 서로 비교하며 "여기는 개, 여기는 자전거"를 정리한다.
- 강의의 쉬운 예: 바다, 산, 해, 하늘을 하나씩 찾고 → "하늘 중간에 해"처럼 관계를 붙이고 → "하늘 중간에 해가 떠 있고, 하늘 아래에는 세 봉우리의 산과 바다가 있다"를 완성
- 학습할 때는 이렇게 만든 문장을 정답 설명과 대조하면서 모델을 고친다.

### 꼬리질문
#### 이미지를 픽셀 하나하나가 아니라 조각 단위로 넣는 이유는 무엇인가요?
224×224 이미지의 픽셀은 5만 개가 넘어서 픽셀마다 토큰을 만들면 서로 비교하는 계산이 감당할 수 없이 커집니다. 16×16 조각으로 묶으면 196개로 줄고, 조각 하나에 선이나 질감 같은 지역 정보가 담겨 의미 있는 단위가 됩니다.

#### GPT-4o 같은 모델의 내부 구조도 정확히 이런가요?
상용 모델의 내부 구조는 공개돼 있지 않습니다. Encoder로 이미지를 정리하고 Decoder가 글을 쓴다는 흐름을 이해하는 용도로 보는 것이 맞습니다.

## Q. Text Decoder가 문장을 쓸 때 Cross-Attention은 어떤 역할을 하나요?
id: mm-cross-attention
type: 개념
level: 심화
tags: Cross-Attention, Text Decoder, Vision
source: 2. Vision › Image to Text가 돌아가는 방식

### 30초 답변
Cross-Attention은 단어를 하나 고를 때마다 이미지의 어느 조각을 더 자세히 볼지 정하는 장치입니다. 지금까지 쓴 문장이 질문(Query)이 되고, Encoder가 만든 이미지 조각 벡터들이 Key와 Value가 되어, 관련 있는 조각의 정보를 더 많이 가져옵니다. 그래서 "dog"을 쓸 때는 개가 있는 부분을, "truck"을 쓸 때는 트럭 쪽을 봅니다.

### 자세히
"a dog sitting next to a bicycle and a truck"을 쓰는 예 (블로그 그림은 손으로 칠한 예시 값이며 실제 모델 값이 아님):

| 단어 | 주로 보는 곳 |
|---|---|
| dog | 개가 있는 조각 |
| next to | 개와 자전거 양쪽 (관계를 나타내는 단어) |
| a truck | 뒤편 트럭 쪽 |
| a, and | 딱히 볼 곳이 없어 주목이 고르게 퍼짐 |

- Q는 텍스트 쪽, K와 V는 이미지 쪽에서 온다. 자연어처리의 Transformer Decoder가 Encoder 출력을 보던 Cross-Attention과 같은 구조다.
- Self-Attention이 한 시퀀스 안에서 서로를 보는 것이라면, Cross-Attention은 다른 시퀀스(이미지 조각)를 본다.

### 꼬리질문
#### Cross-Attention 지도를 "모델이 본 곳"이라고 그대로 해석해도 되나요?
참고는 되지만 그대로 믿기는 어렵습니다. 주목 가중치는 정보를 섞는 비율일 뿐이고, 여러 층과 head에 걸쳐 정보가 섞이기 때문에 한 층의 지도만으로 판단 근거를 단정할 수 없습니다. ViT의 [CLS] Attention 지도를 해석할 때와 같은 주의가 필요합니다.

## Q. 비전-언어 모델은 이미지 인코더를 LLM에 어떻게 연결하나요?
id: mm-vlm-connect
type: 비교
level: 심화
tags: VLM, LLaVA, Flamingo, BLIP-2, Projection
source: 보충 (블로그 밖 내용)

### 30초 답변
가장 단순한 방식은 LLaVA처럼 CLIP의 ViT로 이미지 조각 벡터를 만든 뒤, 작은 projection 층으로 LLM의 단어 임베딩과 같은 공간에 옮겨서 텍스트 토큰 앞에 이어 붙이는 것입니다. LLM은 구조를 바꾸지 않고 이 벡터들을 "이미지 토큰"으로 읽습니다. 다른 방식으로는 Flamingo처럼 LLM 층 사이에 Cross-Attention 층을 끼워 넣어 이미지를 보게 하거나, BLIP-2처럼 Q-Former로 이미지 정보를 소수의 query 벡터로 요약해 넘기는 방법이 있습니다.

### 자세히
| 방식 | 대표 모델 | 연결 방법 | 특징 |
|---|---|---|---|
| Projection | LLaVA | CLIP ViT 출력 → 선형층 또는 MLP → LLM 입력에 이어 붙임 | 단순, LLM 구조 그대로 |
| Cross-Attention 삽입 | Flamingo | 고정된 LLM 층 사이에 gated Cross-Attention 층 추가 | 블로그의 Decoder 설명과 비슷한 흐름 |
| Query 요약 | BLIP-2 | 학습되는 query 벡터들(Q-Former)이 이미지에서 필요한 정보를 뽑아 LLM에 전달 | 이미지 토큰 수를 고정된 소수로 줄임 |

- LLaVA 학습은 보통 두 단계다. 먼저 인코더와 LLM을 고정하고 projection만 이미지-캡션 쌍으로 학습해 두 공간을 맞추고, 다음에 지시 따르기 데이터로 projection과 LLM을 함께 미세조정한다.
- 블로그의 Encoder/Decoder 그림은 이런 구조들의 공통된 흐름을 보여 주는 설명이다. OpenAI 모델이 이 중 무엇인지는 공개돼 있지 않다.

### 꼬리질문
#### 이미지 인코더로 CLIP의 ViT를 자주 쓰는 이유는 무엇인가요?
CLIP은 이미지와 문장을 같은 공간에 맞추도록 대조 학습을 해서, 이미지 벡터가 이미 "언어로 설명할 수 있는 의미"를 담고 있습니다. 그래서 LLM의 단어 공간으로 옮기는 projection을 작게 둬도 잘 연결됩니다.

## Q. Vision 기능이 잘 못하는 일은 무엇인가요?
id: mm-vision-limits
type: 개념
level: 기본
tags: Vision, 한계, OCR, 의료 영상
source: 2. Vision › Vision이 못 하는 것

### 30초 답변
Vision은 모든 이미지를 완벽하게 읽지 못하고, 학습한 패턴을 바탕으로 가장 그럴듯한 답을 낼 뿐입니다. 강의에서 짚은 한계는 세 가지로, CT나 X-ray 같은 의학적 판독에는 맞지 않고, 한글 같은 비라틴 문자나 작은 글씨는 영어보다 인식 성능이 낮을 수 있으며, 기울어지거나 작거나 흐린 글씨는 처리하기 어렵습니다.

### 자세히
| 한계 | 내용 |
|---|---|
| 의학적 분석 | CT, X-ray 등 전문 판독에는 맞지 않다 |
| 영어가 아닌 글자 | 한글 같은 비라틴 문자, 작은 글씨는 성능이 낮을 수 있다 |
| 읽기 어려운 글씨 | 기울어지거나, 작거나, 흐린 텍스트 |

- 영수증이나 손글씨 메모를 읽는 앱을 생각한다면 이 부분부터 확인해야 한다.
- 근본 원인은 "그럴듯한 답"을 내는 생성 모델이라는 점이다. 확실하지 않아도 답을 만들어 낸다.

### 꼬리질문
#### 작은 글씨를 잘 못 읽는 이유는 무엇이라고 생각하나요?
보충: 이미지는 정해진 크기로 줄인 뒤 조각으로 잘라 읽기 때문에, 원본에서 작은 글씨는 줄이는 과정에서 몇 픽셀로 뭉개지고 조각 경계에 걸려 쪼개지기도 합니다. 글자가 들어 있는 부분을 잘라 크게 보내면 나아지는 경우가 많습니다.

## Q. 비전-언어 모델의 환각(hallucination)은 왜 생기고, 어떻게 줄일 수 있나요?
id: mm-vision-hallucination
type: 이유
level: 심화
tags: Hallucination, Object Hallucination, VLM, 신뢰성
source: 보충 (블로그 밖 내용)

### 30초 답변
비전-언어 모델의 환각은 이미지에 없는 물체나 글자를 있다고 말하는 현상입니다. Decoder가 언어 모델이라서, 이미지 정보가 흐릿하면 "식탁이면 포크도 있겠지"처럼 학습 데이터에서 자주 함께 나온 단어를 이어 쓰는 쪽으로 기울기 때문입니다. 줄이려면 해상도를 확보해 이미지 근거를 늘리고, 보이는 것만 답하고 모르면 모른다고 하라고 지시하며, 중요한 결과는 검출 모델이나 사람이 다시 확인하게 합니다.

### 자세히
- **원인 1, 언어 쪽 사전 지식**: 다음 토큰은 이미지와 지금까지 쓴 문장을 함께 보고 고른다. 이미지 근거가 약하면 문장 쪽 확률이 이긴다.
- **원인 2, 해상도와 압축**: 이미지를 줄이고 조각내는 과정에서 작은 물체나 글씨 정보가 사라진다.
- **원인 3, 학습 캡션**: 사람이 쓴 설명에는 사진에 보이지 않는 내용(배경 지식, 추측)도 섞여 있다.
- **줄이는 방법**: 확인하고 싶은 부분을 잘라 크게 보내기, "이미지에 보이는 것만, 확실하지 않으면 '불확실'이라고" 같은 지시, JSON처럼 정해진 형식으로 받아 코드로 검증하기, 중요한 판단은 사람 확인
- 물체 환각을 재는 벤치마크로 POPE처럼 "이미지에 X가 있나요?"를 예/아니오로 묻는 방식이 쓰인다.

### 꼬리질문
#### 텍스트 LLM의 환각과 무엇이 같고 무엇이 다른가요?
둘 다 그럴듯한 다음 토큰을 고르는 방식에서 나온다는 점은 같습니다. 다른 점은 비전 쪽은 확인할 근거(이미지)가 입력 안에 있는데도 그것을 덜 보고 언어 습관을 따른다는 것입니다. 그래서 이미지 근거를 늘리는 해상도 조절이나 부분 잘라 보내기가 따로 효과를 냅니다.

## Q. Responses API에 이미지를 함께 보낼 때 메시지는 어떤 구조인가요?
id: mm-message-structure
type: 개념
level: 기본
tags: Responses API, message, role, content, input_image
source: 2. Vision › 메시지 구조

### 30초 답변
`input`에 메시지 리스트를 넣고, 메시지는 누가 말했는지(`role`)와 무엇을 말했는지(`content`)로 이뤄집니다. 1일차와 다른 점은 `content`가 문자열 하나가 아니라 배열이 될 수 있다는 것으로, 이 배열 안에 `input_text`, `input_image`, `input_audio`, `input_file` 항목을 섞어 넣습니다. 그래서 글 질문과 사진을 한 메시지에 같이 보낼 수 있습니다.

### 자세히
| role | 설명 |
|---|---|
| system | 규칙이나 페르소나. 신형 API에서는 developer |
| user | 사람이 입력한 질문이나 요청 |
| assistant | AI의 응답 |
| tool | AI가 도구를 호출했을 때의 결과 |

| content type | 쓰는 법 |
|---|---|
| input_text | `"text": 문자열` |
| input_image | `"image_url": "https://..."` |
| input_audio | `"input_audio": {"data": base64 문자열, "format": "wav" 또는 "mp3"}` |
| input_file | `client.files.create(file=..., purpose="user_data")`로 올린 뒤 `"file_id": file.id` |

```python
response = client.responses.create(
    model="gpt-4o-mini",
    input=[{
        "role": "user",
        "content": [
            {"type": "input_text", "text": "이 그림을 3줄 내로 설명해줘."},
            {"type": "input_image", "image_url": "https://..."},
        ],
    }],
)
print(response.output_text)
```

- 강의 예제: 에펠탑 야경 사진 URL → "이 그림은 파리의 에펠탑을 담고 있습니다. 밤하늘을 배경으로 에펠탑이 밝혀져…"
- 1일차 코드에서 `content` 부분만 바뀌었고, 답은 똑같이 `output_text`로 꺼낸다.

### 꼬리질문
#### 이미지 여러 장을 비교하게 하려면 어떻게 하나요?
`content` 배열에 `input_image` 항목을 여러 개 넣고, `input_text`에 "첫 번째와 두 번째 그림의 차이를 말해줘"처럼 지시하면 됩니다. 배열 순서가 곧 모델이 읽는 순서라서 "첫 번째", "두 번째"로 가리킬 수 있습니다.

## Q. 이미지를 URL로 보내는 것과 Base64로 보내는 것은 어떻게 다르고, 언제 무엇을 쓰나요?
id: mm-url-vs-base64
type: 비교
level: 기본
tags: Base64, image_url, data URL, file_id
source: 2. Vision › 이미지를 URL 말고 직접 보내려면: Base64

### 30초 답변
인터넷에 올라간 그림이면 주소만 넘기면 되고, 서버가 그 주소에서 그림을 가져갑니다. 내 컴퓨터에 있는 사진은 주소가 없으니, 파일의 바이트를 Base64 글자로 바꿔 `"data:image/jpeg;base64,"` 뒤에 붙여 같은 `image_url` 자리에 넣습니다. URL은 요청이 가볍지만 공개된 주소가 필요하고, Base64는 어떤 파일이든 보낼 수 있지만 요청이 원본의 4/3배만큼 커집니다.

### 자세히
| | URL | Base64 (data URL) |
|---|---|---|
| 넣는 값 | `"https://..."` | `"data:image/jpeg;base64,/9j/4AAQ..."` |
| 그림이 있는 곳 | 인터넷 | 요청 본문 안 |
| 요청 크기 | 작음 | 원본 × 약 4/3 |
| 쓰는 경우 | 공개된 이미지 | 로컬 파일, 업로드받은 사진, 비공개 이미지 |

- 메시지 구조 표에는 파일을 먼저 올리고 `file_id`로 가리키는 `input_file` 방식도 있다(블로그 예는 PDF).
- 보충: URL 방식은 서버가 그 주소에 접근할 수 있어야 한다. 로그인이 필요한 주소나 사내망 주소는 가져갈 수 없다.

### 꼬리질문
#### 그림판 Artist 앱에서는 어느 쪽을 썼나요?
Gradio ImageEditor로 받은 손그림은 주소가 없는 이미지라서, `composite`를 Base64로 바꿔 Vision에 넣는 구조로 짰습니다.

## Q. Base64란 무엇이고, 왜 원본보다 약 4/3배 커지나요?
id: mm-base64
type: 개념
level: 심화
tags: Base64, 인코딩, JSON, 파일 크기
source: 2. Vision › 이미지를 URL 말고 직접 보내려면: Base64

### 30초 답변
Base64는 파일의 바이트를 알파벳 대소문자, 숫자, `+`, `/`의 64개 글자만으로 바꿔 적는 방법입니다. JSON 요청은 글자만 담을 수 있어서 그림 같은 바이너리를 실으려면 글자로 바꿔야 합니다. 3바이트(24비트)를 6비트씩 네 글자로 나눠 적기 때문에 크기가 4/3배가 되고, 실습용 dog.jpg 163,759바이트는 218,348글자가 됐습니다.

### 자세히
- 64 = 2⁶이라 글자 하나가 6비트를 나타낸다. 3바이트 = 24비트 = 6비트 × 4 → 글자 4개
- 실측: dog.jpg 163,759바이트 → Base64 218,348글자 (×1.333). 다시 디코딩하면 원본과 한 바이트도 다르지 않다.
- 계산 확인: 163,759 ÷ 3 = 54,586.3 → 올림 54,587묶음 × 4 = 218,348글자. 끝이 3바이트로 안 떨어지면 `=`로 채운다.
- **압축이 아니라 표기 방식**이라 정보는 그대로이고 크기만 늘어난다.
- 반대 방향에서도 나온다. Images API가 생성한 그림을 Base64 문자열(`b64_json`)로 돌려준다.

### 꼬리질문
#### Base64는 암호화인가요?
아닙니다. 누구나 같은 규칙으로 바로 되돌릴 수 있는 표기 방식일 뿐이라 보안 효과가 없습니다. 민감한 이미지를 보낼 때 보호해 주는 것은 HTTPS 같은 전송 암호화입니다.

#### 크기가 커지는 게 실제로 문제가 되나요?
요청 크기 한도와 전송 시간에 영향을 줍니다. 큰 사진을 그대로 Base64로 보내면 요청이 원본보다 33% 무거워지니, 필요한 해상도로 줄이거나 JPEG로 압축한 뒤 보내는 편이 낫습니다.

## Q. 이미지 해상도는 Vision 모델의 비용과 정확도에 어떤 영향을 주나요?
id: mm-image-tokens
type: 이유
level: 심화
tags: 이미지 토큰, 해상도, detail, 비용
source: 보충 (블로그 밖 내용)

### 30초 답변
이미지는 조각 단위로 토큰이 되기 때문에 해상도가 높을수록 조각 수, 곧 입력 토큰이 늘어 비용과 응답 시간이 커집니다. 같은 16×16 조각이라도 224×224면 196개인데 448×448이면 784개로 네 배가 됩니다. 대신 해상도가 높으면 작은 글씨나 작은 물체를 더 잘 읽으니, 작업에 필요한 만큼만 해상도를 쓰는 것이 핵심입니다.

### 자세히
| 입력 크기 (16×16 조각) | 조각 수 | 조각끼리 비교하는 쌍 |
|---|---|---|
| 224×224 | 14 × 14 = 196 | 196² ≈ 3.8만 |
| 448×448 | 28 × 28 = 784 (4배) | 784² ≈ 61만 (16배) |

- 가로세로를 두 배로 하면 조각 수는 4배, Self-Attention 계산은 조각 수의 제곱이라 16배가 된다.
- OpenAI API에는 이미지 항목에 `detail` 옵션(low, high, auto 등)이 있어 이미지를 얼마나 자세히 처리할지 고를 수 있다. low는 작게 줄여 적은 토큰으로, high는 더 큰 해상도로 처리한다.
- 이미지 한 장이 몇 토큰인지는 모델마다 계산 방식이 달라서, 문서의 모델별 표와 응답의 사용량(usage)으로 확인해야 한다.
- 실무 감각: 사진 분위기 설명은 낮은 해상도로 충분하고, 영수증이나 표 읽기는 높은 해상도나 부분 잘라 보내기가 필요하다.

### 꼬리질문
#### 큰 이미지의 일부만 자세히 보고 싶으면 어떻게 하나요?
전체를 고해상도로 보내기보다 필요한 영역만 잘라서 보내는 편이 싸고 정확합니다. 전체 그림을 저해상도로 한 번 보내 위치를 파악하고, 그 부분만 잘라 다시 묻는 두 단계도 쓸 수 있습니다.

## Q. Images API의 generate와 edit는 어떻게 다른가요?
id: mm-generate-vs-edit
type: 비교
level: 기본
tags: Images API, generate, edit
source: 3. Images API › 코드: 생성, 코드: 편집

### 30초 답변
generate는 텍스트 프롬프트만으로 새 그림을 만들고, edit는 주어진 그림을 프롬프트에 따라 완전히 새로 그리거나 일부분만 고칩니다. edit에는 원본 그림을 `"rb"`로 열어 같이 넘기고, 모델이 원본을 참고하기 때문에 바꾸라고 한 부분 외의 요소가 유지됩니다. 실습에서 코끼리를 호랑이로 바꿨을 때 정장, 라떼 잔, 그림체가 그대로 남았습니다.

### 자세히
| | generate | edit |
|---|---|---|
| 입력 | prompt | image + prompt |
| 원본 | 없음 | `image=open("elephant.png", "rb")` |
| 결과 | 새 그림 | 원본을 참고해 고친 그림 |
| 반환 | `result.data[0].b64_json` (Base64) | 같음 |

```python
result = client.images.generate(model="gpt-image-1-mini",
                                prompt="라떼를 마시는 귀여운 코끼리 신사를 그려줘",
                                size="1024x1024")
result = client.images.edit(model="gpt-image-1-mini",
                            image=open("elephant.png", "rb"),
                            prompt="캐릭터를 코끼리에서 호랑이로 바꿔줘")
```

### 꼬리질문
#### 원본 파일을 왜 "rb"로 여나요?
그림 파일을 글자로 해석하지 않고 바이트 그대로 서버에 보내야 하기 때문입니다. 텍스트 모드로 열면 바이트를 문자로 해석하려다 깨지거나 에러가 납니다.

## Q. Text to Image 모델이 글에서 그림을 만드는 과정을 설명해 보세요.
id: mm-text-to-image
type: 개념
level: 심화
tags: Text to Image, Text Encoder, U-Net, VAE, Diffusion
source: 3. Images API › Text to Image가 돌아가는 방식

### 30초 답변
먼저 Text Encoder가 프롬프트를 숫자 벡터로 바꿉니다. 그다음 아무 의미 없는 노이즈에서 시작해, U-Net이 프롬프트 벡터를 참고하면서 노이즈를 조금씩 걷어 내는 일을 여러 번 반복합니다. 이 작업은 원본 크기가 아니라 작게 압축한 잠재 공간에서 일어나고, 마지막에 VAE Decoder가 그 결과를 실제 크기의 그림으로 펼칩니다.

### 자세히
1. **Text Encoder**: 프롬프트 → 숫자 벡터
2. **노이즈**에서 시작
3. **U-Net**: 프롬프트를 참고하며 노이즈를 조금 걷어 냄. 여러 번 반복
4. 이 과정은 작게 압축한 **잠재 공간(latent space)**에서 진행
5. **VAE Decoder**: 잠재 공간 결과 → 실제 크기 그림

- 강의 자료의 구분이고 Stable Diffusion 계열의 구조다. OpenAI의 gpt-image 모델 내부는 공개돼 있지 않으니 흐름을 이해하는 용도다.
- 보충: U-Net이 프롬프트를 참고하는 방법도 Cross-Attention이다. 이미지 쪽이 Query, 텍스트 벡터가 Key·Value가 되어 Image to Text와 방향이 반대다.

### 꼬리질문
#### 같은 프롬프트로 다시 생성하면 같은 그림이 나오나요?
보통 다르게 나옵니다. 시작 노이즈가 무작위라서 같은 프롬프트라도 출발점이 달라지고, 걷어 낸 결과도 달라집니다. 원본을 유지한 채 일부만 바꾸고 싶으면 edit를 써야 합니다.

## Q. Diffusion 모델은 어떻게 학습하고, 어떻게 노이즈에서 그림을 만들어 내나요?
id: mm-diffusion
type: 개념
level: 심화
tags: Diffusion, DDPM, 노이즈 예측
source: 3. Images API › Text to Image가 돌아가는 방식

### 30초 답변
학습할 때는 깨끗한 그림에 노이즈를 단계적으로 섞어 가면서, 모델에게 "여기에 섞인 노이즈가 무엇이었는지" 맞히게 합니다. 이 능력이 생기면 생성할 때는 과정을 뒤집어, 완전한 노이즈에서 시작해 모델이 예측한 노이즈를 한 단계씩 빼 나가며 그림을 만듭니다. 한 번에 그림을 만드는 대신 작은 수정을 여러 번 반복하는 것이 핵심입니다.

### 자세히
- 노이즈 섞기(DDPM 방식): `x_t = √ᾱ_t · x_0 + √(1 − ᾱ_t) · ε`. t가 클수록 원본 비중 √ᾱ_t가 줄어든다.
- 블로그 그림(개념 설명용): t = 300쯤이면 개와 자전거의 윤곽만 겨우 남고, t = 999에서는 원본 신호가 1% 정도라 그냥 노이즈다.
- 학습: 임의의 t를 골라 노이즈를 섞고, 모델이 그 노이즈 ε를 맞히도록 한다.
- 생성: 완전한 노이즈 → 예측한 노이즈를 조금 뺌 → 반복 → 그림. 프롬프트는 "어느 방향으로 걷어 낼지"를 정한다.
- 보충: 프롬프트를 얼마나 강하게 따를지는 classifier-free guidance로 조절한다. 프롬프트를 준 예측과 주지 않은 예측의 차이를 키워서 프롬프트 쪽으로 더 끌고 간다.

### 꼬리질문
#### 왜 한 번에 그림을 만들지 않고 여러 단계로 나누나요?
노이즈에서 바로 완성된 그림을 맞히는 것은 너무 어려운 문제지만, "지금 상태에서 노이즈를 조금 덜어 내는 것"은 훨씬 쉬운 문제입니다. 쉬운 문제를 여러 번 푸는 방식이라 학습이 안정적이고 결과 품질이 좋습니다. 대가는 단계 수만큼 계산이 늘어 생성이 느리다는 점입니다.

## Q. Diffusion이 노이즈 제거를 픽셀 공간이 아니라 잠재 공간에서 하는 이유는 무엇인가요?
id: mm-latent-space
type: 이유
level: 심화
tags: Latent Space, VAE, Stable Diffusion, 계산량
source: 3. Images API › Text to Image가 돌아가는 방식

### 30초 답변
1024×1024 그림을 픽셀 그대로 수십 번 다듬으면 계산량이 엄청나기 때문입니다. Stable Diffusion은 가로세로를 8분의 1로 줄이고 채널을 4개로 둔 잠재 공간에서 작업해서, 다뤄야 할 숫자가 48배 줄어듭니다. 1024×1024 그림이면 약 315만 개 대신 약 6만 5천 개만 다듬고, 다 끝난 뒤 VAE Decoder가 큰 그림으로 펼칩니다.

### 자세히
| 크기 | 픽셀 공간 H×W×3 | 잠재 공간 H/8×W/8×4 |
|---|---|---|
| 1024×1024 | 3,145,728 | 128×128×4 = 65,536 |
| 비율 | | 8 × 8 × 3 ÷ 4 = **48배** 적음 |

- 비유: 큰 캔버스에 바로 그리지 않고 작은 스케치북에서 먼저 그린다.
- 잠재 공간은 VAE가 학습한 압축 표현이라, 사람이 보기엔 의미 없는 숫자지만 그림의 구조와 내용을 담고 있다.
- OpenAI gpt-image 모델의 내부 구조는 공개돼 있지 않아 Stable Diffusion 설정으로 계산한 값이다.

### 꼬리질문
#### 잠재 공간에서 작업하면 잃는 것은 없나요?
VAE가 압축하고 펼치는 과정에서 아주 세밀한 부분(작은 글자, 손가락 같은 세부)이 흐려지거나 틀어질 수 있습니다. 계산을 크게 아끼는 대신 세부 표현의 일부를 Decoder의 복원 능력에 맡기는 트레이드오프입니다.

## Q. 이미지를 생성할 때 지켜야 할 규칙과 모델 선택 기준은 무엇인가요?
id: mm-image-rules-model
type: 개념
level: 기본
tags: Images API, 규칙, 가격, gpt-image-1-mini
source: 3. Images API › 이미지 생성 규칙과 모델 고르기

### 30초 답변
이미지 타입은 PNG, JPEG, WEBP, GIF이고 크기 제한은 50MB이며, 저작권 침해나 성적인 요소가 없어야 합니다. 모델은 가격 차이가 커서, 이미지 출력 1M 토큰당 gpt-image-2가 30달러, gpt-image-1.5가 32달러인데 gpt-image-1-mini는 8달러입니다. 실습에서는 반드시 gpt-image-1-mini를 쓰라고 했는데, 속담 퀴즈처럼 문제마다 그림을 새로 그리는 앱은 호출이 금방 쌓이기 때문입니다.

### 자세히
| 규칙 | 내용 |
|---|---|
| 이미지 타입 | PNG, JPEG, WEBP, GIF |
| 크기 제한 | 50MB |
| 내용 | 저작권 침해, 성적인 요소 없어야 함 |

| 모델 | 이미지 출력 1M 토큰당 (강의 자료, 바뀔 수 있음) |
|---|---|
| gpt-image-2 | $30 |
| gpt-image-1.5 | $32 |
| gpt-image-1-mini | $8 |

- gpt-image-1-mini는 다른 두 모델의 약 1/4 가격이다.
- 모델 선택은 품질만이 아니라 호출 횟수 × 단가로 본다. 사용자마다, 문제마다 그림을 만드는 앱은 단가가 곧 운영비다.

### 꼬리질문
#### 같은 그림을 여러 사용자가 반복해서 요청한다면 비용을 어떻게 줄이나요?
생성한 그림을 파일로 저장해 두고 같은 프롬프트에는 다시 생성하지 않고 저장본을 돌려주는 캐시를 둘 수 있습니다. 속담 퀴즈라면 속담 100개의 그림을 미리 한 번씩 만들어 두는 방법도 있습니다. 대신 매번 새로운 그림이 나오는 재미는 줄어듭니다.

## Q. images.generate 결과를 실제 이미지 파일로 저장할 때 무엇을 주의해야 했나요?
id: mm-b64-json
type: 경험
level: 기본
tags: b64_json, base64.b64decode, wb, 파일 저장
source: 3. Images API › 코드: 생성

### 30초 답변
처음에 당황했던 부분은 `images.generate`가 PNG 파일을 바로 주지 않는다는 점이었습니다. 그림이 `result.data[0].b64_json`에 Base64 문자열로 담겨 와서, `base64.b64decode()`로 바이트로 되돌린 뒤 `"wb"` 모드로 열어 파일에 써야 비로소 그림 파일이 됩니다. 반대로 TTS는 `write_to_file`로 바로 mp3가 저장돼서 API마다 결과 형태가 다르다는 걸 알게 됐습니다.

### 자세히
```python
import base64

result = client.images.generate(model="gpt-image-1-mini",
                                prompt="라떼를 마시는 귀여운 코끼리 신사를 그려줘",
                                size="1024x1024")
image_base64 = result.data[0].b64_json
with open("elephant.png", "wb") as f:
    f.write(base64.b64decode(image_base64))
```

- `"wb"`(write binary): 디코딩한 것은 글자가 아니라 바이트라서 바이너리 모드로 써야 한다.
- 강의 예제 결과: 정장을 입고 라떼를 마시는 코끼리
- 블로그 정리: 이미지는 Base64 문자열로 오가고, 받으면 디코딩해서 `"wb"`로 쓴다. 오디오는 `"rb"`로 열어 바이트 그대로 보낸다.

### 꼬리질문
#### 디코딩 없이 Base64 문자열을 그대로 .png로 저장하면 어떻게 되나요?
파일 안에 PNG 바이트가 아니라 글자들이 들어가서 이미지 뷰어가 열지 못합니다. 크기도 원래보다 약 4/3배 큽니다.

## Q. 이미지 편집 API를 실제로 써 보니 결과가 어땠나요?
id: mm-edit-exp
type: 경험
level: 기본
tags: images.edit, 편집, 원본 유지
source: 3. Images API › 코드: 편집

### 30초 답변
앞에서 만든 라떼 마시는 코끼리 신사 그림을 `images.edit`에 넣고 "캐릭터를 코끼리에서 호랑이로 바꿔줘"라고 했습니다. 코끼리가 호랑이로 바뀌었는데 정장, 라떼 잔, 그림체는 그대로 남아 있었습니다. 처음부터 다시 그리는 generate와 달리 원본을 참고한다는 게 결과물에서 바로 보였습니다. 이 결과는 강의 자료의 실행 결과를 옮긴 것입니다.

### 자세히
```python
result = client.images.edit(model="gpt-image-1-mini",
                            image=open("elephant.png", "rb"),
                            prompt="캐릭터를 코끼리에서 호랑이로 바꿔줘")
with open("elephant.png", "wb") as f:
    f.write(base64.b64decode(result.data[0].b64_json))
```

| 바뀐 것 | 유지된 것 |
|---|---|
| 캐릭터 (코끼리 → 호랑이) | 정장, 라떼 잔, 그림체 |

- 같은 파일 이름으로 저장해서 원본을 덮어썼다. 원본을 남기려면 다른 이름으로 저장해야 한다.

### 꼬리질문
#### 편집이 원본을 유지할 수 있는 이유는 무엇이라고 생각하나요?
보충: 원본 이미지가 생성의 조건으로 함께 들어가기 때문입니다. 완전한 노이즈에서 새로 시작하는 대신 원본의 구조와 스타일 정보를 참고해서 그리니, 프롬프트가 바꾸라고 한 부분 외에는 원본과 비슷하게 나옵니다. 다만 모델 내부 방식이 공개되지 않아 구체적인 메커니즘은 단정하기 어렵습니다.

## Q. Gradio의 gr.Image와 gr.Gallery는 무엇이고, type 파라미터는 왜 중요한가요?
id: mm-gradio-image
type: 개념
level: 기본
tags: Gradio, gr.Image, gr.Gallery, type
source: 4. Gradio로 이미지 다루기

### 30초 답변
gr.Image는 이미지를 넣고 보여 주는 컴포넌트이고, `type`이 함수가 이미지를 어떤 형태로 받을지 정합니다. 딥러닝 모델에 바로 넣을 때는 numpy, Pillow로 처리할 때는 pil, 파일 경로가 필요할 때는 filepath를 고릅니다. gr.Gallery는 여러 장을 한 번에 보여 주는 컴포넌트로, 함수가 (이미지, 캡션) 튜플의 리스트를 돌려주면 캡션까지 달아 줍니다.

### 자세히
| type | 함수가 받는 값 | 언제 쓰나 |
|---|---|---|
| numpy | 배열 | 딥러닝 모델에 바로 넣을 때 |
| pil | PIL Image | Pillow로 처리할 때 |
| filepath | 파일 경로 문자열 | 파일 경로가 필요할 때 (예: 파일을 열어 API에 보낼 때) |

- gr.Image: `label`, `height`, `width`로 이름과 크기
- gr.Gallery: `columns`(한 줄에 몇 장), `rows`(몇 줄)
- 실험: API 대신 Pillow로 흑백, 좌우 반전, 포스터 효과를 주는 함수를 붙여 원본과 함께 Gallery에 띄웠다. 실제 앱에서는 이 함수 안에서 Images API를 부르면 된다.

### 꼬리질문
#### Images API로 생성한 그림은 어떻게 출력 컴포넌트에 띄우나요?
API가 돌려준 건 Base64 문자열이라 컴포넌트에 바로 넣을 수 없습니다. 디코딩해서 파일로 저장한 다음 그 파일 이름을 함수에서 돌려주면, 출력 쪽 gr.Image가 파일을 열어 보여 줍니다. 강의 예제에서는 곰이 사람에게 덤벼드는 흑백 그림이 떴습니다.

## Q. gr.ImageEditor의 값에는 무엇이 들어 있고, 손그림을 받을 때 무엇을 꺼내 쓰나요?
id: mm-gradio-image-editor
type: 개념
level: 기본
tags: Gradio, gr.ImageEditor, composite, layers
source: 4. Gradio로 이미지 다루기

### 30초 답변
gr.ImageEditor는 올린 그림 위에 직접 그림을 그릴 수 있는 컴포넌트이고, 값은 딕셔너리 하나에 background, layers, composite 세 가지가 들어 있습니다. background는 원본, layers는 사용자가 위에 그린 레이어들, composite는 둘을 합친 결과입니다. 대부분은 composite만 꺼내 쓰고, 원본과 덧그린 부분을 따로 처리하고 싶을 때 background와 layers를 씁니다.

### 자세히
| 키 | 내용 |
|---|---|
| background | 원본 이미지 |
| layers | 사용자가 위에 그린 레이어들 |
| composite | background + layers를 합친 결과 |

```python
def show_layers(edit):
    return edit["composite"] if edit else None

editor.change(show_layers, editor, comp)
```

- 실험: 왼쪽 에디터에서 붓으로 선 두 개를 그었더니 오른쪽에 composite가 바로 떴다.
- 그림판 Artist에서 손그림 입력으로 쓴다.

### 꼬리질문
#### layers를 따로 쓰면 어떤 기능을 만들 수 있나요?
사용자가 칠한 영역만 알 수 있으니 "칠한 부분만 바꿔 줘" 같은 기능을 만들 수 있습니다. 보충: 이미지 편집에서 바꿀 영역을 알려 주는 마스크를 사용자가 직접 칠하게 하는 식입니다.

## Q. Audio API의 TTS, Transcription, Translation은 각각 무엇을 하나요?
id: mm-audio-api
type: 비교
level: 기본
tags: Audio API, TTS, STT, Transcription, Translation
source: 5. Audio API

### 30초 답변
Audio API는 방향에 따라 세 가지로 나뉩니다. TTS는 글을 소리로 바꾸고, STT 중 Transcription은 소리를 같은 언어의 글로 받아 적고, Translation은 소리를 영어 글로 번역합니다. 실습에서 TTS는 gpt-4o-mini-tts, Transcription은 gpt-4o-mini-transcribe, Translation은 whisper-1을 썼습니다.

### 자세히
| 기능 | 방향 | 메서드 | 실습 모델 | 가격 (강의 자료) |
|---|---|---|---|---|
| TTS | 글 → 소리 | `client.audio.speech.create` | gpt-4o-mini-tts | 오디오 출력 1M 토큰당 $12 |
| Transcription | 소리 → 같은 언어 글 | `client.audio.transcriptions.create` | gpt-4o-mini-transcribe | 분당 $0.003 |
| Translation | 소리 → 영어 글 | `client.audio.translations.create` | whisper-1 | |

- 결과 형태도 다르다. TTS는 `write_to_file`로 바로 mp3, STT는 `.text`로 문자열

### 꼬리질문
#### 전사 모델은 왜 토큰이 아니라 분 단위로 가격을 매기나요?
보충: 사용자가 예측하기 쉬운 단위가 소리의 길이이기 때문이라고 생각합니다. 같은 1분이라도 말의 빠르기에 따라 글자 수는 달라지지만, 녹음 길이는 업로드 전에 알 수 있습니다. 강의 표에서도 전사는 분당 추정 비용으로 정리돼 있었습니다.

## Q. 컴퓨터는 소리를 어떤 데이터로 다루고, 음성 모델은 왜 스펙트로그램을 쓰나요?
id: mm-sound-representation
type: 개념
level: 심화
tags: 파형, 샘플링, 스펙트로그램, 음성 인식
source: 5. Audio API › 컴퓨터가 소리를 보는 법

### 30초 답변
마이크는 공기의 떨림을 일정한 간격으로 재서 숫자로 기록하고, 16kHz로 녹음하면 1초에 숫자가 16,000개 나옵니다. 이 파형만으로는 무슨 소리인지 알아보기 어려워서, 음성 모델은 보통 가로가 시간, 세로가 주파수, 밝기가 세기인 스펙트로그램으로 바꿔서 봅니다. 스펙트로그램에서는 모음마다 밝은 줄의 높이가 달라 "아"와 "이"를 구분할 수 있고, 소리가 그림이 되니 Vision처럼 조각내서 Encoder에 넣을 수 있습니다.

### 자세히
| 표현 | 축 | 보이는 것 |
|---|---|---|
| 파형 | 시간 × 진폭 | 마이크가 기록한 원래 모양. 16kHz면 1초에 16,000개 |
| 스펙트로그램 | 시간 × 주파수, 밝기 = 세기 | 자음 같은 잡음은 세로로 길쭉한 띠, 모음은 가로로 몇 줄 밝은 띠 |

- 블로그 실험: 배음과 포먼트를 흉내 내 모음 세 개와 자음 같은 잡음을 합성해 그렸다(실제 녹음 아님).
- 스펙트로그램은 짧은 구간마다 어떤 주파수가 얼마나 센지 계산해 이어 붙인 것이다.
- 보충: Whisper는 사람의 청각에 맞춰 주파수 축을 바꾼 log-Mel 스펙트로그램을 입력으로 받는 Transformer Encoder-Decoder 모델이다.

### 꼬리질문
#### 음성 인식에 16kHz면 충분한 이유는 무엇인가요?
보충: 샘플링 주파수의 절반까지의 주파수를 기록할 수 있어서 16kHz면 8kHz까지 담깁니다. 말소리를 알아듣는 데 중요한 정보는 대부분 그 아래에 있어서, 음악용 44.1kHz보다 훨씬 작은 파일로도 인식에는 충분합니다.

## Q. TTS API의 주요 파라미터와 쓰임을 설명해 보세요.
id: mm-tts
type: 개념
level: 기본
tags: TTS, voice, instructions, speed, gpt-4o-mini-tts
source: 5. Audio API › TTS: 글을 소리로

### 30초 답변
`client.audio.speech.create`에 모델, 목소리(`voice`), 읽을 글(`input`)을 넣으면 결과를 바로 mp3 파일로 저장할 수 있습니다. `instructions`로 "차분한 뉴스 앵커처럼"처럼 말투와 감정을 지시할 수 있고, `speed`는 0.25에서 4.0까지라 기본 속도로 1분 걸리는 글이 4분에서 15초까지 바뀝니다. 오디오 출력 1M 토큰당 gpt-audio가 64달러인데 gpt-4o-mini-tts는 12달러라 실습에서는 이것만 썼습니다.

### 자세히
```python
speech = client.audio.speech.create(model="gpt-4o-mini-tts", voice="alloy",
                                    input="안녕하세요, 좋은 날씨예요!")
speech.write_to_file("speech.mp3")
```

| 파라미터 | 옵션 | 설명 |
|---|---|---|
| model | tts-1, tts-1-hd, gpt-4o-mini-tts | 모델 |
| input | 최대 4096자 | 소리로 바꿀 텍스트 |
| voice | alloy, ash, ballad, coral, echo, fable, onyx, nova, sage, shimmer, verse, marin, cedar 등 | 목소리 |
| instructions | 최대 4096자 | 말투, 감정, 억양 지시 |
| response_format | mp3, opus, aac, flac, wav, pcm | 출력 포맷 |
| speed | 0.25 ~ 4.0, 기본 1.0 | 재생 속도 |
| stream_format | audio, sse | 스트리밍 방식 |

- 재생 시간 = 기본 시간 ÷ speed. 60초 글이 0.25면 240초(4분), 4.0이면 15초
- 이미지와 달리 Base64를 디코딩하는 과정이 없다.

### 꼬리질문
#### 영어회화 퀴즈에서 TTS를 어떻게 쓰면 좋을까요?
모범 답안을 TTS로 읽혀 Audio 컴포넌트에 띄우고, speed를 0.75 정도로 살짝 늦추면 따라 하기 좋을 것 같습니다. 60초 분량이면 80초가 됩니다. instructions로 "또박또박 천천히 발음하는 영어 선생님처럼"을 줄 수도 있습니다.

## Q. Transcription API로 음성을 받아 적어 보니 결과가 어땠나요?
id: mm-transcription-exp
type: 경험
level: 기본
tags: Transcription, gpt-4o-mini-transcribe, 문장부호
source: 5. Audio API › Transcription: 소리를 글로

### 30초 답변
TTS로 만든 speech.mp3를 그대로 gpt-4o-mini-transcribe에 넣었더니 '안녕하세요? 좋은 날씨예요?'가 나왔습니다. 원래 문장은 "안녕하세요, 좋은 날씨예요!"였는데 글자는 다 맞았지만 느낌표가 물음표로 바뀌었습니다. 문장부호는 소리에 직접 들어 있지 않고 억양으로 추측해야 해서 완벽하지 않다는 게 보였습니다. 이 결과는 강의 자료의 실행 결과를 옮긴 것입니다.

### 자세히
```python
audio = open("speech.mp3", "rb")
transcription = client.audio.transcriptions.create(model="gpt-4o-mini-transcribe", file=audio)
transcription.text   # '안녕하세요? 좋은 날씨예요?'
```

| 모델 | 분당 가격 (강의 자료) |
|---|---|
| gpt-4o-transcribe | 약 $0.006 |
| Whisper | 약 $0.006 |
| gpt-4o-mini-transcribe | $0.003 (가장 쌈) |

- 파일을 `"rb"`로 여는 이유는 이미지 편집과 같다. 오디오 바이트를 그대로 보내야 한다.

### 꼬리질문
#### 전사 결과의 문장부호를 믿고 후처리해도 되나요?
문장부호로 의문문인지 판단하는 식의 로직은 위험합니다. 이번처럼 평서문이 물음표로 바뀔 수 있으니, 의미 판단은 전사 결과를 LLM에 넘겨 맥락으로 하게 하는 편이 낫습니다.

## Q. 전사 API의 25MB 파일 제한은 왜 신경 써야 하고, 긴 녹음은 어떻게 처리하나요?
id: mm-audio-file-size
type: 이유
level: 심화
tags: 파일 크기, 25MB, WAV, MP3, 비트레이트
source: 5. Audio API › Transcription: 소리를 글로

### 30초 답변
OpenAI 문서 기준으로 전사 API는 파일 하나에 25MB까지 받는데, 형식에 따라 들어가는 길이가 크게 다릅니다. CD 음질 WAV(44.1kHz 스테레오)는 1분에 약 10MB라 2분 반밖에 안 들어가지만, 음성 인식에 충분한 16kHz 모노는 1분에 1.83MB라 약 14분, MP3 64kbps는 약 55분이 들어갑니다. 긴 회의 녹음은 인식에 충분한 음질로 낮추거나 압축하고, 그래도 넘으면 잘라서 보내야 합니다.

### 자세히
| 형식 | 1분 크기 | 25MB에 들어가는 길이 |
|---|---|---|
| WAV 44.1kHz 스테레오 16비트 | 약 10.09MB | 약 2.5분 |
| WAV 16kHz 모노 16비트 | 1.83MB | 약 14분 |
| MP3 128kbps | 약 0.92MB | 약 27분 |
| MP3 64kbps | 약 0.46MB | 약 55분 |

- WAV 크기 = 샘플링 주파수 × 채널 수 × 2바이트 × 초. 16kHz 모노 1분 = 16,000 × 1 × 2 × 60 = 1,920,000바이트
- 블로그 계산은 MB = 1024 × 1024바이트 기준이다. 실제 파일은 메타데이터 때문에 조금 다를 수 있다.
- 25MB 제한은 강의 자료에 없던 내용으로, 블로그에서 OpenAI 문서를 보고 덧붙였다.

### 꼬리질문
#### 녹음을 잘라서 보낼 때 주의할 점은 무엇인가요?
정해진 시간마다 기계적으로 자르면 단어나 문장 중간이 잘려 그 부분이 틀리게 전사될 수 있습니다. 보충: 말이 끊긴 조용한 구간에서 자르거나 조각끼리 몇 초씩 겹치게 자른 뒤, 이어 붙일 때 겹친 부분을 정리하는 방법을 씁니다.

## Q. whisper-1로 한국어 음성을 영어로 번역해 보니 어떤 문제가 있었나요?
id: mm-whisper-translation-exp
type: 경험
level: 기본
tags: Translation, whisper-1, 로마자, 한계
source: 5. Audio API › Translation: 소리를 바로 번역하기

### 30초 답변
"안녕하세요. 오늘도 공부하기 참 좋은 날씨입니다. 모두 힘내 봐요"라는 한국어 음성을 whisper-1 번역에 넣었더니, 영어로 번역하지 않고 'Annyeonghaseyo. Oneuldo gungbuhagi cham joeun nalssi imnida…'처럼 한국어 발음을 로마자로 옮겨 적었습니다. 음성 번역 모델은 사실상 whisper-1 하나뿐인데, 강의에서도 "조금 도태된 모델"이라 소개했고 특히 한국어에서 제대로 동작하지 않을 때가 많다고 했습니다. 이 결과는 강의 자료의 실행 결과를 옮긴 것입니다.

### 자세히
```python
translation = client.audio.translations.create(model="whisper-1", file=audio)
translation.text
# 'Annyeonghaseyo. Oneuldo gungbuhagi cham joeun nalssi imnida. Modu himnae boahyo.'
```

- 번역이 아니라 소리 나는 대로 받아 적은 결과다. "공부"가 "gungbu"로 적히는 등 받아쓰기도 정확하지 않다.
- 실시간 번역 모델(gpt-realtime-translate 등)도 있지만 2배에서 6배까지 비싸다.
- 강의 슬라이드 제목: "Whisper는… 조금 실전에서 쓰기 어렵긴 하다"

### 꼬리질문
#### 이 경험으로 앱 설계를 어떻게 바꿨나요?
통역기 앱을 whisper-1 한 번에 맡기지 않고, gpt-4o-mini-transcribe로 받아 적은 뒤 LLM에게 번역시키는 두 단계로 짰습니다. 중간에 받아 적은 한국어도 화면에 보여 줘서 어디서 틀렸는지 확인할 수 있게 했습니다.

## Q. 음성 번역에 Translation 모델 대신 Transcription + LLM 조합을 쓰는 이유는 무엇인가요?
id: mm-transcription-llm
type: 이유
level: 기본
tags: Transcription, LLM, 번역, instructions, 파이프라인
source: 5. Audio API › 그래서 Transcription + LLM

### 30초 답변
번역 모델은 영어로만 번역할 수 있고, 말투를 지정할 수 없으며, LLM이 이미 번역을 아주 잘하기 때문입니다. 받아쓰기는 받아쓰기 모델에게, 번역은 LLM에게 나눠 맡기면 각자 잘하는 일만 하게 됩니다. 실제로 whisper-1이 로마자로 옮기던 같은 음성이 이 방식에서는 'Hello. It's a wonderfully pleasant day for studying today…'처럼 자연스러운 영어로 나왔습니다.

### 자세히
```python
transcription = client.audio.transcriptions.create(model="gpt-4o-mini-transcribe", file=audio)
response = client.responses.create(
    model="gpt-4o-mini",
    input=transcription.text,
    instructions="너는 한영 번역가이다. 한글이 들어오면, 그 문장을 영어로 번역해서 반환하라.",
)
response.output_text
```

| 이유 | 내용 |
|---|---|
| 출력 언어 | Translation 모델은 여러 언어 음성을 영어로만 번역한다(OpenAI 문서 기준. 강의 슬라이드의 "출력 13개 언어"와 다름). LLM은 원하는 언어로 |
| 말투 지정 | "비즈니스 영어로", "초등학생도 이해할 수 있게", "고유명사는 원어 그대로" 등을 `instructions`에 |
| 번역 품질 | LLM은 전문 번역가 수준 이상의 성능 |

- 1일차의 `instructions`가 여기서 다시 쓰인다.

### 꼬리질문
#### 이 방식의 단점은 무엇인가요?
호출이 두 번이라 시간과 비용이 더 들고, 앞 단계에서 잘못 받아 적으면 LLM이 틀린 문장을 자연스럽게 번역해 버려 오류가 그럴듯하게 감춰집니다. 그래서 받아 적은 원문을 사용자에게 함께 보여 주는 것이 좋습니다.

## Q. 음성 앱을 여러 모델을 이은 파이프라인으로 만드는 것과 음성을 직접 다루는 실시간 모델로 만드는 것은 어떻게 다른가요?
id: mm-cascade-vs-realtime
type: 비교
level: 심화
tags: 파이프라인, Realtime API, 지연, 음성 대화
source: 1. 멀티모달이 뭔가 · 5. Audio API (일부 보충)

### 30초 답변
파이프라인은 받아쓰기 → LLM → TTS처럼 단계마다 모델을 따로 불러서, 중간 결과를 글로 확인하고 instructions로 세밀하게 조절할 수 있고 싼 모델을 골라 쓸 수 있습니다. 대신 호출이 세 번이라 지연이 쌓이고, 글로 바꾸는 순간 목소리의 감정이나 억양 같은 정보가 사라집니다. Realtime API는 소리를 스트림으로 보내고 바로 음성으로 답을 받는 방식이라 대화가 자연스럽지만, 블로그 표에서 실시간 번역 모델이 2배에서 6배 비쌌던 것처럼 비용이 큽니다.

### 자세히
| | 파이프라인 (Transcription → LLM → TTS) | 실시간 음성 모델 (Realtime API) |
|---|---|---|
| 연결 | 단계마다 따로 호출 | WebSocket, WebRTC로 스트림 |
| 지연 | 단계마다 기다림이 쌓임 | 짧음 |
| 중간 결과 | 글로 남아 확인·저장·검증 가능 | 보충: 확인하기 어려움 |
| 제어 | 단계별 모델 선택, instructions | 보충: 한 모델에 맡김 |
| 잃는 것 | 보충: 말투, 감정, 억양 (글로 바꾸며 사라짐) | |
| 비용 | 싼 모델 조합 가능 | 비쌈 |

- 이번 실습은 비용과 제어 때문에 파이프라인으로 만들었다.

### 꼬리질문
#### 파이프라인에서 지연을 줄이려면 어떻게 하나요?
보충: LLM 답을 stream으로 받아 문장이 끝날 때마다 TTS에 넘기면, 전체 답이 끝나기 전에 첫 문장부터 읽어 줄 수 있습니다. 1일차의 stream 옵션이 여기서 쓰입니다.

## Q. Gradio로 마이크 입력을 받아 전사하는 앱은 어떻게 만들고, Colab에서는 무엇을 주의해야 하나요?
id: mm-gradio-audio
type: 개념
level: 기본
tags: Gradio, gr.Audio, microphone, share
source: 6. Gradio로 소리 다루기

### 30초 답변
gr.Audio에 `sources=["microphone"]`, `type="filepath"`를 주면 함수가 녹음 파일의 경로를 받습니다. 이 경로를 `"rb"`로 열어 그대로 전사 모델에 넘기고 결과 텍스트를 Textbox에 돌려주면 끝입니다. Colab 안에서는 사생활 노출 위험 때문에 마이크를 쓰기 어려워서, `share=True`로 띄운 gradio.live 주소에서 테스트해야 했습니다.

### 자세히
| 파라미터 | 설명 |
|---|---|
| sources | `["upload", "microphone"]` 중 필요한 것만 노출 |
| label | 컴포넌트 위 이름 |
| type | `"numpy"` 또는 `"filepath"` |

```python
def translate(audio, lang):
    with open(audio, "rb") as f:
        transcription = client.audio.transcriptions.create(model="gpt-4o-mini-transcribe", file=f)
    return transcription.text

with gr.Blocks() as app:
    audio = gr.Audio(sources=["microphone"], type="filepath", label="마이크 입력")
    btn = gr.Button("transcribe", variant="primary")
    output = gr.Textbox(label="output")
    btn.click(fn=translate, inputs=[audio], outputs=output)
```

- 올린 소리는 파형으로 보이고 재생, 속도 조절, 자르기 버튼이 기본으로 붙는다.
- 강의 화면에서는 "마이크 테스트."라고 말한 게 그대로 찍혀 나왔다.
- gradio.live에서 처음 열면 마이크를 못 찾는다고 나온다. 녹음 버튼을 눌러 권한을 허용하고, 껐다가 다시 녹음하면 잡힌다.

### 꼬리질문
#### share=True 링크를 쓸 때 주의할 점은 무엇인가요?
공개 링크라서 주소를 아는 사람은 누구나 들어와 앱을 쓸 수 있고, 그 호출 비용은 제 API 키로 나갑니다. 테스트가 끝나면 꺼 두는 게 안전합니다.

#### 강의 예제의 함수는 lang 매개변수를 받는데 inputs에는 audio 하나뿐입니다. 문제가 없나요?
inputs의 컴포넌트 개수와 함수 매개변수 개수가 맞지 않아 그대로는 오류가 날 수 있습니다. 언어 선택 Dropdown을 inputs에 추가하거나, 매개변수에 기본값을 주거나 지워야 합니다. 이름도 translate이지만 실제로는 전사만 하니, 통역기로 확장할 때 LLM 번역 단계를 붙이면 이름과 맞게 됩니다.

## Q. 속담 퀴즈 앱은 어떻게 설계했고, 설계하면서 어떤 점을 고민했나요?
id: mm-proverb-quiz-exp
type: 경험
level: 심화
tags: 속담 퀴즈, Images API, gr.State, 프롬프트
source: 7. 실습 앱들 › 속담 퀴즈

### 30초 답변
속담 100개 리스트에서 하나를 무작위로 골라 Images API로 그림을 그리고, 사용자가 무슨 속담인지 맞히면 정답 여부와 정답, 누적 점수를 보여 준 뒤 자동으로 다음 문제로 넘어가는 구조입니다. 고민한 점은 두 가지였습니다. 우물 안 개구리처럼 그리기 쉬운 속담과 "모르면 약이요, 아는 게 병"처럼 거의 불가능한 속담이 섞여 있고, 프롬프트를 "속담의 장면을 글자 없이 그려줘"처럼 짜지 않으면 그림에 정답이 글자로 박혀 나올 수 있다는 점입니다.

### 자세히
| 요구 조건 | 구현 방법 |
|---|---|
| AI가 그림을 만들어 보여 줌 | `images.generate` → 파일 저장 → gr.Image에 경로 반환 |
| 사람이 답 입력 | gr.Textbox |
| 정답·오답과 정답 표시 | 현재 문제의 속담과 비교 |
| 점수 | 맞힌 수를 `gr.State`에 유지 |
| 자동으로 다음 문제 | 채점 후 새 속담을 골라 다시 생성 |

- 실습 노트북에는 속담 리스트만 들어 있어서 구조부터 직접 정해야 했다.
- 문제마다 그림을 새로 그리니 호출이 금방 쌓인다. 그래서 gpt-image-1-mini를 쓴다.

### 꼬리질문
#### 사용자의 답이 정답과 글자 하나만 달라도 오답 처리되는 문제는 어떻게 하나요?
문자열 완전 일치 대신, 정답 속담과 사용자 답을 LLM에 함께 주고 같은 속담인지 판단하게 하는 방법이 있습니다. 띄어쓰기나 조사 차이 정도는 공백을 지우고 비교하는 간단한 처리로도 많이 줄어듭니다.

## Q. 그림판 Artist 앱에서 Vision과 Images API를 어떻게 이어 붙였나요?
id: mm-artist-exp
type: 경험
level: 기본
tags: 그림판 Artist, Image to Text, Text to Image, ImageEditor
source: 7. 실습 앱들 › 그림판 Artist

### 30초 답변
손그림을 ImageEditor로 받아 composite를 Base64로 바꾼 뒤 Vision에 넣어 무엇을 그렸는지 글로 바꾸고, 그 설명으로 화가에게 넘길 프롬프트를 쓰게 했습니다. 그 프롬프트를 Images API에 넘겨 작품을 만들고, 생성된 그림을 파일로 저장해 출력 컴포넌트로 돌려주는 구조입니다. Image to Text와 Text to Image를 글을 사이에 두고 잇는 방식이라, 오늘 배운 걸 거의 다 쓰는 실습이었습니다.

### 자세히
```
ImageEditor (손그림) → composite → Base64 → Vision (I2T): "무엇을 그렸나" + 화가용 프롬프트
→ Images API (T2I) → b64_json 디코딩 → 파일 저장 → gr.Image 또는 Gallery
```

- 확장 아이디어: "고흐 스타일로", "수채화로" 같은 화풍 선택 Radio를 붙여 Gallery에 여러 버전을 띄운다.
- 두 모델 사이를 잇는 것은 글(프롬프트)이다. 그래서 중간 프롬프트를 확인하면 결과가 왜 그렇게 나왔는지 추적할 수 있다.

### 꼬리질문
#### 손그림을 바로 images.edit에 넣지 않고 Vision을 거치는 이유는 무엇일까요?
edit는 원본을 유지하려는 성질이 있어서 대충 그린 선과 구도가 그대로 남기 쉽습니다. Vision으로 "무엇을 그렸는지"만 글로 뽑아 새로 그리게 하면 손그림의 서툰 선에서 벗어나 화가 그림처럼 바꾸기 쉽습니다. 대신 원래 구도는 덜 지켜집니다.

## Q. 통역기와 영어회화 퀴즈처럼 음성이 들어가는 앱은 어떤 구조로 설계했나요?
id: mm-voice-apps-exp
type: 경험
level: 심화
tags: 통역기, 영어회화 퀴즈, Transcription, TTS, JSON
source: 7. 실습 앱들 › 통역기 앱, 영어회화 퀴즈

### 30초 답변
통역기는 마이크 입력을 Transcription으로 받아 적고, LLM이 고른 언어로 번역한 뒤, TTS로 읽어 주는 세 단계입니다. 언어 선택은 Dropdown으로 받아 `instructions`의 "영어로" 부분에 끼워 넣습니다. 영어회화 퀴즈는 녹음한 답을 받아 적고 LLM에게 문제와 내 답을 같이 줘서 점수, 피드백, 모범 답안을 받은 다음, 모범 답안을 TTS로 읽어 Audio 컴포넌트에 띄우는 구조로, 화면에 따로 뿌리기 쉽게 LLM에게 JSON 형식으로 답하게 하는 것이 편하다고 판단했습니다.

### 자세히
| 앱 | 흐름 | 포인트 |
|---|---|---|
| 통역기 | 마이크 → Transcription → LLM 번역 → TTS → 글 + 소리 | whisper-1은 한국어를 로마자로 적을 수 있어 받아쓰기 + LLM 번역이 맞다 |
| 영어회화 퀴즈 | 한국어 문제 → 녹음 → Transcription → LLM 채점(JSON) → 모범 답안 TTS | 점수 기준은 딱히 없어도 된다는 조건 |

- 통역기는 "내가 한 말을 글로 보여 주기"가 요구 조건에 있어, 받아 적은 원문과 번역문을 함께 보여 준다.
- 영어회화 퀴즈의 모범 답안 발음은 speed를 0.75 정도로 늦추면 따라 하기 좋을 것 같다.
- 마지막 자율 프로젝트는 이미지와 음성이 둘 다 들어간 앱을 직접 기획하는 과제였다.

### 꼬리질문
#### 발음 자체를 평가하려면 이 구조로 충분한가요?
충분하지 않습니다. 전사 모델은 발음이 조금 틀려도 맥락으로 맞는 단어를 적어 주기 때문에, 글로 바뀐 뒤에는 발음 정보가 대부분 사라집니다. 이 구조는 "무슨 말을 했는가"를 평가하는 것이고, 발음까지 보려면 소리를 직접 다루는 모델이나 발음 평가 전용 도구가 필요합니다.
