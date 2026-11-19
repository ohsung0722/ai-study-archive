# 자연어처리
blog: nlp_blog/nlp_blog.md
blog_title: [딥러닝 5일차] 자연어처리 정리: 단어를 숫자로 바꾸는 것부터 BERT와 GPT까지

## Q. 자연어처리(NLP)란 무엇이고, NLU와 NLG는 어떻게 다른가요?
id: nlp-nlu-nlg
type: 개념
level: 기본
tags: NLP, NLU, NLG, 역사
source: 1. 자연어처리란

### 30초 답변
자연어처리는 한국어, 영어처럼 사람이 일상에서 쓰는 자연어를 컴퓨터가 이해하거나 생성하게 만드는 분야입니다. 그래서 자연어 이해(NLU)와 자연어 생성(NLG)으로 나눠 부르기도 합니다. NLU는 "전화 실행해"에서 동작과 대상을 알아내는 것처럼 입력의 뜻을 파악하는 일이고, NLG는 기온 숫자를 받아 "이번 주는 지난주보다 6도가량 덥겠습니다" 같은 문장을 만드는 일입니다.

### 자세히
- 자연어의 반대는 인공어(프로그래밍 언어, 수식)다. 자연어는 단어 순서가 뜻을 바꾸는 시퀀스 데이터다.
- `NLP = NLU + NLG`

| | NLU | NLG |
|---|---|---|
| 하는 일 | 입력 문장의 의미 파악 | 의미나 데이터로 문장 생성 |
| 예 | 의도(intent)·대상(slot) 추출, 감성 분류 | 요약, 번역문 생성, 대화 응답 |

- 컴퓨터는 숫자만 계산하므로 언어를 숫자로 바꾸는 과정이 모든 방법의 출발점이다.
- 흐름: 규칙 기반(1966 ELIZA, 패턴 매칭) → 통계 기반(1990년대, n-gram) → 표현 학습(2013 Word2Vec) → 사전학습·LLM(2017 Transformer, 2018 BERT·GPT)

### 꼬리질문
#### 규칙 기반 방식은 왜 한계에 부딪혔나요?
언어에는 예외가 끝없이 많아서 예외 처리 규칙이 기하급수적으로 늘어나고, 결국 유지보수가 불가능해졌습니다. 그래서 규칙을 사람이 쓰지 않고 대량의 텍스트에서 확률을 추정하는 통계 기반 방식으로 넘어갔습니다.

#### n-gram 언어 모델의 한계는 무엇인가요?
앞의 n-1개 단어만 보고 다음 단어 확률을 계산하므로 그보다 먼 문맥은 반영하지 못합니다. n을 키우면 단어 조합 수가 폭발해서 대부분의 조합이 데이터에 한 번도 나오지 않는 희소성 문제가 생깁니다.

## Q. 토큰화란 무엇이고, 띄어쓰기 기준 토큰화는 한국어에서 어떤 한계가 있나요?
id: nlp-tokenization
type: 개념
level: 기본
tags: 토큰화, 전처리, 한국어
source: 2. NLP 입문: 문장을 숫자로 바꾸기

### 30초 답변
토큰화는 문장을 모델이 다룰 단위인 토큰으로 자르는 과정이고, 이후 토큰마다 번호를 붙여 어휘 사전을 만듭니다. 가장 단순한 방법은 띄어쓰기 기준으로 자르는 것인데, 토큰은 글자 모양으로만 구분되기 때문에 "최고였어요"와 "최고예요"처럼 뜻이 거의 같아도 다른 토큰이 됩니다. 한국어는 단어에 조사와 어미가 붙는 교착어라 이런 변형이 많아서, 어휘 사전이 커지고 처음 보는 토큰도 늘어납니다.

### 자세히
전처리 다섯 단계: **토큰화 → 어휘 사전 → 특수 토큰 → 인코딩·패딩 → 임베딩**

```
"정말 재미있었다"  →  ["정말", "재미있었다"]
"학교에", "학교를", "학교에서"  →  전부 다른 토큰
```

- 같은 단어가 조사마다 따로 사전에 올라가서 어휘가 커지고, 토큰 하나당 학습 데이터는 줄어든다.
- 보충: 한국어는 형태소 분석기(KoNLPy의 Mecab, Okt 등)로 "학교 / 에"처럼 나누거나, 서브워드 토큰화를 많이 쓴다.

### 꼬리질문
#### 문자 단위로 자르면 이 문제가 없지 않나요?
사전에 없는 토큰은 거의 사라지지만 시퀀스가 길어지고, 글자 하나에는 뜻이 거의 없어서 모델이 의미를 조합하는 부담이 커집니다. 그래서 단어와 문자의 중간인 서브워드 단위를 많이 씁니다.

#### 어휘 사전은 어떤 데이터로 만들어야 하나요?
학습 데이터로만 만듭니다. 테스트 데이터까지 넣어 만들면 평가 때 처음 보는 단어가 없는 것처럼 되어 성능이 실제보다 좋게 나옵니다.

## Q. 서브워드 토큰화(BPE, WordPiece)를 설명해 보세요.
id: nlp-subword
type: 개념
level: 심화
tags: BPE, WordPiece, SentencePiece, 토큰화
source: 보충 (블로그 밖 내용)

### 30초 답변
서브워드 토큰화는 단어를 자주 나오는 더 작은 조각으로 나누는 방법입니다. 자주 쓰는 단어는 통째로 토큰 하나가 되고 드문 단어는 여러 조각으로 쪼개지기 때문에, 어휘 크기를 적당히 유지하면서도 처음 보는 단어를 대부분 표현할 수 있습니다. BPE는 문자에서 시작해 가장 자주 붙어 나오는 쌍을 반복해서 합치고, BERT가 쓰는 WordPiece는 빈도 대신 학습 데이터의 가능도를 가장 많이 높이는 쌍을 합칩니다.

### 자세히
```
BPE 학습
1. 모든 단어를 문자 단위로 나눈다:  l o w e r
2. 인접한 쌍 중 가장 자주 나오는 쌍을 합친다:  (l, o) → lo
3. 정한 어휘 크기가 될 때까지 2를 반복한다
```

| | BPE | WordPiece | SentencePiece |
|---|---|---|---|
| 합치는 기준 | 쌍의 빈도 | 가능도 증가량 | BPE 또는 Unigram 방식 |
| 사용 예 | GPT 계열 (GPT-2는 바이트 단위 BPE) | BERT (어휘 약 3만) | T5, 다국어 모델 |
| 특징 | 단순 | 이어지는 조각에 `##` 표시 | 공백도 문자(`▁`)로 취급 |

- 예: `"playing"` → `["play", "##ing"]`
- 어휘가 작으면 쪼개짐이 많아 시퀀스가 길어지고, 크면 임베딩 표가 커진다. 병합 횟수로 이 균형을 조절한다.

### 꼬리질문
#### 서브워드를 쓰면 OOV가 완전히 사라지나요?
기본 단위인 문자가 모두 사전에 있으면 어떤 단어든 조각으로 표현할 수 있어 거의 사라집니다. 다만 학습 때 없던 문자는 여전히 `[UNK]`가 될 수 있고, 바이트 단위 BPE는 바이트 256개를 기본 단위로 둬서 이 문제까지 없앴습니다.

#### SentencePiece가 한국어 같은 언어에 잘 맞는 이유는?
띄어쓰기를 미리 단어 경계로 가정하지 않고 공백까지 하나의 문자로 다루기 때문입니다. 그래서 띄어쓰기가 불규칙하거나 없는 언어에도 같은 방식으로 적용할 수 있고, 토큰을 이어 붙이면 원래 문장으로 그대로 복원됩니다.

## Q. OOV 문제란 무엇이고, `<PAD>`, `<UNK>` 같은 특수 토큰은 왜 필요한가요?
id: nlp-oov-special-tokens
type: 개념
level: 기본
tags: OOV, 특수 토큰, 패딩, 전처리
source: 2. NLP 입문: 문장을 숫자로 바꾸기

### 30초 답변
OOV(Out-Of-Vocabulary)는 테스트나 실제 사용 때 어휘 사전에 없는 단어가 들어오는 문제입니다. 사전에 없는 단어는 번호가 없으니 `<UNK>` 토큰 하나로 바꿔서 처리합니다. `<PAD>`는 여러 문장을 배치로 묶기 위해 길이를 맞출 때 빈칸을 채우는 토큰이고, 이때 패딩 전 실제 길이를 따로 저장해 두어야 모델이 실제 마지막 단어 위치를 찾을 수 있습니다.

### 자세히
```
vocab = {"<PAD>": 0, "<UNK>": 1, "정말": 2, "영화": 3, ...}   # 실제 단어는 2번부터
"정말 대박 영화"  →  [2, 1, 3]            # "대박"이 사전에 없으면 <UNK>
최대 길이 5로 패딩 →  [2, 1, 3, 0, 0],  lengths = 3
```

- `nn.Embedding(..., padding_idx=0)`으로 두면 `<PAD>` 행은 0 벡터로 고정되고 학습되지 않는다.
- LSTM 출력은 `<PAD>`까지 다 읽은 마지막 시점이 아니라 `lengths - 1` 위치에서 꺼낸다. `<PAD>`를 다 읽은 출력은 문장이 끝난 지점의 정보가 흐려져 있다.
- 다른 특수 토큰: Seq2Seq의 시작·끝 토큰(`<SOS>`, `<EOS>`), BERT의 `[CLS]`, `[SEP]`, `[MASK]`

### 꼬리질문
#### `<UNK>`가 많아지면 어떤 문제가 생기나요?
서로 다른 단어가 전부 같은 벡터 하나로 뭉개져서 뜻을 잃습니다. 신조어나 오타가 많은 리뷰 데이터에서 특히 심하고, 그래서 서브워드 토큰화로 OOV 자체를 줄입니다.

#### 모든 문장을 가장 긴 문장 길이에 맞추면 낭비가 크지 않나요?
그래서 비슷한 길이의 문장끼리 배치로 묶거나, 배치마다 그 배치 안의 최대 길이에만 맞추는 방식을 씁니다. PyTorch에서는 `pack_padded_sequence`로 RNN이 패딩 부분을 건너뛰게 할 수도 있습니다.

## Q. 단어를 정수 인덱스 그대로 모델에 넣지 않고 임베딩을 쓰는 이유는 무엇인가요?
id: nlp-embedding
type: 이유
level: 기본
tags: Embedding, 전처리
source: 2. NLP 입문 › 번호만으로는 왜 안 될까

### 30초 답변
단어 번호는 이름표일 뿐이라 크기나 차이에 의미가 없습니다. 4번 단어가 3번 단어보다 크다거나, 번호끼리 뺀 값이 다른 단어를 뜻하지 않습니다. 그래서 단어마다 여러 숫자로 된 벡터를 할당하는 임베딩을 쓰고, 이 벡터를 모델과 함께 학습시켜 단어 사이의 관계가 거리와 방향으로 표현되게 만듭니다.

### 자세히
- 주가 100과 200은 크기도 차이도 의미가 있다. 단어 번호는 그렇지 않다. `5(별로다) − 3(정말) = 2(영화)` 같은 계산은 아무 뜻이 없다.
- 번호를 그대로 넣으면 모델은 숫자 크기에 의미가 있다고 보고 가중치를 곱한다. 번호 붙이는 순서만 바꿔도 입력이 달라진다.
- 임베딩은 단어마다 학습되는 벡터를 하나씩 둔다. 비슷한 단어는 가깝게, 다른 단어는 멀게 배치될 수 있다.
- 임베딩이 가져온 변화
  - 단어 사이 유사성을 거리와 방향으로 계산할 수 있다.
  - 처음 보는 표현도 이미 배운 비슷한 단어에 빗대어 이해할 수 있다.
  - 언어의 관계를 벡터 연산으로 다룰 수 있다.

### 꼬리질문
#### 임베딩 차원은 어떻게 정하나요?
어휘 크기와 데이터 양에 맞춰 정하는 하이퍼파라미터입니다. 너무 작으면 단어 사이의 여러 관계를 담지 못하고, 너무 크면 파라미터가 늘어 적은 데이터에서 과적합되기 쉽습니다. 실습 같은 작은 예제는 2~4차원, Word2Vec은 보통 100~300차원, BERT-base는 768차원을 씁니다.

#### 미리 학습된 임베딩을 가져다 쓸 수도 있나요?
네, Word2Vec 같은 미리 학습된 벡터로 임베딩 표를 초기화할 수 있습니다. 데이터가 적을 때 유리하고, 표를 고정할지 함께 학습할지는 데이터 양을 보고 정합니다.

## Q. 원-핫 인코딩과 임베딩을 비교해 보세요.
id: nlp-onehot-vs-embedding
type: 비교
level: 기본
tags: One-hot, Embedding, 희소 벡터
source: 2. NLP 입문 › 번호만으로는 왜 안 될까

### 30초 답변
원-핫 인코딩은 어휘 수만큼 긴 벡터에서 자기 자리만 1이고 나머지는 0인 희소 벡터입니다. 모든 단어 쌍이 서로 직교해서 내적이 0이므로 어떤 단어끼리 비슷한지 표현하지 못합니다. 임베딩은 수십~수백 차원의 밀집 벡터이고 값을 학습하기 때문에 비슷한 단어를 가깝게 둘 수 있습니다. 원-핫 벡터에 임베딩 행렬을 곱하면 그 행을 꺼내는 것과 같아서, 임베딩은 이 곱셈을 조회로 바꾼 것으로 볼 수 있습니다.

### 자세히
| | 원-핫 | 임베딩 |
|---|---|---|
| 차원 | 어휘 크기 V (수만) | 정한 차원 d (수십~수백) |
| 값 | 0과 1, 대부분 0 (희소) | 실수 (밀집) |
| 단어 사이 관계 | 모두 직교, 유사도 0 | 학습으로 거리·방향에 담김 |
| 학습 | 고정 | 파라미터로 학습 |

```
E: (V, d) 임베딩 행렬
one_hot(3) @ E  ==  E[3]        # [0,0,0,1,0] @ E → E의 3번 행
```

- 실제로 곱하면 대부분 0을 곱하는 계산이라 낭비다. `nn.Embedding`은 행을 바로 꺼낸다.

### 꼬리질문
#### 원-핫 벡터끼리의 유클리드 거리는 어떻게 되나요?
서로 다른 두 단어는 항상 √2로 같습니다. "최고"와 "감동" 사이나 "최고"와 "최악" 사이가 똑같아서 의미 차이를 전혀 담지 못합니다.

#### 원-핫은 지금도 어디에 쓰이나요?
분류 문제의 정답 라벨을 표현할 때나, 종류가 적은 범주형 특성을 다룰 때 씁니다. Softmax 출력과 비교하는 정답 분포도 원-핫 형태입니다.

## Q. nn.Embedding은 어떻게 동작하고, 어떻게 학습되나요?
id: nlp-nn-embedding
type: 개념
level: 기본
tags: Embedding, PyTorch, 역전파
source: 2. NLP 입문 › 임베딩도 학습된다

### 30초 답변
`nn.Embedding`은 (어휘 크기, 임베딩 차원) 크기의 표를 파라미터로 들고 있다가, 단어 번호가 들어오면 그 번호의 행을 꺼내 주는 층입니다. 처음 값은 무작위지만 표 자체가 학습 파라미터라서, optimizer에 넘겨 주면 `loss.backward()`가 출력층을 지나 표까지 기울기를 보내고 `optimizer.step()`이 표의 값을 고칩니다. `padding_idx`를 지정하면 그 행은 0으로 고정되어 학습되지 않습니다.

### 자세히
```python
emb = nn.Embedding(num_embeddings=10000, embedding_dim=128, padding_idx=0)
x = torch.tensor([[2, 5, 9, 0]])   # (batch=1, seq_len=4) LongTensor
emb(x).shape                       # (1, 4, 128)
emb.weight.shape                   # (10000, 128)  ← 학습 파라미터

optimizer = torch.optim.Adam(list(emb.parameters()) + list(fc.parameters()))
```

- 입력은 정수 텐서여야 하고, 출력은 입력 shape 뒤에 차원 하나가 붙는다.
- 기울기는 이번 배치에 나온 단어의 행에만 생기고, 나머지 행의 기울기는 0이다.
- `padding_idx` 행은 0으로 초기화되고 기울기를 받지 않는다.

### 꼬리질문
#### 임베딩 층과 bias 없는 Linear 층은 무엇이 다른가요?
원-핫 입력에 bias 없는 Linear를 적용한 것과 결과가 같습니다. 다만 Embedding은 곱셈 없이 행을 바로 꺼내므로 빠르고 메모리를 덜 씁니다.

#### 학습된 임베딩 값은 어떻게 꺼내 보나요?
`emb.weight`가 (어휘 크기, 차원) 텐서라서 단어 번호로 행을 꺼내면 됩니다. 감성 분류 실험에서도 이렇게 꺼낸 2차원 벡터를 그대로 좌표로 그렸습니다.

## Q. 감성 분류만 학습시켰는데 단어 임베딩은 어떻게 변했나요?
id: nlp-sentiment-embedding-exp
type: 경험
level: 기본
tags: Embedding, 감성 분류, 실험
source: 2. NLP 입문 › 임베딩도 학습된다

### 30초 답변
긍정 단어 6개, 부정 단어 6개, 양쪽에 다 나오는 단어 6개로 리뷰 120개를 만들고 2차원 임베딩을 학습시켜 봤습니다. 학습 전에는 단어가 무작위로 흩어져 있었는데, 300 epoch 뒤에는 긍정 단어와 부정 단어가 반대쪽으로 모였고 "영화", "배우"처럼 양쪽에 다 나오는 단어는 가운데에 남았습니다. 단어끼리 비슷하다고 알려 준 적이 없는데, 긍정·부정을 맞히다 보니 분류에 유리한 방향으로 벡터가 정리된 것입니다.

### 자세히
- 모델: 문장 안 토큰 벡터의 평균으로 긍정 확률을 내는 단순한 구조, 임베딩 2차원
- 학습 후 좌표

| 단어 | 벡터 | 위치 |
|---|---|---|
| 최고 | (-1.48, -0.42) | 긍정 쪽 (왼쪽 아래) |
| 최악 | (1.86, 0.78) | 부정 쪽 (오른쪽 위) |
| 영화 | (-0.25, 0.01) | 원점 근처 |

- 핵심은 optimizer에 `emb.parameters()`도 같이 넘긴 것이다. 그래서 `backward()`가 표까지 기울기를 보냈다.
- "영화"는 긍정·부정 리뷰에 고르게 나와서 어느 한쪽으로 옮겨도 손실이 줄지 않으니 원점 근처에 머문다.

### 꼬리질문
#### 이렇게 학습된 임베딩을 다른 작업에도 쓸 수 있나요?
쓰기 어렵습니다. 이 벡터는 긍정·부정을 가르는 방향만 배웠기 때문에, 감성이 같은 "최고"와 "감동"은 가깝지만 그 밖의 뜻이나 관계는 담겨 있지 않습니다. 단어의 일반적인 의미를 담으려면 Word2Vec처럼 문맥 예측으로 학습해야 합니다.

#### 2차원으로 실험한 이유는 무엇인가요?
좌표를 그대로 그려서 학습 전후를 눈으로 비교하기 위해서입니다. 실제 모델은 훨씬 큰 차원을 쓰고, 그림으로 볼 때는 PCA나 t-SNE로 2차원으로 줄입니다.

## Q. Word2Vec은 어떤 아이디어로 단어 벡터를 학습하나요?
id: nlp-word2vec
type: 개념
level: 기본
tags: Word2Vec, Embedding, 분포 가설
source: 3. Word2Vec › 아이디어

### 30초 답변
Word2Vec(2013)은 비슷한 문맥에 나오는 단어는 의미도 비슷하다는 분포 가설을 바탕으로 단어 벡터를 학습합니다. 중심 단어와 주변 단어를 서로 예측하도록 학습시키면, 같은 예측을 해야 하는 단어들은 벡터도 비슷해질 수밖에 없습니다. 단어 유사도를 직접 학습하는 것이 아니라 문맥 예측의 결과로 의미가 담긴 벡터가 생긴다는 점이 핵심입니다.

### 자세히
- 예: `"__ is sweet fruit"`의 빈칸에는 apple, banana, grape가 다 들어간다. 주변 단어가 같으니 벡터도 비슷해진다.
- 모델은 `임베딩 → 선형층 → Softmax`. 정답이 어휘 중 하나인 다중 분류라 `CrossEntropyLoss`를 쓴다.

```python
class SkipGram(nn.Module):
    def __init__(self, vocab_size, dim=3):
        super().__init__()
        self.emb = nn.Embedding(vocab_size, dim)
        self.out = nn.Linear(dim, vocab_size)
    def forward(self, center):
        return self.out(self.emb(center))    # 모든 단어에 대한 점수
```

- 학습이 끝나면 `emb.weight`를 단어 벡터로 쓴다.
- 라벨이 필요 없다. 문장만 있으면 (중심, 주변) 쌍을 만들 수 있다.

### 꼬리질문
#### 어휘가 수만 개면 Softmax 계산이 너무 무겁지 않나요?
그렇습니다. 학습 표본마다 어휘 전체의 점수를 내야 하므로, 원 논문은 Negative Sampling과 Hierarchical Softmax로 이를 줄였습니다. Negative Sampling은 실제 주변 단어 하나와 무작위로 뽑은 몇 개의 오답 단어만 두고 이진 분류로 바꿔 계산량을 크게 줄입니다.

#### 학습 때 보지 못한 단어는 어떻게 하나요?
Word2Vec은 사전에 있는 단어만 벡터가 있어서 처리하지 못합니다. FastText는 단어를 문자 n-gram 조각들의 합으로 표현해서, 처음 보는 단어도 조각 벡터를 더해 벡터를 만들 수 있게 했습니다.

## Q. CBOW와 Skip-gram은 어떻게 다른가요?
id: nlp-cbow-vs-skipgram
type: 비교
level: 기본
tags: Word2Vec, CBOW, Skip-gram
source: 3. Word2Vec › CBOW와 Skip-gram

### 30초 답변
둘 다 Word2Vec의 학습 방식인데 예측 방향이 반대입니다. CBOW는 주변 단어 벡터를 평균 내서 가운데 중심 단어를 맞히고, Skip-gram은 중심 단어 하나로 주변 단어 각각을 맞힙니다. CBOW는 문맥 하나당 예측이 한 번이라 빠르고, Skip-gram은 중심 단어 하나가 여러 학습 쌍을 만들어서 느리지만 드문 단어의 벡터를 더 잘 배우는 편입니다.

### 자세히
문장 `the cat sat on the mat`, 중심 단어 `sat`, window = 2

| | 입력 | 정답 | 학습 표본 |
|---|---|---|---|
| CBOW | the, cat, on, the (평균) | sat | 1개 |
| Skip-gram | sat | the / cat / on / the | 4개 |

```python
for j in range(max(0, i - window), min(len(ids), i + window + 1)):
    if j != i:
        pairs.append((center, ids[j]))   # Skip-gram (중심, 주변) 쌍
```

- 실습에서는 Skip-gram을 썼다. window가 2면 중심 단어마다 최대 4쌍이 나온다.

### 꼬리질문
#### Skip-gram이 드문 단어에 유리한 이유는 무엇인가요?
드문 단어도 나올 때마다 중심 단어로서 주변 단어 수만큼 자기 벡터를 직접 업데이트받기 때문입니다. CBOW에서는 드문 단어가 주변 단어로 들어가면 다른 단어들과 평균되어 신호가 묽어집니다.

#### window 크기를 키우면 어떻게 되나요?
더 먼 단어까지 문맥으로 쓰므로 주제가 비슷한 단어들이 가까워지는 경향이 있고, 작으면 문법적 역할이 비슷한 단어끼리 가까워지는 경향이 있습니다. 학습 쌍이 늘어나므로 계산도 많아집니다.

## Q. 코사인 유사도란 무엇이고, 왜 유클리드 거리 대신 많이 쓰나요?
id: nlp-cosine-similarity
type: 개념
level: 기본
tags: 코사인 유사도, Embedding, Word2Vec
source: 3. Word2Vec › 코사인 유사도

### 30초 답변
코사인 유사도는 두 벡터의 내적을 두 벡터 크기의 곱으로 나눈 값으로, 두 벡터 사이 각도의 코사인입니다. 1에 가까우면 방향이 같고, 0이면 관련이 없고, -1이면 반대 방향입니다. 길이는 무시하고 방향만 보기 때문에, 빈도 같은 요인으로 벡터 길이가 달라져도 의미가 비슷한 단어를 비슷하다고 판단할 수 있어서 유클리드 거리 대신 많이 씁니다.

### 자세히
```
cos θ = (A · B) / (‖A‖ ‖B‖)
```

- A = (1, 1), B = (3, 3)이면 유클리드 거리는 2.83으로 멀지만 코사인 유사도는 1이다.
- 길이 1로 정규화한 벡터끼리는 코사인 유사도가 내적과 같고, `‖A − B‖² = 2 − 2cos θ`라서 유클리드 거리와 순위도 같아진다.
- 실험: 학습한 9개 단어의 코사인 유사도는 같은 묶음끼리 0.81~1.00, 과일과 동물 사이 -0.6~-0.9였다.

### 꼬리질문
#### 코사인 유사도와 Attention의 내적 점수는 어떤 관계인가요?
Attention 점수는 정규화하지 않은 내적이라 방향뿐 아니라 벡터 크기도 반영됩니다. 코사인 유사도는 내적을 두 크기로 나눈 것이므로, 길이가 1인 벡터끼리라면 둘은 같습니다.

#### 유사도가 음수면 반대 의미라는 뜻인가요?
꼭 그렇지는 않습니다. "최고"와 "최악" 같은 반의어는 문맥이 비슷해서 Word2Vec에서 오히려 유사도가 높게 나오기도 합니다. 실험의 음수는 작은 말뭉치에서 과일과 동물이 서로 겹치지 않는 문맥에만 나왔기 때문입니다.

## Q. King − Man + Woman ≈ Queen은 무엇을 의미하나요?
id: nlp-word-analogy
type: 개념
level: 심화
tags: Word2Vec, 벡터 연산, 유추
source: 3. Word2Vec › 벡터 차이로 관계를 계산한다

### 30초 답변
단어 사이의 관계가 벡터 공간에서 비슷한 방향의 차이 벡터로 나타난다는 뜻입니다. King에서 Queen으로 가는 차이와 Man에서 Woman으로 가는 차이가 비슷하므로, King − Man + Woman을 계산하면 Queen 근처의 벡터가 나옵니다. "나라→수도" 같은 관계도 비슷하게 나타나서, 임베딩이 비슷한 단어를 모을 뿐 아니라 관계 구조까지 담는다는 근거로 자주 쓰입니다.

### 자세히
```
Queen − King ≈ Woman − Man
King − Man + Woman ≈ Queen
Seoul − Korea + Japan ≈ Tokyo        # 나라 → 수도
```

- 계산한 벡터와 코사인 유사도가 가장 높은 단어를 답으로 고른다. 보통 입력 단어(King, Man, Woman)는 후보에서 뺀다.
- 이런 관계도 따로 가르친 것이 아니라 문맥 예측에서 생긴다. King과 Queen은 비슷한 문맥에 나오고, 차이는 주로 성별을 드러내는 주변 단어에서 생긴다.
- 블로그 그림은 설명용 좌표다. 54문장 말뭉치에는 King도 Queen도 없어서 직접 확인하지는 못했고, 이런 관계가 나오려면 큰 말뭉치가 필요하다.

### 꼬리질문
#### 입력 단어를 후보에서 빼지 않으면 어떻게 되나요?
King − Man + Woman의 결과가 King 자신과 가장 가깝게 나오는 경우가 많습니다. 그래서 유추 성능을 잴 때도 보통 입력 단어를 제외하고 가장 가까운 단어를 찾습니다.

#### 이런 벡터 관계가 문제가 되는 경우도 있나요?
말뭉치의 편향도 그대로 관계로 담깁니다. 예를 들어 직업 단어가 성별 방향으로 치우쳐 배치되는 현상이 보고되었고, 그래서 임베딩의 편향을 줄이는 연구가 이어졌습니다.

## Q. Word2Vec 같은 정적 임베딩과 BERT 같은 문맥 임베딩은 어떻게 다른가요?
id: nlp-static-vs-contextual
type: 비교
level: 심화
tags: Word2Vec, BERT, 문맥 임베딩, 다의어
source: 보충 (블로그 밖 내용)

### 30초 답변
Word2Vec 같은 정적 임베딩은 단어마다 벡터가 하나로 고정되어, 문장이 달라도 같은 단어는 같은 벡터를 씁니다. BERT 같은 문맥 임베딩은 문장 전체를 Self-Attention으로 읽은 뒤 단어마다 출력 벡터를 만들기 때문에, 같은 단어라도 주변 문맥에 따라 다른 벡터가 나옵니다. 그래서 "배를 먹었다"와 "배를 탔다"의 "배"처럼 뜻이 여러 개인 단어를 구분할 수 있습니다.

### 자세히
| | 정적 임베딩 | 문맥 임베딩 |
|---|---|---|
| 예 | Word2Vec, GloVe, FastText | ELMo, BERT, GPT의 층 출력 |
| 단어당 벡터 | 1개 (표에서 조회) | 문장마다 다름 (모델 출력) |
| 다의어 | 여러 뜻이 섞인 벡터 하나 | 문맥에 맞게 구분 |
| 비용 | 조회만 하면 됨 | 문장마다 모델을 통과해야 함 |

- BERT도 입력 단계의 토큰 임베딩은 정적인 표다. 층을 지나며 주변 단어 정보를 섞어서 문맥 벡터가 된다.
- Self-Attention 출력은 주변 단어의 Value를 비중대로 섞은 값이라, 같은 단어도 문장마다 결과가 달라진다.

### 꼬리질문
#### ELMo는 어떻게 문맥 임베딩을 만들었나요?
ELMo(2018)는 왼쪽에서 오른쪽, 오른쪽에서 왼쪽으로 읽는 LSTM 언어 모델을 각각 학습하고, 여러 층의 hidden state를 가중합해 단어 벡터로 썼습니다. 두 방향을 따로 학습해 이어 붙인 것이라, 한 층에서 양쪽을 동시에 보는 BERT와는 다릅니다.

#### 그럼 정적 임베딩은 이제 쓸모가 없나요?
조회만 하면 되므로 가볍고 빨라서, 계산 자원이 적은 환경이나 단어 수준 유사어 검색 같은 곳에서는 여전히 씁니다. 정확도가 중요하고 문맥이 필요하면 문맥 임베딩을 씁니다.

## Q. Word2Vec을 직접 학습시켜 보니 어땠나요?
id: nlp-word2vec-exp
type: 경험
level: 기본
tags: Word2Vec, Skip-gram, 실험
source: 3. Word2Vec › 직접 학습시켜 보기

### 30초 답변
과일, 동물, 탈것 세 묶음으로 영어 문장 54개를 만들어 3차원 Skip-gram을 학습시켰습니다. 1500 epoch 동안 손실이 4.21에서 2.27로 내려갔고, 같은 묶음 단어끼리 코사인 유사도가 0.81~1.00으로 묶음별로 확실히 모였습니다. 의외였던 건 dog와 탈것 단어 사이가 0.46~0.59로 높게 나온 점인데, "the dog runs fast"가 "the train moves fast"와 주변 단어를 공유했기 때문이었습니다.

### 자세히
- 설정: 묶음 단어 9개(apple, banana, grape / dog, cat, rabbit / car, bus, train), 임베딩 3차원, Adam, 학습률 0.01(수업 노트북 0.001에서 올림), 1500 epoch

| 비교 | 코사인 유사도 |
|---|---|
| 같은 묶음끼리 | 0.81 ~ 1.00 |
| 과일 ↔ 동물 | -0.6 ~ -0.9 |
| dog ↔ 탈것 | 0.46 ~ 0.59 |

- 과일끼리는 거의 1.00으로 붙었다. 말뭉치가 작고 문장 틀이 반복된 탓이다.
- 손실이 0까지 안 가는 이유: "the" 다음에 cat이 올지 dog가 올지는 원래 정해져 있지 않다. 같은 입력에 정답이 여러 개라 확률을 나눠 가질 수밖에 없다.

### 꼬리질문
#### 손실이 2.27이면 학습이 덜 된 것 아닌가요?
아닙니다. Skip-gram은 같은 중심 단어에 정답 주변 단어가 여러 개라서 이론상 최소 손실도 0보다 큽니다. 손실 값 자체보다 같은 묶음끼리 모였는지로 판단하는 것이 맞습니다.

#### 이 결과를 큰 말뭉치에서도 그대로 기대할 수 있나요?
그렇지 않습니다. 같은 묶음이 1.00에 가깝게 붙은 것은 문장 틀이 반복된 작은 말뭉치라서 나온 결과입니다. 반대로 dog가 문장 한 줄 때문에 탈것 쪽으로 끌려간 현상은 Word2Vec이 문맥만 보고 배운다는 것을 잘 보여 줍니다.

## Q. Seq2Seq 모델의 구조를 설명해 보세요.
id: nlp-seq2seq
type: 개념
level: 기본
tags: Seq2Seq, Encoder-Decoder, RNN
source: 4. Seq2Seq › Encoder와 Decoder

### 30초 답변
Seq2Seq은 번역처럼 입력과 출력 길이가 다른 문제를 풀기 위해 RNN(또는 LSTM) 두 개를 이은 Encoder-Decoder 구조입니다. Encoder가 입력 문장을 한 단어씩 읽어 마지막 hidden state를 문장 전체를 압축한 context vector로 만들고, Decoder는 이 벡터를 첫 hidden state로 받아 단어를 하나씩 생성합니다. 방금 만든 단어를 다음 시점 입력으로 다시 넣고, `<EOS>`가 나오면 멈춥니다.

### 자세히
- 기존 RNN 구조로 안 되는 이유: Many-to-Many는 입력과 출력 길이가 같고, Many-to-One은 출력이 하나다. "공부하러 도서관에 갑니다"(3어절) → "I am going to the library to study."(8단어)는 둘 다 아니다.

```
Encoder:  x₁ → x₂ → … → x_T   →  h_T = context vector
Decoder:  (h_T, <SOS>) → y₁ → y₂ → … → <EOS>
```

- Encoder와 Decoder를 잇는 것은 고정 크기 context vector 하나뿐이다.
- 원 논문(Sutskever et al., 2014)은 여러 층 LSTM으로 영어→프랑스어 번역을 했고, 입력 문장을 거꾸로 넣으면 성능이 좋아진다고 보고했다.
- 한계는 두 가지다. 긴 문장의 정보 손실(context vector 병목)과 추론 때의 오류 누적.

### 꼬리질문
#### Encoder와 Decoder는 가중치를 공유하나요?
보통 공유하지 않습니다. 입력 언어와 출력 언어가 다르고 하는 일도 읽기와 생성으로 다르기 때문에, 각자 따로 RNN과 임베딩을 둡니다.

#### 입력을 거꾸로 넣으면 왜 좋아졌나요?
원문의 앞쪽 단어가 Decoder가 처음 만드는 단어와 가까워지기 때문입니다. 번역문 앞부분과 대응하는 원문 단어 사이의 거리가 짧아져서 초반 단어를 맞히기 쉬워지고, 그 덕분에 학습이 잘 됐다고 설명합니다.

## Q. Seq2Seq의 context vector 병목을 직접 확인해 봤나요?
id: nlp-context-bottleneck-exp
type: 경험
level: 심화
tags: Seq2Seq, 장기 의존성, 실험
source: 4. Seq2Seq › Seq2Seq의 한계

### 30초 답변
context vector가 4칸인 tanh RNN에 문장을 넣고, 첫 단어만 다른 단어로 바꿨을 때 마지막 context vector가 얼마나 달라지는지 길이별로 300번씩 평균을 내 봤습니다. 길이가 10만 돼도 차이가 10⁻³~10⁻² 수준으로 줄었고, 30단어에서는 거의 0이었습니다. 첫 단어를 무엇으로 바꾸든 context vector가 거의 같다는 뜻이라, RNN의 장기 의존성 문제가 Seq2Seq에 그대로 남아 있다는 것을 확인했습니다.

### 자세히
- 방법: 무작위 가중치 tanh RNN, hidden 4칸, 첫 단어만 교체 → 마지막 hidden state 차이 측정, 길이별 300회 평균

| 문장 길이 | 첫 단어를 바꿨을 때 context vector 차이 |
|---|---|
| 10 | 10⁻³ ~ 10⁻² |
| 30 | 거의 0 |

- W_h의 가장 큰 고유값 크기를 0.7과 1.0으로 맞춰 비교했다. 0.7일 때 흔적이 더 빨리 사라졌다.
- 이유: 한 칸 지날 때마다 W_h를 곱하고 tanh를 거친다. 이 배율이 1보다 작으면 첫 단어의 영향이 길이에 따라 지수적으로 줄어든다.
- 한계: 무작위 가중치 실험이라 학습된 모델이 꼭 이렇다는 뜻은 아니다.

### 꼬리질문
#### LSTM을 쓰면 이 문제가 해결되나요?
cell state 덕분에 정보가 더 오래 남아서 완화되지만, 고정 크기 벡터 하나에 문장 전체를 담아야 한다는 병목은 그대로입니다. 그래서 Encoder의 모든 hidden state를 남겨 두고 다시 참고하는 Attention이 나왔습니다.

#### 배율을 1보다 크게 하면 흔적이 오래 남지 않나요?
배율이 1보다 크면 값이 커지다가 tanh에 포화되거나, 학습 중에 기울기 폭주가 생기기 쉽습니다. 그래서 배율 조절만으로는 긴 의존성을 안정적으로 다루기 어렵습니다.

## Q. Teacher Forcing이란 무엇이고, 단점과 보완 방법은 무엇인가요?
id: nlp-teacher-forcing
type: 개념
level: 심화
tags: Teacher Forcing, Seq2Seq, Exposure Bias
source: 4. Seq2Seq › Teacher Forcing

### 30초 답변
Teacher Forcing은 Decoder를 학습할 때 다음 시점 입력으로 직전 예측 대신 정답 단어를 넣는 방법입니다. 앞에서 틀려도 다음 입력은 정답이라 학습이 안정적이고 빠릅니다. 단점은 추론 때는 정답이 없어 자기 출력을 넣어야 하므로, 학습 때 겪어 보지 않은 자기 실수가 입력으로 들어오고 오류가 뒤로 갈수록 쌓인다는 점입니다. 이런 학습과 추론의 차이를 exposure bias라고 하고, 실습에서는 시점마다 50% 확률로 정답과 직전 출력을 섞어 완화했습니다.

### 자세히
| | 학습 (Teacher Forcing) | 추론 |
|---|---|---|
| 다음 입력 | 직전 시점의 정답 | 직전 시점의 자기 출력 |
| 앞에서 틀리면 | 다음 입력은 그래도 정답 | 틀린 단어가 다음 입력이 됨 |

```python
# 개념 코드: 시점마다 50% 확률로 정답, 나머지는 직전 출력
use_teacher = random.random() < 0.5
next_input = target[t] if use_teacher else output.argmax(-1)
```

- 보충: Scheduled Sampling(Bengio et al., 2015)은 학습 초반에는 정답 비율을 높게 두고, 학습이 진행될수록 자기 출력 비율을 점점 늘린다.

### 꼬리질문
#### Transformer Decoder도 Teacher Forcing을 쓰나요?
네. 학습할 때 정답 문장 전체를 한 칸 밀어 한꺼번에 넣고, Masked Attention으로 뒤 단어를 못 보게 해서 모든 위치를 병렬로 학습합니다. 정답을 입력으로 쓰므로 이것도 Teacher Forcing입니다.

#### Teacher Forcing 비율을 0으로 하면 어떻게 되나요?
학습 초반에는 출력이 거의 무작위라서 엉뚱한 입력을 받으며 학습하게 됩니다. 그래서 수렴이 매우 느리고 불안정해집니다.

## Q. Attention 메커니즘에 대해 설명해 보세요.
id: nlp-attention
type: 개념
level: 기본
tags: Attention, Seq2Seq, 정렬
source: 5. Attention

### 30초 답변
Attention은 Decoder가 단어를 하나 출력할 때마다 Encoder의 모든 hidden state를 다시 보고, 지금 필요한 입력에 더 큰 비중을 두어 섞는 방법입니다. 사람이 번역할 때 문장을 통째로 외우지 않고 원문의 해당 부분을 다시 확인하는 것과 같습니다. 계산은 세 단계로, Decoder 상태와 각 입력 hidden state를 내적해 점수를 구하고, Softmax로 합이 1인 비중을 만든 뒤, 그 비중으로 입력을 가중합해 이번 시점의 context를 만듭니다.

### 자세히
```python
# enc_out: (src_len, hidden), dec_h: (hidden,)
scores = enc_out @ dec_h                  # ① 점수
weights = torch.softmax(scores, dim=0)    # ② 비중 (합 1)
z = weights @ enc_out                     # ③ 가중합
```

- Seq2Seq은 마지막 h 하나만 넘겼지만, Attention은 Encoder hidden state h₁~h_T를 전부 남겨 두고 출력 시점마다 새 context를 만든다.
- 설명용 예시 값: "나는"을 출력할 때 "i"의 점수가 2.90으로 가장 높아 Softmax 뒤 비중이 0.81이 되고, z도 거의 "i"의 벡터를 닮는다.
- 출력 단어별 비중을 표로 그리면 어떤 출력이 어떤 입력을 봤는지 보인다. 영어와 한국어는 어순이 달라 대각선이 아니라 엇갈린 모양이 나온다.

### 꼬리질문
#### Attention 결과는 Query 자체와 무엇이 다른가요?
결과는 Query가 아니라 Value들을 비중대로 섞은 새 벡터입니다. Query는 어디를 볼지 정하는 데만 쓰입니다.

#### Bahdanau Attention과 Luong Attention은 어떻게 다른가요?
Bahdanau(2015)는 Decoder 상태와 Encoder 상태를 작은 신경망(tanh 층)에 넣어 점수를 내는 덧셈 방식이고, Luong(2015)은 내적처럼 곱셈으로 점수를 냅니다. 위 코드의 `enc_out @ dec_h`는 내적 방식이고, Transformer도 내적 방식을 씁니다.

## Q. Attention의 Query, Key, Value는 각각 무엇이고, Self-Attention에서는 어떻게 만드나요?
id: nlp-self-attention
type: 개념
level: 기본
tags: Self-Attention, Query-Key-Value, Transformer
source: 6. Transformer › Self-Attention

### 30초 답변
Query는 지금 정보를 찾는 쪽의 질문, Key는 각 단어가 가진 이름표로 Query와 비교되는 대상, Value는 실제로 가져갈 내용입니다. Query와 Key를 내적해 관련도 점수를 내고, Softmax로 만든 비중으로 Value를 가중합합니다. Self-Attention에서는 같은 문장의 단어 벡터 X에 서로 다른 학습 가중치 W_Q, W_K, W_V를 곱해 세 가지를 만듭니다.

### 자세히
```python
Q, K, V = X @ W_Q, X @ W_K, X @ W_V      # X: (n, d_model)
scores = Q @ K.T / math.sqrt(d_k)        # (n, n)
alpha = torch.softmax(scores, dim=-1)    # 행마다 합 1
out = alpha @ V                          # (n, d_v)
```

- 수식: `Attention(Q, K, V) = softmax(QKᵀ / √d_k) V`
- 검색에 비유하면 Query는 검색어, Key는 문서 제목, Value는 문서 내용이다.
- 점수 표의 행은 Query, 열은 Key다. "the cat sat on the mat"으로 계산하면 행마다 합이 1이고 단어마다 보는 곳이 달랐다. 무작위 가중치라 패턴 자체에 의미는 없다.
- Seq2Seq Attention에서는 Key와 Value가 같은 Encoder hidden state였다. Transformer는 W_K, W_V로 따로 만든다.

### 꼬리질문
#### Q와 K를 같은 가중치로 만들면 안 되나요?
같으면 점수 표가 대칭이 되어, A가 B를 보는 정도와 B가 A를 보는 정도가 항상 같아집니다. 또 자기 자신과의 내적이 커지기 쉬워 자기만 보는 쪽으로 쏠립니다. 따로 두면 찾는 역할과 찾아지는 역할을 분리해서 배울 수 있습니다.

#### Key와 Value를 따로 두는 이유는 무엇인가요?
어떤 단어를 볼지 정하는 기준과, 보기로 한 단어에서 가져올 정보가 꼭 같을 필요가 없기 때문입니다. 나누어 두면 비교하기 좋은 표현과 전달하기 좋은 표현을 따로 학습할 수 있습니다.

## Q. Seq2Seq의 Attention과 Self-Attention은 어떻게 다른가요?
id: nlp-seq2seq-attn-vs-self-attn
type: 비교
level: 심화
tags: Attention, Self-Attention, Seq2Seq
source: 6. Transformer › Self-Attention

### 30초 답변
Seq2Seq Attention은 Decoder의 현재 상태가 Query이고 Encoder의 hidden state들이 Key와 Value라서, 두 문장 사이에서 출력 단어와 입력 단어의 대응을 찾습니다. Self-Attention은 Q, K, V를 모두 같은 문장에서 만들어서, 한 문장 안의 단어끼리 서로 얼마나 관련 있는지를 봅니다. 또 Seq2Seq Attention은 RNN 위에 얹혀 Decoder 시점마다 순서대로 계산되지만, Self-Attention은 모든 단어 쌍을 행렬 곱 한 번으로 동시에 계산합니다.

### 자세히
| | Seq2Seq Attention | Self-Attention |
|---|---|---|
| Query | Decoder hidden state | 같은 문장의 단어 (X W_Q) |
| Key / Value | Encoder hidden state (K = V) | 같은 문장의 단어 (X W_K, X W_V) |
| 보는 관계 | 출력 단어 ↔ 입력 단어 | 문장 안 단어 ↔ 단어 |
| 위치 | RNN Encoder와 Decoder 사이 | Transformer의 각 층 |
| 계산 | Decoder 시점마다 순서대로 | 모든 위치를 한 번에 |

- Self-Attention 예: "The animal didn't cross the street because it was too tired"에서 it이 animal을 많이 보도록 학습될 수 있다.
- Transformer 안에서 Seq2Seq Attention과 같은 역할을 하는 것은 Decoder의 Encoder-Decoder Attention이다.

### 꼬리질문
#### Self-Attention에서 단어는 자기 자신도 보나요?
네, 점수 표의 대각선이 자기 자신과의 점수라서 자기 정보도 비중대로 섞입니다. 거기에 잔차 연결이 원래 입력을 다시 더해 주기 때문에 자기 정보가 사라지지 않습니다.

## Q. Transformer의 전체 구조를 설명해 보세요.
id: nlp-transformer-architecture
type: 개념
level: 기본
tags: Transformer, Encoder-Decoder, 구조
source: 6. Transformer › Encoder와 Decoder 한 층

### 30초 답변
Transformer는 RNN 없이 Attention만으로 만든 Encoder-Decoder 모델입니다. 입력 토큰을 임베딩하고 Positional Encoding을 더한 뒤, Self-Attention과 Feed Forward로 이루어진 Encoder 층을 N개 통과시킵니다. Decoder는 지금까지의 출력 토큰을 받아 Masked Self-Attention, Encoder 출력을 참고하는 Encoder-Decoder Attention, Feed Forward 순서의 층을 N개 지나고, 마지막에 Linear와 Softmax로 다음 단어 확률을 냅니다. 각 서브층 뒤에는 잔차 연결과 정규화(Add & Norm)가 붙습니다.

### 자세히
```
입력 토큰 → Embedding + Positional Encoding
  Encoder × N : Multi-Head Self-Attention → Add & Norm → Feed Forward → Add & Norm
출력 토큰(한 칸 밀림) → Embedding + Positional Encoding
  Decoder × N : Masked Multi-Head Self-Attention → Add & Norm
                → Encoder-Decoder Attention (K, V = Encoder 출력) → Add & Norm
                → Feed Forward → Add & Norm
→ Linear (d_model → 어휘 크기) → Softmax → 다음 단어 확률
```

| 원 논문 base 설정 | 값 |
|---|---|
| 층 수 N | Encoder 6, Decoder 6 |
| d_model | 512 |
| head 수 | 8 (head당 64차원) |
| d_ff | 2048 |

### 꼬리질문
#### 그림에서 Decoder 입력이 "Outputs (shifted right)"인 이유는 무엇인가요?
학습 때 Decoder 입력은 정답 문장을 한 칸 오른쪽으로 밀고 맨 앞에 시작 토큰을 붙인 것입니다. 그래야 위치 t에서 t-1번째까지의 정답만 보고 t번째 단어를 예측하게 됩니다.

#### 임베딩과 마지막 Linear 층은 따로 학습하나요?
원 논문은 입력 임베딩, 출력 임베딩, Softmax 직전 Linear 층이 같은 가중치 행렬을 공유했습니다. 그리고 임베딩 값에는 √d_model을 곱해서 썼습니다.

## Q. Transformer가 RNN 기반 모델보다 나은 점은 무엇인가요?
id: nlp-transformer-vs-rnn
type: 비교
level: 기본
tags: Transformer, RNN, 병렬화
source: 6. Transformer › RNN을 아예 빼 버리자

### 30초 답변
두 가지입니다. 첫째, 병렬 계산입니다. RNN은 단어 50개면 50단계를 순서대로 기다려야 하지만 Transformer는 한 층에서 모든 단어를 동시에 계산해서 GPU를 잘 활용하고, 덕분에 모델과 데이터를 크게 키울 수 있었습니다. 둘째, 먼 단어 사이의 경로입니다. RNN은 첫 단어 정보가 사이 시점을 전부 거쳐야 닿지만, Self-Attention은 모든 단어 쌍이 점수 하나로 바로 연결됩니다.

### 자세히
원 논문의 비교 (n: 문장 길이, d: 표현 차원)

| | RNN | Self-Attention |
|---|---|---|
| 층당 계산량 | O(n·d²) | O(n²·d) |
| 순서대로 해야 하는 연산 | O(n) | O(1) |
| 먼 두 단어 사이 경로 길이 | O(n) | O(1) |

- 병렬화 덕분에 모델과 데이터 규모를 크게 키울 수 있었고, 이것이 LLM의 토대가 됐다.
- 경로가 짧으면 기울기도 사이 시점을 전부 거치지 않으므로 먼 단어 관계를 배우기 쉽다.

### 꼬리질문
#### RNN이 Transformer보다 나은 점은 없나요?
생성할 때 RNN은 hidden state 하나만 들고 다음 단어를 만들어서, 길이와 상관없이 한 단계의 비용과 메모리가 일정합니다. Transformer는 앞 토큰 전부와 Attention을 계산해야 해서 문장이 길수록 한 단계가 무거워집니다. 또 RNN은 순서 정보가 구조에 들어 있어 위치 인코딩이 필요 없습니다.

#### Transformer도 문장을 생성할 때는 순차적이지 않나요?
맞습니다. 생성할 때는 단어를 하나씩 만들어야 해서 Decoder는 순서대로 돕니다. 병렬화의 이점은 정답 문장 전체를 한꺼번에 넣을 수 있는 학습 단계와, 입력 문장을 읽는 Encoder에서 크게 나타납니다.

## Q. Attention 점수를 √d_k로 나누는 이유는 무엇인가요?
id: nlp-scaled-dot-product
type: 이유
level: 심화
tags: Scaled Dot-Product, Softmax, Self-Attention
source: 6. Transformer › Self-Attention

### 30초 답변
Q와 K의 내적은 d_k개 항의 합이라 차원이 커질수록 값의 크기도 커집니다. 점수가 크면 Softmax가 가장 큰 값 하나에 거의 1을 몰아주고 나머지는 0에 가까워지는데, 이 영역에서는 Softmax의 기울기가 거의 0이라 학습이 잘 되지 않습니다. 그래서 √d_k로 나눠 점수의 분산을 1 근처로 맞춥니다.

### 자세히
```
q, k의 각 성분이 평균 0, 분산 1인 독립 값이면
q · k = Σ qᵢkᵢ  (d_k개 항)  →  평균 0, 분산 d_k
(q · k) / √d_k               →  분산 1
```

- 예: 점수 [2, 1]의 Softmax는 [0.73, 0.27]인데, 10배한 [20, 10]은 [0.99995, 0.00005]다.
- 원 논문 base는 d_k = 64라서 점수를 8로 나눈다.
- 그래서 이름이 Scaled Dot-Product Attention이다.

### 꼬리질문
#### Softmax가 한쪽으로 쏠리면 왜 기울기가 작아지나요?
Softmax의 미분은 sᵢ(1 − sᵢ)와 −sᵢsⱼ 꼴인데, 출력이 1과 0으로 쏠리면 이 값들이 모두 0에 가까워집니다. 그래서 점수를 조금 바꿔도 출력이 거의 변하지 않고 기울기가 전달되지 않습니다.

#### 스케일링을 하지 않으면 실제로 성능 차이가 나나요?
원 논문에 따르면 d_k가 작을 때는 덧셈 방식 Attention과 내적 방식이 비슷했지만, d_k가 크면 스케일링하지 않은 내적 방식이 덧셈 방식보다 성능이 떨어졌습니다. 저자들은 그 원인을 내적 값이 커져 Softmax 기울기가 작아지는 데서 찾았습니다.

## Q. Transformer에 Positional Encoding이 필요한 이유는 무엇인가요?
id: nlp-positional-encoding
type: 이유
level: 기본
tags: Positional Encoding, Transformer, Self-Attention
source: 6. Transformer › Positional Encoding

### 30초 답변
Self-Attention은 단어 순서를 구분하는 장치가 없어서, 단어 순서를 섞어도 각 단어의 출력이 그대로입니다. 실제로 위치 정보 없이 "the cat sat on the mat"을 계산하면 첫 번째 the와 다섯 번째 the의 출력이 완전히 같습니다. 그래서 위치마다 다른 벡터를 단어 벡터에 더해 순서를 알려 줍니다.

### 자세히
- 실험: 위치 정보가 없으면 두 the는 입력 벡터도, Q·K·V도, 다른 단어를 보는 비중도 같아서 출력이 똑같았다.
- 단어 순서를 섞으면 출력도 같은 방식으로 섞일 뿐, 각 단어의 결과 값은 그대로다. Self-Attention은 입력을 순서 없는 집합처럼 다룬다.
- RNN은 한 칸씩 읽는 구조 자체에 순서가 들어 있지만, Transformer는 모든 단어를 한꺼번에 넣는다.
- 해결: `입력 = 단어 임베딩 + 위치 벡터`. 위치 정보를 더하자 두 the의 출력이 서로 달라졌다.

### 꼬리질문
#### 위치 벡터를 이어 붙이지 않고 더하는 이유는 무엇인가요?
차원을 늘리지 않고도 내용과 위치 정보를 한 벡터에 담을 수 있기 때문입니다. 차원이 충분히 크면 두 정보가 서로 크게 간섭하지 않고, 이후 층의 선형 변환이 필요한 정보를 분리해 쓸 수 있습니다.

#### 위치 정보가 없으면 "dog bites man"과 "man bites dog"를 구분할 수 있나요?
구분하지 못합니다. 두 문장은 토큰 구성이 같아서 각 단어의 출력 벡터가 똑같이 나오고 순서만 바뀝니다. 출력을 평균 내서 분류한다면 두 문장은 완전히 같은 입력이 됩니다.

## Q. sin·cos 위치 인코딩은 어떤 특징이 있고, 학습형 위치 임베딩이나 RoPE 같은 상대 위치 방식과는 어떻게 다른가요?
id: nlp-pe-variants
type: 개념
level: 심화
tags: Positional Encoding, RoPE, 상대 위치
source: 6. Transformer › Positional Encoding

### 30초 답변
원 논문의 위치 인코딩은 차원마다 주파수가 다른 sin과 cos 값으로 위치마다 고유한 무늬를 만드는 고정된 벡터입니다. 학습할 파라미터가 없고 공식으로 어떤 길이든 계산할 수 있으며, 위치를 k만큼 옮기는 것이 선형 변환으로 표현되는 성질이 있습니다. BERT와 GPT-2는 위치마다 벡터를 학습하는 방식을 썼고, 최근 LLM은 두 토큰의 상대 거리를 Attention 점수에 직접 반영하는 RoPE 같은 방식을 많이 씁니다.

### 자세히
```
PE(pos, 2i)   = sin(pos / 10000^(2i/d))
PE(pos, 2i+1) = cos(pos / 10000^(2i/d))
```

- 앞쪽 칸은 빠르게, 뒤쪽 칸은 천천히 진동한다. 초침, 분침, 시침이 함께 돌며 시각을 나타내는 것과 비슷하다.

| 방식 | 예 | 특징 |
|---|---|---|
| 고정 sin·cos | 원 Transformer | 파라미터 없음, 임의 길이 계산 가능 |
| 학습형 절대 위치 | BERT, GPT-2 | 위치별 벡터 학습, 학습한 최대 길이(BERT 512)를 넘기 어려움 |
| 상대 위치 (보충) | RoPE (LLaMA 등) | Q, K를 위치만큼 회전, 점수가 상대 거리에 의존 |

- 원 논문은 학습형 위치 임베딩도 실험했는데 결과가 거의 같아서, 학습 때보다 긴 문장에도 쓸 수 있을 것으로 본 sin·cos를 택했다.

### 꼬리질문
#### sin·cos를 같이 쓰면 상대 위치를 어떻게 표현할 수 있나요?
같은 주파수의 sin·cos 쌍은 위치가 k만큼 이동하면 2차원 회전 행렬을 곱한 것과 같습니다. 그래서 PE(pos + k)를 PE(pos)의 선형 변환으로 나타낼 수 있고, 모델이 상대 위치를 배우기 쉬울 것이라는 것이 원 논문의 설명입니다.

#### RoPE는 왜 위치를 입력에 더하지 않고 Q, K에 적용하나요?
Attention 점수는 Q와 K의 내적으로 정해지므로, 둘을 각자의 위치 각도만큼 회전시키면 내적 결과가 두 위치의 차이에만 의존하게 됩니다. 이렇게 하면 단어 사이의 상대 거리가 점수 계산에 바로 들어갑니다.

## Q. Multi-Head Attention은 무엇이고 왜 쓰나요?
id: nlp-multi-head-attention
type: 개념
level: 기본
tags: Multi-Head Attention, Self-Attention, Transformer
source: 6. Transformer › Self-Attention

### 30초 답변
Multi-Head Attention은 Self-Attention을 서로 다른 가중치로 여러 번 나누어 하고, 결과를 이어 붙인 뒤 선형 변환하는 방법입니다. head마다 W_Q, W_K, W_V가 달라서, 어떤 head는 문법 관계를, 다른 head는 의미나 먼 단어 관계를 보는 식으로 여러 관점을 동시에 볼 수 있습니다. 원 논문은 d_model 512를 head 8개로 나눠 head당 64차원을 쓰므로, 전체 계산량은 512차원 head 하나로 할 때와 비슷합니다.

### 자세히
```
head_i = Attention(X W_Qᵢ, X W_Kᵢ, X W_Vᵢ)        # i = 1..h
MultiHead(X) = Concat(head_1, ..., head_h) W_O
d_k = d_v = d_model / h                           # 512 / 8 = 64
```

- head 하나는 Softmax 한 번으로 비중을 정하므로, 여러 곳을 봐야 할 때 정보가 평균되어 버린다. head가 여럿이면 서로 다른 단어를 동시에 볼 수 있다.
- W_O는 이어 붙인 head 결과를 다시 섞어 d_model 차원으로 돌려놓는다.
- 구현할 때는 head마다 따로 곱하지 않고 큰 행렬 하나로 계산한 뒤 `(batch, h, n, d_k)` 모양으로 나눈다.

### 꼬리질문
#### head 수를 늘리면 항상 좋아지나요?
아닙니다. d_model이 고정이면 head가 많을수록 head당 차원이 작아집니다. 원 논문 실험에서도 head 1개는 8개보다 성능이 낮았고, head를 너무 많이 늘려도 성능이 떨어졌습니다.

#### head마다 정말 다른 것을 보나요?
학습된 모델의 head별 Attention을 그려 보면 바로 앞 단어를 보는 head, 대명사가 가리키는 명사를 보는 head처럼 역할이 갈리는 경우가 보고되었습니다. 다만 모든 head가 뚜렷한 역할을 갖지는 않고, 일부 head는 지워도 성능이 거의 그대로라는 연구도 있습니다.

## Q. Decoder의 Masked Self-Attention은 어떻게 동작하나요?
id: nlp-masked-attention
type: 개념
level: 심화
tags: Masked Attention, Decoder, Transformer
source: 6. Transformer › Masked Attention

### 30초 답변
Decoder는 단어를 하나씩 만들어 가는 모델인데, 학습할 때는 정답 문장 전체를 한꺼번에 넣기 때문에 아직 만들지 않은 뒤 단어를 미리 볼 수 있습니다. 이를 막기 위해 현재 위치보다 뒤에 있는 단어의 점수를 Softmax 전에 -∞로 바꿉니다. 그러면 그 위치의 가중치가 0이 되어 각 단어는 자기 자신과 앞 단어만 보게 되고, 학습과 생성의 조건이 같아집니다.

### 자세히
```python
n = scores.size(-1)
mask = torch.triu(torch.ones(n, n, dtype=torch.bool), diagonal=1)  # 대각선 위쪽(미래) True
scores = scores.masked_fill(mask, float('-inf'))
alpha = torch.softmax(scores, dim=-1)       # 미래 위치는 0, 아래 삼각 행렬 모양
```

- 예시 계산: 첫 단어 the는 볼 수 있는 것이 자기뿐이라 가중치가 1.00이 됐다. 마지막 단어 mat은 원래 모든 단어를 볼 수 있는 위치라 마스크가 있든 없든 값이 같았다.
- Softmax 뒤에 0을 곱하지 않고 Softmax 전에 -∞로 바꾸는 이유: 그래야 남은 가중치의 합이 다시 1이 된다.

### 꼬리질문
#### 마스크가 없으면 어떤 문제가 생기나요?
학습 때 모델이 다음 단어를 입력에서 그대로 베껴 오는 쪽으로 배워서 손실은 쉽게 내려가지만, 실제 생성 때는 뒤 단어가 없으니 제대로 동작하지 않습니다.

#### 패딩 마스크와는 어떻게 다른가요?
패딩 마스크는 `<PAD>` 위치를 가려서 Encoder와 Decoder 모두에서 쓰고, 미래를 가리는 마스크는 Decoder의 Self-Attention에서만 씁니다. Decoder에서는 두 마스크를 합쳐서 적용합니다.

## Q. Encoder-Decoder(Cross) Attention에서 Q, K, V는 각각 어디서 오나요?
id: nlp-cross-attention
type: 개념
level: 심화
tags: Cross Attention, Encoder-Decoder, Transformer
source: 6. Transformer › Encoder와 Decoder 한 층

### 30초 답변
Query는 Decoder 쪽, 정확히는 바로 앞의 Masked Self-Attention을 지난 Decoder 표현에서 오고, Key와 Value는 Encoder 마지막 층의 출력에서 옵니다. 그래서 Decoder의 각 위치가 지금 만들 단어에 입력 문장의 어느 부분이 필요한지 묻고, 입력 문장 전체에서 필요한 정보를 가져옵니다. Seq2Seq에 얹었던 Attention과 같은 역할을 Transformer 안에서 하는 층입니다.

### 자세히
```
Q = Decoder 표현 (tgt_len, d_model) · W_Q
K = Encoder 출력 (src_len, d_model) · W_K
V = Encoder 출력 (src_len, d_model) · W_V
가중치 표: (tgt_len, src_len)  →  출력: (tgt_len, d_model)
```

- 출력 길이는 Query 쪽(Decoder) 길이를 따른다. 그래서 입력과 출력 문장 길이가 달라도 된다.
- Encoder 마지막 층 출력 하나가 Decoder 모든 층의 Cross Attention에 똑같이 들어간다.
- 입력 문장은 이미 전부 주어져 있으므로 미래를 가리는 마스크는 필요 없고, 입력 쪽 패딩 마스크만 쓴다.

### 꼬리질문
#### 생성할 때 Encoder는 매번 다시 계산하나요?
아닙니다. 입력 문장은 바뀌지 않으므로 Encoder는 한 번만 돌리고, 그 출력을 Decoder가 단어를 만들 때마다 재사용합니다.

#### BERT나 GPT에도 Cross Attention이 있나요?
없습니다. BERT는 Encoder만, GPT는 Decoder에서 Cross Attention을 뺀 블록만 쌓았습니다. Cross Attention은 원래 Transformer나 T5처럼 Encoder와 Decoder가 모두 있는 모델에 있습니다.

## Q. Add & Norm에서 잔차 연결과 Layer Normalization은 각각 무슨 역할을 하나요?
id: nlp-add-norm
type: 개념
level: 심화
tags: Residual Connection, LayerNorm, Pre-LN, Transformer
source: 6. Transformer › Encoder와 Decoder 한 층

### 30초 답변
Add & Norm은 서브층의 입력을 출력에 더하는 잔차 연결(Add)과, 그 결과를 Layer Normalization으로 정규화하는 부분(Norm)입니다. 잔차 연결은 기울기가 서브층을 건너뛰는 지름길로 바로 흐르게 해서 층을 깊게 쌓아도 학습이 되게 하고, LayerNorm은 층마다 값의 범위를 일정하게 맞춰 학습을 안정시킵니다. 원 논문은 더한 뒤 정규화하는 Post-LN이었지만, 지금은 깊은 모델에서 더 안정적인 Pre-LN을 많이 씁니다.

### 자세히
```
Post-LN (원 논문):       x = LayerNorm(x + Sublayer(x))
Pre-LN (GPT-2 이후 다수): x = x + Sublayer(LayerNorm(x))
```

- 잔차 연결의 미분은 `∂(x + F(x))/∂x = 1 + ∂F/∂x`. F의 기울기가 작아도 1이 남아 앞 층으로 신호가 전달된다.
- 서브층은 입력 전체를 다시 만들 필요 없이 입력에 더할 변화량만 배우면 된다.
- 보충: Post-LN은 깊어지면 학습 초반이 불안정해 learning rate warmup이 중요했고, Pre-LN은 이 문제가 덜하다. 최신 LLM(LLaMA 등)은 평균 빼기를 생략한 RMSNorm을 Pre-Norm 위치에 쓴다.

### 꼬리질문
#### 잔차 연결을 하려면 어떤 조건이 필요한가요?
입력과 서브층 출력의 모양이 같아야 더할 수 있습니다. 그래서 Transformer는 모든 서브층의 출력 차원을 d_model(원 논문 512)로 통일합니다.

#### Pre-LN을 쓰면 마지막에 따로 해 줄 것이 있나요?
마지막 블록의 출력은 정규화를 거치지 않은 상태라서, 보통 맨 끝에 LayerNorm을 한 번 더 둡니다. GPT-2도 마지막 블록 뒤에 LayerNorm을 추가했습니다.

## Q. Transformer는 왜 BatchNorm 대신 LayerNorm을 쓰나요?
id: nlp-layernorm-vs-batchnorm
type: 비교
level: 심화
tags: LayerNorm, BatchNorm, 정규화, Transformer
source: 보충 (블로그 밖 내용)

### 30초 답변
BatchNorm은 같은 특성에 대해 배치 안의 여러 샘플에 걸쳐 평균과 분산을 구하고, LayerNorm은 샘플 하나, 문장이라면 토큰 하나의 모든 특성에 대해 평균과 분산을 구합니다. 문장은 길이가 제각각이라 패딩이 섞이고, 위치마다 안정적인 배치 통계를 내기 어렵고, 큰 모델은 배치를 크게 잡기도 어렵습니다. LayerNorm은 토큰마다 독립적으로 정규화하므로 배치 크기, 문장 길이, 패딩의 영향을 받지 않고 학습과 추론에서 똑같이 동작합니다.

### 자세히
| | BatchNorm | LayerNorm |
|---|---|---|
| 평균·분산을 구하는 범위 | 배치 전체 × 같은 특성 | 토큰 하나 × 모든 특성 (d_model) |
| 배치 크기 영향 | 작으면 통계가 불안정 | 없음 |
| train / eval 차이 | 있음 (running 통계) | 없음 |
| 주 사용처 | CNN 이미지 분류 | Transformer, RNN, LLM |

```python
# x: (batch, seq_len, d_model)
ln = nn.LayerNorm(d_model)   # 마지막 차원으로 평균·분산, 토큰마다 따로
```

- BatchNorm을 문장에 쓰면 `<PAD>` 위치의 값도 통계에 섞인다.
- 생성할 때는 토큰을 하나씩 넣는데, LayerNorm은 배치 1개, 토큰 1개로도 학습 때와 같은 계산을 한다.

### 꼬리질문
#### LayerNorm에도 학습 파라미터가 있나요?
네, BatchNorm처럼 정규화한 뒤 곱하는 γ와 더하는 β가 특성 차원마다 있습니다. 범위는 맞추되, 필요하면 모델이 원래 크기와 위치를 다시 찾을 수 있게 하는 장치입니다.

#### CNN에서는 왜 여전히 BatchNorm을 많이 쓰나요?
이미지는 크기를 맞춰 배치로 묶기 쉽고, 채널마다 이미지의 모든 위치에서 통계를 모으므로 값이 안정적입니다. ResNet 같은 CNN에서 BatchNorm이 학습을 크게 안정시켜 표준이 됐습니다.

## Q. Transformer의 Feed Forward Network는 무슨 역할을 하나요?
id: nlp-ffn
type: 개념
level: 심화
tags: Feed Forward, Transformer, Position-wise
source: 6. Transformer › Encoder와 Decoder 한 층

### 30초 답변
Feed Forward Network는 Attention 다음에 오는 두 층짜리 MLP로, 각 토큰 위치에 같은 가중치를 따로따로 적용합니다. Attention이 다른 토큰의 정보를 섞어 오는 역할이라면, FFN은 섞어 온 정보를 토큰마다 비선형 변환해서 표현을 가공하는 역할입니다. 원 논문은 512차원을 2048차원으로 넓혔다가 ReLU를 거쳐 다시 512차원으로 줄입니다.

### 자세히
```
FFN(x) = max(0, x W₁ + b₁) W₂ + b₂
W₁: (512, 2048),  W₂: (2048, 512)      # d_ff = 4 × d_model
```

- position-wise: 위치끼리는 섞지 않는다. 같은 층에서는 모든 위치가 같은 W₁, W₂를 쓰고, 층이 다르면 가중치도 다르다.
- Attention 출력은 Value에 대해서는 선형 결합이다. FFN의 활성화 함수가 토큰마다 비선형성을 더한다.
- 파라미터 수: 한 층에서 FFN이 약 2 × 512 × 2048 ≈ 2.1M, Attention(W_Q, W_K, W_V, W_O)이 약 4 × 512 × 512 ≈ 1.05M로 FFN이 약 2/3를 차지한다.
- 보충: BERT와 GPT는 ReLU 대신 GELU를 쓴다.

### 꼬리질문
#### 위치마다 따로 적용하는 FFN은 1×1 Convolution과 같나요?
네, 원 논문도 커널 크기 1인 Convolution 두 개로 볼 수 있다고 설명합니다. 모든 위치에 같은 가중치를 쓰고 이웃 위치는 섞지 않는다는 점이 같습니다.

#### FFN이 하는 일을 더 구체적으로 볼 수 있나요?
FFN의 첫 층을 key, 둘째 층을 value로 보고, 입력 패턴에 반응해 관련 정보를 꺼내는 키-값 메모리처럼 동작한다는 분석이 있습니다. 사전학습 모델의 사실 지식이 상당 부분 FFN에 저장된다는 연구도 이 관점에서 나왔습니다.

## Q. Self-Attention의 계산 복잡도는 어떻게 되고, 긴 문장에서는 어떤 한계가 있나요?
id: nlp-attention-complexity
type: 이유
level: 심화
tags: Self-Attention, 계산 복잡도, 긴 문맥
source: 보충 (블로그 밖 내용)

### 30초 답변
Self-Attention은 모든 토큰 쌍의 점수를 계산하므로 길이 n인 문장에서 n×n 점수 표가 생기고, 계산량은 O(n²·d), 점수 표 메모리는 O(n²)에 비례합니다. 문장 길이가 2배가 되면 점수 표는 4배가 됩니다. 그래서 긴 문서를 다루면 계산과 GPU 메모리가 빠르게 늘어나고, 모델이 한 번에 다룰 수 있는 최대 길이가 제한됩니다.

### 자세히
```
Q Kᵀ : (n, d) × (d, n) → (n, n)     # 약 n²·d 곱셈
α V  : (n, n) × (n, d) → (n, d)     # 약 n²·d 곱셈
```

| n | 점수 표 원소 수 (head 하나) |
|---|---|
| 512 | 약 26만 |
| 4096 | 약 1,678만 |

- RNN은 층당 O(n·d²)라서, 문장이 d보다 짧으면 Self-Attention이 오히려 계산이 적다. 원 논문도 이 점을 짚었다.
- 개선 방향: 일부 위치만 보는 sparse·sliding window Attention(Longformer 등), 메모리 접근을 줄여 같은 계산을 빠르게 하는 FlashAttention, 생성할 때 앞 토큰의 K, V를 저장해 다시 쓰는 KV cache.

### 꼬리질문
#### FlashAttention은 계산 복잡도를 줄이나요?
아닙니다. 계산량은 그대로 O(n²·d)이지만, n×n 점수 표를 GPU 메모리에 통째로 만들지 않고 블록 단위로 나눠 계산해서 메모리 사용을 길이에 선형으로 줄이고 속도를 높입니다.

#### 생성할 때 KV cache를 쓰면 무엇이 줄어드나요?
매 단계 앞 토큰들의 K, V를 다시 계산하지 않고 저장해 둔 것을 쓰므로, 새 토큰 하나의 Q, K, V만 계산하면 됩니다. 대신 저장할 K, V가 문장 길이에 비례해 메모리를 차지합니다.

## Q. BERT와 GPT의 차이를 설명해 보세요.
id: nlp-bert-vs-gpt
type: 비교
level: 기본
tags: BERT, GPT, 사전학습
source: 7. BERT와 GPT › 구조와 읽는 방향

### 30초 답변
둘 다 Transformer의 한쪽만 쌓은 사전학습 모델입니다. BERT는 Encoder만 쌓아서 문장 전체를 양방향으로 보고, 가린 단어를 맞히는 MLM으로 학습해 분류나 검색 같은 이해(NLU) 작업에 강합니다. GPT는 Decoder만 쌓아서 Masked Attention으로 앞 단어까지만 보고, 다음 단어를 예측하며 학습해 문장 생성(NLG)에 강합니다. 둘 다 라벨 없는 대규모 텍스트로 사전학습한 뒤 작업에 맞게 활용한다는 점은 같습니다.

### 자세히
| | BERT (2018, Google) | GPT (2018, OpenAI) |
|---|---|---|
| 구조 | Transformer Encoder | Transformer Decoder |
| 보는 방향 | 양방향 | 왼쪽에서 오른쪽 (앞 단어만) |
| 사전학습 | MLM (+ NSP) | 다음 단어 예측 |
| 강점 | 이해: 분류, 질의응답, 검색 | 생성: 글쓰기, 대화 |
| 입력 형식 | `[CLS] 문장A [SEP] 문장B [SEP]` | 이어지는 토큰 |

- BERT는 2019년 10월부터 구글 검색에 들어가 검색어 전체의 문맥을 파악하는 데 쓰였다.
- GPT는 GPT-2, GPT-3, GPT-4로 이어지며 생성형 AI의 대표 구조가 됐다.

### 꼬리질문
#### T5 같은 Encoder-Decoder 모델은 어디에 해당하나요?
둘을 합친 원래 Transformer 구조입니다. 입력 전체를 양방향으로 읽는 Encoder와 출력을 생성하는 Decoder가 모두 있어서, 번역이나 요약처럼 입력을 이해해 다른 문장으로 바꾸는 작업에 잘 맞습니다.

#### BERT로는 문장을 생성할 수 없나요?
다음 단어를 하나씩 예측하도록 학습하지 않았고 문장 전체를 양방향으로 보기 때문에, GPT처럼 왼쪽에서 오른쪽으로 자연스럽게 이어 쓰기는 어렵습니다. 대신 빈칸을 채우거나 문장을 분류하는 데 강합니다.

## Q. BERT는 어떤 방법으로 사전학습하나요?
id: nlp-bert-pretraining
type: 개념
level: 기본
tags: BERT, MLM, NSP, 사전학습
source: 7. BERT와 GPT › BERT는 어떻게 학습할까

### 30초 답변
BERT는 라벨이 없는 대규모 텍스트로 두 가지 과제를 함께 학습합니다. MLM은 문장 속 토큰의 15%를 골라 가리고 양쪽 문맥으로 원래 단어를 맞히는 과제이고, NSP는 두 문장을 주고 문장 B가 A의 실제 다음 문장인지 맞히는 과제입니다. MLM으로 단어 수준의 양방향 문맥을, NSP로 문장 사이 관계를 배우게 하려는 설계였습니다.

### 자세히
| | MLM | NSP |
|---|---|---|
| 입력 | 15%를 가린 문장 | `[CLS] A [SEP] B [SEP]` |
| 맞히는 것 | 가린 자리의 원래 토큰 | B가 실제 다음 문장인가 (IsNext / NotNext) |
| 쓰는 출력 | 가린 위치의 출력 | `[CLS]` 위치의 출력 |
| 손실 | 고른 15% 자리에서만 | 문장 쌍마다 이진 분류 |

- NSP 데이터는 절반은 실제 다음 문장, 절반은 말뭉치에서 무작위로 고른 문장이다.
- 원 논문의 사전학습 데이터는 BooksCorpus와 영어 Wikipedia다.
- 보충: 이후 RoBERTa(2019)는 NSP를 빼고 MLM만 더 많은 데이터로 더 오래 학습해 성능을 높였다.

### 꼬리질문
#### NSP는 왜 이후 모델에서 빠졌나요?
무작위로 고른 문장은 주제부터 달라서, 모델이 문장 사이 논리보다 주제가 같은지만 보고도 쉽게 맞힐 수 있었습니다. RoBERTa에서 NSP를 빼도 성능이 떨어지지 않았고, ALBERT는 두 문장의 순서가 맞는지 맞히는 SOP로 바꿨습니다.

#### 양방향 모델에서 그냥 다음 단어를 예측하면 안 되나요?
양방향으로 보면 각 위치가 Self-Attention으로 정답 단어 자체를 볼 수 있어서, 예측할 것 없이 그대로 베끼게 됩니다. 그래서 정답 자리를 가려 놓고 맞히는 MLM을 씁니다.

## Q. BERT의 MLM에서 가린 단어를 전부 [MASK]로 바꾸지 않는 이유는 무엇인가요?
id: nlp-bert-mlm
type: 이유
level: 심화
tags: BERT, MLM, 사전학습
source: 7. BERT와 GPT › BERT는 어떻게 학습할까

### 30초 답변
BERT는 단어의 15%를 골라 그중 80%는 [MASK]로, 10%는 무작위 단어로 바꾸고, 10%는 그대로 둡니다. 실제로 모델을 쓸 때는 입력에 [MASK]가 나오지 않기 때문에, 전부 [MASK]로 가리면 학습과 사용 사이에 차이가 생깁니다. 일부를 무작위 단어나 원래 단어로 두면 모델이 [MASK] 자리뿐 아니라 모든 토큰의 표현을 신경 써서 만들게 됩니다.

### 자세히
```
전체 토큰 중 15%를 고름
 ├ 80% → [MASK]       : 15% × 80% = 12%
 ├ 10% → 무작위 단어   : 15% × 10% = 1.5%
 └ 10% → 그대로        : 15% × 10% = 1.5%
```

- 같은 규칙을 단어 10만 개에 적용한 시뮬레이션에서도 약 12%가 [MASK], 1.5% 안팎씩이 무작위 단어와 원래 단어 유지로 나왔다.
- 손실은 고른 15% 자리에서만 계산한다. 그대로 둔 10%도 정답을 맞혀야 하는 자리다.

### 꼬리질문
#### 15%만 가리면 학습 효율이 떨어지지 않나요?
맞습니다. 한 문장에서 15% 자리만 손실을 계산하니 모든 위치에서 다음 단어를 예측하는 GPT 방식보다 신호가 적습니다. 그래서 이후 ELECTRA처럼 모든 토큰에서 원래 토큰인지 바뀐 토큰인지를 판별하게 해 효율을 높인 방법이 나왔습니다.

#### 무작위 단어로 바꾼 자리는 오히려 학습을 방해하지 않나요?
전체 토큰의 1.5%뿐이라 언어 이해를 해칠 정도는 아니라고 원 논문은 설명합니다. 오히려 모델이 입력 토큰을 그대로 믿지 못하게 해서 모든 위치의 문맥 표현을 꼼꼼히 만들게 합니다.

## Q. BERT의 입력은 어떻게 구성되나요? [CLS], [SEP]와 세 가지 임베딩을 설명해 보세요.
id: nlp-bert-input
type: 개념
level: 기본
tags: BERT, CLS, Segment Embedding, 입력 표현
source: 7. BERT와 GPT › 구조와 읽는 방향

### 30초 답변
BERT 입력은 `[CLS] 문장 A [SEP] 문장 B [SEP]` 형태입니다. `[CLS]`는 문장 전체의 표현을 모으는 자리라서 분류할 때 이 위치의 출력을 쓰고, `[SEP]`는 문장 사이와 끝을 나누는 표시입니다. 각 토큰의 입력 벡터는 WordPiece 토큰 임베딩, 문장 A인지 B인지 나타내는 Segment 임베딩, 위치를 나타내는 Position 임베딩 세 개를 더해서 만듭니다.

### 자세히
```
토큰      [CLS]  my   dog  is   cute [SEP]  he   likes play ##ing [SEP]
Segment    A     A    A    A    A    A      B    B     B    B     B
Position   0     1    2    3    4    5      6    7     8    9     10
입력 벡터 = Token 임베딩 + Segment 임베딩 + Position 임베딩   (BERT-base 768차원)
```

- 보충: Position 임베딩은 sin·cos가 아니라 학습되는 벡터이고, 최대 512 위치까지 있다.
- 보충: Segment 임베딩은 A, B 두 종류뿐인 작은 표다. 문장이 하나면 전부 A다.
- `[CLS]`는 자기 단어 뜻이 없는 자리라서, Self-Attention으로 문장 전체 정보를 모으도록 NSP와 fine-tuning에서 학습된다.

### 꼬리질문
#### 사전학습만 한 BERT의 [CLS] 출력은 좋은 문장 표현인가요?
그렇지 않습니다. 사전학습에서는 NSP 과제로만 학습되므로, fine-tuning 없이 그대로 문장 유사도에 쓰면 성능이 낮은 것으로 알려져 있습니다. 문장 임베딩이 필요하면 Sentence-BERT처럼 따로 학습한 모델을 씁니다.

#### 질의응답에서는 입력을 어떻게 넣나요?
질문을 문장 A, 지문을 문장 B로 넣고, 지문의 토큰마다 정답의 시작 위치와 끝 위치일 확률을 출력해 정답 구간을 찾습니다.

## Q. GPT는 어떻게 학습하고, 어떻게 문장을 생성하나요?
id: nlp-gpt-generation
type: 개념
level: 기본
tags: GPT, 언어 모델, Autoregressive, 생성
source: 7. BERT와 GPT › 구조와 읽는 방향

### 30초 답변
GPT는 Transformer Decoder를 쌓은 모델로, 앞 단어들을 보고 다음 단어를 예측하는 과제로 라벨 없는 대규모 텍스트를 학습합니다. Masked Attention 덕분에 학습할 때는 문장 하나에서 모든 위치의 다음 단어 예측을 한꺼번에 계산할 수 있습니다. 생성할 때는 다음 단어를 하나 고르고 그 단어를 입력 뒤에 붙여 다시 넣는 과정을, 끝 토큰이 나오거나 정한 길이가 될 때까지 반복하는 autoregressive 방식으로 문장을 만듭니다.

### 자세히
```
학습:  입력  [나는, 오늘, 학교에]
       정답  [오늘, 학교에, 갔다]          # 한 칸씩 밀린 문장, 모든 위치에서 손실
       loss = −Σ log P(wₜ | w₁, …, wₜ₋₁)
생성:  "나는" → "나는 오늘" → "나는 오늘 학교에" → …
```

- 구조: Decoder에서 Cross Attention을 뺀 `Masked Self-Attention → FFN` 블록을 쌓는다.
- 흐름: GPT-1(2018)은 사전학습 후 작업별 fine-tuning, GPT-2(2019)는 fine-tuning 없이 프롬프트만으로 작업하는 zero-shot, GPT-3(2020)는 프롬프트에 예시 몇 개를 넣는 few-shot을 보여 줬다.

### 꼬리질문
#### 다음 단어는 어떻게 고르나요?
가장 확률 높은 단어를 고르는 greedy는 결정적이지만 같은 말을 반복하기 쉽습니다. 그래서 확률대로 뽑는 샘플링을 쓰고, temperature로 분포를 날카롭거나 평평하게 조절하며, 상위 k개(top-k)나 누적 확률 p까지(top-p)만 후보로 남깁니다. 번역처럼 정확한 문장이 필요하면 후보 문장 여러 개를 유지하며 진행하는 beam search도 씁니다.

#### GPT로 분류 같은 작업은 어떻게 하나요?
GPT-1은 입력 마지막 토큰의 출력에 선형층을 붙여 작업별로 fine-tuning했습니다. 이후 큰 GPT 모델은 "이 리뷰는 긍정인가요?" 같은 프롬프트를 주고 답을 생성하게 해서, 따로 학습하지 않고도 분류를 합니다.

## Q. 사전학습과 파인튜닝이란 무엇이고, 이 방식이 NLP를 어떻게 바꿨나요?
id: nlp-pretrain-finetune
type: 개념
level: 기본
tags: 사전학습, Fine-tuning, 전이학습, BERT, GPT
source: 7. BERT와 GPT › 사전학습과 fine-tuning

### 30초 답변
사전학습은 라벨 없는 대규모 텍스트로 MLM이나 다음 단어 예측 같은 과제를 풀게 해서 언어 자체를 먼저 크게 학습시키는 단계이고, 파인튜닝은 그 가중치에서 출발해 감성 분류나 질의응답 같은 작업의 라벨 데이터로 조금 더 학습시키는 단계입니다. 이전에는 작업마다 라벨 데이터를 많이 모아 처음부터 학습해야 했는데, 이제는 이미 언어를 아는 모델에서 출발하니 적은 라벨 데이터로도 높은 성능을 낼 수 있습니다. 사전학습 모델 하나를 여러 작업에 재사용하게 된 것이 가장 큰 변화입니다.

### 자세히
```
1단계 사전학습: 대규모 무라벨 텍스트 → MLM / 다음 단어 예측 → 범용 언어 모델
2단계 파인튜닝: 작업 라벨 데이터 → 출력층 추가 + 전체 가중치를 작은 학습률로 조금 더 학습
```

- 라벨이 필요 없는 이유: 정답(가린 단어, 다음 단어)을 텍스트 자체에서 만든다. 이런 방식을 자기지도학습이라고 한다.
- 처음 실습에서는 리뷰 120개로 임베딩을 무작위 값부터 학습시켰다. 파인튜닝은 이미 단어와 문맥 관계를 아는 상태에서 시작한다.
- BERT 파인튜닝 예: 문장 분류는 `[CLS]` 출력에 선형층, 개체명 인식은 토큰마다 선형층을 붙인다.
- 이미지에서 ImageNet으로 사전학습한 모델을 가져와 전이학습하는 것과 같은 아이디어다.

### 꼬리질문
#### 파인튜닝할 때 학습률을 작게 잡는 이유는 무엇인가요?
사전학습으로 얻은 가중치를 크게 망가뜨리지 않고 조금만 조정하려는 것입니다. 학습률이 크면 적은 작업 데이터에 맞추느라 사전학습에서 배운 일반적인 언어 지식을 잃어버릴 수 있습니다(catastrophic forgetting). BERT 논문은 2e-5~5e-5 범위를 썼습니다.

#### 모델이 아주 크면 전체를 파인튜닝하기 어렵지 않나요?
그래서 원래 가중치는 고정하고 작은 행렬만 추가로 학습하는 LoRA 같은 방법을 쓰거나, 가중치를 바꾸지 않고 프롬프트에 예시를 넣는 in-context learning을 씁니다.
