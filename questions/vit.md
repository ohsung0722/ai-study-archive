# Transformer 비전
blog: vit_blog/vit_clip_blog.md
blog_title: [CV 5일차] 이미지를 문장처럼 읽기: Vision Transformer와 CLIP

## Q. ViT(Vision Transformer) 모델에 대해 설명해 보세요.
id: vit-overview
type: 개념
level: 기본
tags: ViT, Transformer, 개요
source: 3. ViT 구조

### 30초 답변
ViT는 이미지를 작은 Patch로 잘라 문장의 단어처럼 줄 세운 다음, Transformer Encoder로 처리하는 이미지 분류 모델입니다. ViT-B/16 기준으로 224×224 이미지를 16×16 Patch 196개로 자르고, 각 Patch를 선형 투영해 768차원 토큰으로 만듭니다. 맨 앞에 학습 가능한 [CLS] 토큰을 붙이고 위치 임베딩을 더한 뒤 Encoder 12층을 통과시키고, 최종 [CLS] 출력만 분류 Head에 넣어 클래스를 예측합니다.

### 자세히
흐름은 **자르기 → 펼치기 → Patch Embedding → [CLS] 붙이기 → 위치 더하기 → Encoder → [CLS]만 꺼내기 → 분류** 8단계다.

- 핵심 아이디어: Patch 하나 = 토큰 하나. 이미지를 토큰 줄로 바꾸면 문장용 Transformer를 거의 그대로 쓸 수 있다.
- 논문: Dosovitskiy et al., 2020, "An Image is Worth 16x16 Words".
- 큰 데이터로 사전학습한 뒤 Head만 바꿔 파인튜닝하는 방식으로 쓴다.

| 모델 | Encoder 층 | Hidden size | 파라미터 |
|---|---|---|---|
| ViT-Base | 12 | 768 | 약 86M |
| ViT-Large | 24 | 1024 | 약 307M |
| ViT-Huge | 32 | 1280 | 약 632M |

### 꼬리질문
#### 모델 이름 ViT-B/16에서 B와 16은 무엇인가요?
B는 Base 크기를, 16은 Patch 한 변의 픽셀 수를 뜻합니다. Hugging Face의 `vit-base-patch16-224`는 여기에 입력 해상도 224까지 이름에 넣은 것입니다.

#### ViT도 번역 모델처럼 Decoder를 쓰나요?
분류용 ViT는 Encoder만 씁니다. 단어를 하나씩 생성할 필요가 없으므로, Encoder 출력 중 [CLS] 하나를 Linear Head에 넣으면 끝입니다.

## Q. CNN과 ViT는 이미지를 처리하는 방식이 어떻게 다른가요?
id: vit-vs-cnn
type: 비교
level: 기본
tags: CNN, ViT, Self-Attention
source: 1. CNN에서 Transformer로

### 30초 답변
CNN은 작은 커널을 이미지 위로 미끄러뜨리며 가까운 픽셀끼리 특징을 뽑고, 층을 쌓을수록 점점 넓은 영역을 봅니다. ViT는 이미지를 Patch 토큰으로 바꾼 뒤 Self-Attention으로 첫 층부터 모든 Patch 쌍의 관계를 직접 계산합니다. 또 CNN 필터는 학습이 끝나면 입력과 상관없이 고정이지만, ViT의 Attention 비중은 입력 이미지에 따라 그때그때 달라집니다.

### 자세히
| | CNN | ViT |
|---|---|---|
| 기본 단위 | 픽셀, 작은 커널 | 16×16 Patch 토큰 |
| 기본 연산 | 합성곱 | Self-Attention + MLP |
| 보는 범위 | 층이 깊어질수록 넓어짐 | 1층부터 이미지 전체 가능 |
| 섞는 비중 | 고정된 필터 가중치 | 입력에 따라 달라지는 Attention |
| 해상도 변화 | 층마다 줄어드는 계층 구조 | 모든 층에서 토큰 수 그대로 (197개) |
| 계산량 | 픽셀 수에 비례 | 토큰 수의 제곱에 비례 |

### 꼬리질문
#### 입력 해상도가 커지면 두 모델의 계산량은 어떻게 달라지나요?
CNN은 픽셀 수에 비례해 늘지만, ViT는 토큰 수가 늘고 Attention이 모든 토큰 쌍을 비교하므로 토큰 수의 제곱으로 늘어납니다. 그래서 고해상도 입력에서는 Swin Transformer처럼 작은 윈도우 안에서만 Attention을 하는 구조가 쓰입니다.

#### CNN과 ViT를 섞은 구조도 있나요?
있습니다. ViT 논문의 Hybrid 모델은 이미지를 직접 자르는 대신 ResNet이 뽑은 특징 맵을 Patch로 잘라 Transformer에 넣습니다.

## Q. 귀납적 편향(inductive bias)이란 무엇이고, CNN과 ViT는 각각 어떤 편향을 갖나요?
id: vit-inductive-bias
type: 개념
level: 심화
tags: Inductive Bias, CNN, ViT
source: 1. CNN에서 Transformer로

### 30초 답변
귀납적 편향은 데이터를 보기 전에 모델 구조에 미리 넣어 둔 가정입니다. CNN은 가까운 픽셀끼리 관련이 깊다는 지역성과, 같은 필터를 이미지 전체에 쓰는 가중치 공유를 구조에 갖고 있어서 적은 데이터로도 효율적으로 배웁니다. ViT는 이런 가정이 거의 없어서 Patch 사이 관계를 데이터에서 스스로 배워야 하고, 그만큼 데이터가 많이 필요합니다.

### 자세히
| 편향 | CNN | ViT |
|---|---|---|
| 지역성 | 커널이 주변 픽셀만 봄 | Attention은 처음부터 전역 |
| 가중치 공유 | 같은 필터를 모든 위치에 | Patch Embedding과 MLP는 토큰마다 공유 |
| 이동 등변성 | 구조적으로 보장 | 거의 없음 |
| 2D 이웃 구조 | 커널 모양 자체가 2D | Patch로 자를 때와 위치 임베딩 보간 때만 사용 |

- 가정이 데이터와 맞으면 적은 데이터로 빨리 배운다. 대신 가정에 없는 패턴은 배우기 어렵다.
- ViT 논문 표현으로는 ViT에서 지역적인 부분은 MLP뿐이고 Self-Attention은 전역이다.

### 꼬리질문
#### 이동 등변성(equivariance)과 이동 불변성(invariance)은 어떻게 다른가요?
등변성은 입력이 옮겨지면 출력 특징 맵도 같이 옮겨지는 성질이고, 합성곱이 이 성질을 가집니다. 불변성은 입력이 옮겨져도 출력이 그대로인 성질로, 풀링이나 전역 평균을 거치면서 생깁니다.

#### Swin Transformer는 어떤 편향을 다시 넣었나요?
작은 윈도우 안에서만 Attention을 계산해 지역성을 넣었고, Patch를 단계적으로 합쳐 CNN처럼 해상도가 줄어드는 계층 구조를 만들었습니다. 윈도우를 번갈아 밀어서(shifted window) 윈도우 사이 정보도 섞습니다.

## Q. ViT는 왜 대규모 데이터 사전학습이 특히 중요한가요?
id: vit-pretraining-scale
type: 이유
level: 기본
tags: 사전학습, 데이터 규모, ViT
source: 1. CNN에서 Transformer로

### 30초 답변
ViT는 귀납적 편향이 약해서, CNN이라면 구조로 공짜로 얻는 지역성 같은 성질까지 데이터에서 배워야 하기 때문입니다. ViT 논문에서도 ImageNet-1k만으로 학습하면 비슷한 크기의 ResNet보다 낮았고, ImageNet-21k에서는 비슷해졌으며, JFT-300M으로 사전학습하면 앞섰습니다. 데이터가 충분히 많아지면 오히려 고정된 가정이 없는 쪽이 더 유리해진다는 결과입니다.

### 자세히
| 사전학습 데이터 | 규모 | ResNet 대비 ViT |
|---|---|---|
| ImageNet-1k | 약 130만 장 | 낮음 |
| ImageNet-21k | 약 1,400만 장 | 비슷함 |
| JFT-300M | 약 3억 장 | 앞섬 |

- 블로그 실습도 ImageNet-21k로 사전학습한 `google/vit-base-patch16-224-in21k`에서 출발했다.
- 작은 데이터셋으로 처음부터 학습하면 충분한 특징을 배우기 어렵다.

### 꼬리질문
#### 큰 데이터 없이 ViT를 처음부터 학습하고 싶다면 어떻게 하나요?
DeiT처럼 ImageNet-1k만으로도 학습되게 만든 방법을 씁니다. 강한 데이터 증강과 정규화를 쓰고, CNN teacher의 예측을 distillation 토큰으로 배우게 해서 데이터 효율을 높였습니다.

#### 그러면 데이터가 적은 실무에서는 ViT를 못 쓰나요?
쓸 수 있습니다. 직접 학습하는 대신 이미 대규모로 사전학습된 ViT를 가져와 파인튜닝하면 됩니다. 콩잎 1,034장으로도 이 방식으로 test 0.961이 나왔습니다.

## Q. ViT의 Patch Embedding 과정을 설명해 보세요.
id: vit-patch-embedding
type: 개념
level: 기본
tags: Patch Embedding, Linear Projection, Conv2d
source: 3. ViT 구조 › ② 펼치기와 ③ Patch Embedding

### 30초 답변
이미지를 16×16 Patch로 자르고, 각 Patch의 RGB 값 16×16×3 = 768개를 한 줄로 펼친 다음, 학습되는 행렬을 곱해 Hidden size 길이의 벡터로 바꾸는 과정입니다. 이 벡터가 Transformer에 들어가는 토큰 하나입니다. 실제 구현은 커널 크기와 stride가 모두 16인 Conv2d 한 층과 같습니다.

### 자세히
- 자르기와 펼치기(Flatten)는 모양만 바꾸고 학습되는 값이 없다. 학습되는 건 투영 행렬 W뿐이다.
- 입력 `(B, 3, 224, 224)`가 `(B, 196, 768)`이 된다.
- 펼친 길이 768(= 16×16×3)과 Hidden size 768은 **다른 설정**이다. B/16에서 우연히 같을 뿐이고, B/32는 펼친 길이 3,072에 Hidden size는 그대로 768이다.

```python
proj = nn.Conv2d(3, 768, kernel_size=16, stride=16)  # Patch마다 겹치지 않게 한 번씩
x = proj(img)                       # (B, 768, 14, 14)
x = x.flatten(2).transpose(1, 2)    # (B, 196, 768)
```

### 꼬리질문
#### 학습된 Patch Embedding 가중치를 꺼내 보면 어떤 모양인가요?
가중치 (768, 3, 16, 16)을 16×16×3 필터 768개로 보고 주성분을 그려 보니, 초록과 보라의 색 대비, 가로·세로 줄무늬, 점 모양 패턴이 나왔습니다. CNN 첫 층 필터와 꽤 닮았는데, Patch 안에서 저수준 특징을 뽑는 역할이 비슷하기 때문입니다.

#### 왜 Linear가 아니라 Conv2d로 구현하나요?
계산 결과는 같습니다. stride가 커널 크기와 같으면 Patch가 겹치지 않으므로 Patch마다 같은 행렬을 곱하는 것과 동일하고, Conv2d를 쓰면 자르기와 펼치기를 따로 하지 않아도 되어 간편합니다.

## Q. ViT에서 [CLS] 토큰은 어떤 역할을 하나요?
id: vit-cls-token
type: 개념
level: 기본
tags: CLS, ViT, BERT
source: 3. ViT 구조 › ④ [CLS] 토큰

### 30초 답변
[CLS]는 이미지 전체를 대표하는 표현을 만들기 위해 Patch 토큰 맨 앞에 붙이는 학습 가능한 특수 토큰입니다. 처음 값은 이미지와 상관없이 학습으로 정해진 벡터인데, Encoder의 Self-Attention을 거치면서 다른 Patch들의 정보를 끌어모읍니다. 분류할 때는 197개 출력 중 이 [CLS] 출력 하나만 분류 Head에 넣습니다. BERT의 [CLS]와 같은 아이디어입니다.

### 자세히
- 196개 Patch + [CLS] 1개 = 197개 토큰.
- [CLS] 자체에는 이미지 내용이 없다. 정보는 전부 Attention으로 Patch에서 가져온다.
- 최종 출력 `(B, 197, 768)`에서 0번 토큰만 골라 `(B, 768)`을 만들고, Linear Head로 `(B, num_labels)`를 낸다.
- 고르는 동작은 학습되지 않고, [CLS] 벡터와 Head는 학습된다.

### 꼬리질문
#### [CLS] 토큰의 처음 값은 이미지마다 다른가요?
아닙니다. 모든 이미지에 같은 학습된 벡터가 붙습니다. Encoder를 지나면서 이미지마다 다른 Patch 정보를 모으기 때문에 출력이 달라집니다.

#### [CLS] 위치에도 Position Embedding을 더하나요?
더합니다. 위치 임베딩은 [CLS]를 포함한 197개 위치마다 하나씩 있습니다.

## Q. 분류에 [CLS] 토큰을 쓰는 것과 Patch 출력의 평균(Global Average Pooling)을 쓰는 것은 어떻게 다른가요?
id: vit-cls-vs-gap
type: 비교
level: 심화
tags: CLS, GAP, Pooling
source: 보충 (블로그 밖 내용)

### 30초 답변
[CLS] 방식은 별도 토큰이 Attention으로 필요한 Patch 정보를 골라 모으게 하고, GAP 방식은 마지막 층의 Patch 출력 196개를 그냥 평균 냅니다. ViT 논문 부록에서는 학습률을 각각 맞춰 주면 두 방식의 성능이 비슷했습니다. 원래 ViT는 BERT 구조를 최대한 그대로 가져오려고 [CLS]를 썼고, Swin Transformer처럼 GAP을 쓰는 모델도 많습니다.

### 자세히
| | [CLS] 토큰 | GAP |
|---|---|---|
| 요약 방법 | 학습되는 토큰이 Attention으로 모음 | Patch 출력의 단순 평균 |
| 추가 파라미터 | [CLS] 벡터 768개 | 없음 |
| 토큰 수 | 197 | 196 |
| 쓰는 모델 | ViT, BERT, CLIP의 ViT | Swin Transformer, 대부분의 CNN |

- GAP은 CNN 분류기의 마지막 단계와 같은 방식이라 익숙하다.
- [CLS]는 모든 정보를 한 토큰에 모으므로, 그 토큰의 Attention을 꺼내 보면 어디서 정보를 가져왔는지 볼 수 있다.

### 꼬리질문
#### GAP을 쓰면 [CLS] 토큰은 아예 없어도 되나요?
네, 분류에 쓰지 않는다면 붙일 이유가 없습니다. 이 경우 위치 임베딩도 196개만 있으면 됩니다.

## Q. ViT에 Position Embedding이 왜 필요하고, 어떻게 넣나요?
id: vit-position-why
type: 이유
level: 기본
tags: Position Embedding, Self-Attention, ViT
source: 3. ViT 구조 › ⑤ 위치 더하기

### 30초 답변
Self-Attention은 토큰 순서를 섞어도 각 토큰이 섞는 값이 그대로라서, Patch가 원래 이미지의 어디에 있었는지 알 수 없기 때문입니다. 그래서 [CLS]를 포함한 197개 위치마다 768차원 위치 벡터를 Patch 임베딩에 더합니다. 기본 ViT는 위치마다 학습되는 1차원 Position Embedding을 씁니다.

### 자세히
- 이어 붙이지 않고 **더한다**. 이어 붙이면 차원이 늘어나지만, 더하면 768차원을 유지하면서 내용 정보에 위치 정보를 보탠다.
- 파라미터 모양은 `(1, 197, 768)`이고 다른 가중치와 함께 학습된다.
- 1차원이라는 건 위치를 0~196 번호로만 구분한다는 뜻이다. 행과 열을 따로 알려 주지 않는다.
- ViT 논문 실험에서 위치 정보가 없으면 성능이 확실히 떨어졌고, 1D 학습형, 2D 학습형, 상대 위치 방식끼리는 차이가 작았다.

### 꼬리질문
#### Transformer 원 논문의 sin·cos 위치 인코딩과 무엇이 다른가요?
sin·cos는 공식으로 정해져 학습되지 않고, 길이가 늘어나도 계산할 수 있습니다. 학습형은 데이터에 맞춰 유연하게 배우지만, 학습한 개수 이상으로 늘리려면 보간이 필요합니다.

#### 더하면 내용 정보와 위치 정보가 섞여서 구분이 안 되지 않나요?
768차원처럼 차원이 크면 두 정보가 서로 다른 방향을 차지할 여지가 충분해서, 이후 층의 선형 변환이 둘을 나눠 쓸 수 있습니다. 실제로 더하는 방식으로도 위치 구조가 잘 학습됩니다.

## Q. 학습된 ViT의 Position Embedding을 직접 시각화해 보니 어땠나요?
id: vit-position-embedding
type: 경험
level: 심화
tags: Position Embedding, 시각화, 실험
source: 3. ViT 구조 › ⑤ 위치 더하기

### 30초 답변
사전학습된 ViT-B/16의 Patch 위치 벡터 196개끼리 코사인 유사도를 구해서, 위치마다 나와 비슷한 위치가 어디인지 14×14 지도로 그렸습니다. 모든 위치에서 같은 행과 같은 열의 위치끼리 비슷한 십자 모양이 나타났습니다. 1차원 번호만 받았는데도 학습만으로 2차원 격자 구조를 배운 것입니다.

### 자세히
- 모델: `google/vit-base-patch16-224`, [CLS]를 뺀 위치 벡터 196개.
- (i, j) 칸 = (i, j) 위치 벡터와 196개 위치의 유사도를 다시 14×14로 배치한 작은 지도.
- 결과: 모든 칸에 십자 모양. 자기 자신 근처와 같은 행, 같은 열이 밝다.
- ViT 논문에 실린 그림과 같은 결과다. 아무도 2차원 격자라고 알려 주지 않았다는 점이 핵심이다.

### 꼬리질문
#### 1차원 번호만 줬는데 어떻게 행과 열을 배우나요?
같은 행이나 같은 열, 가까운 Patch는 이미지 내용도 비슷한 경우가 많아서, 학습 과정에서 그런 위치들이 비슷한 역할을 하도록 위치 벡터가 맞춰집니다. 구조가 데이터에 있으니 학습형 임베딩이 그것을 흡수한 것입니다.

#### 그렇다면 2D 위치 임베딩을 따로 설계할 필요는 없나요?
ViT 논문에서도 2D 방식이 1D보다 뚜렷하게 낫지 않았습니다. 시각화 결과처럼 1D로도 2D 구조를 배우기 때문으로 해석할 수 있습니다.

## Q. ViT의 Patch 크기를 16에서 32로 키우면 어떤 변화가 생기나요?
id: vit-patch-size
type: 이유
level: 심화
tags: Patch, 계산량, ViT
source: 3. ViT 구조 › ① 자르기

### 30초 답변
224×224 기준 Patch가 196개에서 49개로 약 4분의 1이 됩니다. Self-Attention은 모든 토큰 쌍을 비교하므로 계산량이 토큰 수의 제곱에 비례해서, 197²에서 50²로 약 15분의 1로 줄어듭니다. 대신 Patch 한 칸이 큰 영역을 통째로 보게 되어 작은 물체나 세밀한 무늬를 구분하는 데는 불리해집니다.

### 자세히
| | B/16 | B/32 |
|---|---|---|
| Patch 수 | 14×14 = 196 | 7×7 = 49 |
| 토큰 수 ([CLS] 포함) | 197 | 50 |
| 펼친 Patch 길이 | 768 | 3,072 |
| Hidden size | 768 | 768 |
| Attention 표 크기 | 197² ≈ 38,800 | 50² = 2,500 |

CLIP 실습에서 쓴 `clip-vit-base-patch32`는 32 쪽이다.

### 꼬리질문
#### 반대로 Patch를 14나 8로 줄이면 어떻게 되나요?
토큰 수가 늘어 세밀한 정보를 더 보지만 Attention 계산량이 제곱으로 커집니다. CLIP의 큰 모델 ViT-L/14가 14 크기 Patch를 씁니다.

#### Patch 크기가 바뀌면 Patch Embedding 파라미터 수는 어떻게 되나요?
투영 행렬이 (펼친 길이 × Hidden size)라서 B/32는 3,072 × 768로 B/16의 768 × 768보다 4배 큽니다. 다만 Encoder 파라미터에 비하면 작은 부분입니다.

## Q. 파인튜닝 때 입력 해상도를 바꾸면 Position Embedding은 어떻게 처리하나요?
id: vit-position-interpolation
type: 이유
level: 심화
tags: Position Embedding, 해상도, 보간
source: 보충 (블로그 밖 내용)

### 30초 답변
Patch 크기를 그대로 두고 해상도를 올리면 토큰 수가 늘어나는데, 학습된 위치 임베딩은 원래 개수만큼만 있기 때문에 그대로 쓸 수 없습니다. 그래서 Patch 위치 임베딩을 원래 격자 모양으로 되돌린 뒤 2D 보간으로 새 격자 크기에 맞게 늘리고, [CLS] 위치 벡터는 따로 그대로 둡니다. ViT 논문도 사전학습보다 높은 해상도로 파인튜닝하면서 이 방법을 썼습니다.

### 자세히
224 → 384, Patch 16이면 14×14 = 196개가 24×24 = 576개가 된다.

```python
cls_pe, patch_pe = pe[:, :1], pe[:, 1:]                        # (1,1,768), (1,196,768)
patch_pe = patch_pe.reshape(1, 14, 14, 768).permute(0, 3, 1, 2)
patch_pe = F.interpolate(patch_pe, size=(24, 24), mode="bicubic", align_corners=False)
patch_pe = patch_pe.permute(0, 2, 3, 1).reshape(1, 576, 768)
pe_new = torch.cat([cls_pe, patch_pe], dim=1)                  # (1, 577, 768)
```

- 1차원 벡터 줄이 아니라 2차원 격자로 보간해야 행과 열 구조가 유지된다.
- Hugging Face ViT는 forward에 `interpolate_pos_encoding=True`를 주면 이 보간을 해 준다.

### 꼬리질문
#### 왜 파인튜닝을 더 높은 해상도로 하나요?
토큰 수가 늘어 같은 물체를 더 많은 Patch로 보므로 세밀한 특징을 잡기 좋습니다. 사전학습은 비용 때문에 낮은 해상도로 하고, 비교적 짧은 파인튜닝에서 해상도를 올리는 식으로 나눕니다.

#### Patch 크기를 키워서 토큰 수를 유지하면 안 되나요?
그러면 학습된 Patch Embedding 행렬의 입력 크기가 달라져 그대로 쓸 수 없습니다. 그래서 보통 Patch 크기는 고정하고 위치 임베딩만 보간합니다.

## Q. ViT-B/16에서 입력부터 출력까지 텐서 shape이 어떻게 바뀌나요?
id: vit-shape-flow
type: 개념
level: 기본
tags: Tensor shape, ViT-B/16, 하이퍼파라미터
source: 3. ViT 구조 › ⑥ Encoder ~ ⑧ 분류

### 30초 답변
입력 (B, 3, 224, 224)가 Patch Embedding을 거쳐 (B, 196, 768)이 되고, [CLS]를 붙이고 위치를 더하면 (B, 197, 768)이 됩니다. Encoder 12층을 지나도 shape은 (B, 197, 768) 그대로이고, 0번 [CLS]만 골라 (B, 768), 분류 Head를 거쳐 (B, num_labels)가 됩니다. forward hook으로 직접 찍어 확인했습니다.

### 자세히
| 단계 | shape | 설명 |
|---|---|---|
| 입력 | (B, 3, 224, 224) | 원본 이미지 배치 |
| Patch Embedding | (B, 196, 768) | Patch를 펼쳐 768차원으로 선형 투영 |
| CLS 추가 + 위치 | (B, 197, 768) | 맨 앞에 [CLS], 위치 벡터를 더함 |
| Encoder 출력 | (B, 197, 768) | Self-Attention과 MLP를 12번 |
| 최종 CLS | (B, 768) | 0번 토큰만 골라냄 |
| Classifier 출력 | (B, num_labels) | Linear Head로 클래스별 점수 |

하이퍼파라미터: Patch 16, Hidden 768, Encoder 12층, Head 12개(Head당 64차원), MLP 내부 3,072(Hidden의 4배), 전체 약 86M.

### 꼬리질문
#### 층마다의 중간 결과는 어떻게 꺼내나요?
`output_hidden_states=True`를 주면 길이 13인 튜플(임베딩 출력 1 + Encoder 12층)이 나오고, 전부 (1, 197, 768)입니다.

#### Attention 표의 shape은 어떻게 되나요?
`output_attentions=True`를 주면 층마다 (1, 12, 197, 197)이 나옵니다. 이미지 1장, Head 12개, Query 197개, Key 197개입니다. 기본 구현(`sdpa`)은 이 표를 내주지 않아서 `attn_implementation="eager"`로 불러와야 합니다.

## Q. ViT Encoder 블록 한 층은 어떻게 구성되어 있나요?
id: vit-encoder-block
type: 개념
level: 심화
tags: Encoder, LayerNorm, GELU, Residual
source: 2. Attention 다시 보기 › Self-Attention, 위치 정보, Encoder 한 층

### 30초 답변
LayerNorm 다음 Multi-Head Self-Attention을 하고 입력을 더하는 잔차 연결, 다시 LayerNorm 다음 MLP를 하고 입력을 더하는 잔차 연결, 이렇게 두 부분입니다. 정규화를 각 연산 앞에 두는 pre-LayerNorm 구조이고, MLP는 768에서 3,072로 늘렸다가 GELU를 거쳐 768로 되돌립니다. 입력과 출력 shape이 같아서 이 블록을 12번 쌓을 수 있습니다.

### 자세히
```
z' = z  + MSA(LN(z))
z  = z' + MLP(LN(z'))      # MLP: Linear(768→3072) → GELU → Linear(3072→768)
```

- MSA: Head 12개 × 64차원. 토큰끼리 정보가 섞이는 곳은 여기뿐이다.
- MLP: 토큰마다 같은 가중치로 따로 적용된다.
- 마지막 블록 뒤에 LayerNorm이 한 번 더 있고, 그 출력의 [CLS]가 Head로 간다.
- Q, K, V와 Multi-Head 자체의 원리는 Transformer 기초(NLP 파트)와 같다.

### 꼬리질문
#### pre-LayerNorm과 post-LayerNorm은 무엇이 다른가요?
원래 Transformer는 잔차를 더한 뒤 정규화하는 post-LN이고, ViT는 연산 앞에서 정규화하는 pre-LN입니다. pre-LN은 잔차 경로가 정규화 없이 그대로 이어져서, 층이 깊어도 학습이 더 안정적인 편입니다.

#### 잔차 연결은 왜 필요한가요?
입력을 출력에 그대로 더해 두면 기울기가 깊은 층까지 잘 전달되고, 각 블록은 입력에 대한 수정분만 배우면 됩니다. 그래서 shape이 같아야 하고, 블록을 여러 개 쌓을 수 있습니다.

## Q. ViT에서 학습되는 부분과 학습되지 않는 부분을 구분해 보세요.
id: vit-trainable-parts
type: 개념
level: 기본
tags: 파라미터, 학습, ViT
source: 3. ViT 구조 › 학습되는 부분과 아닌 부분

### 30초 답변
이미지를 Patch로 자르고 펼치는 단계, 그리고 마지막에 [CLS] 하나를 골라내는 단계는 모양만 바꾸거나 고르는 것이라 학습되는 값이 없습니다. 학습되는 것은 Patch Embedding 행렬, [CLS] 벡터, 위치 임베딩, Encoder 안의 가중치, 분류 Head입니다. 파라미터의 대부분은 Encoder에 있습니다.

### 자세히
| 단계 | 학습되나 |
|---|---|
| ① 자르기, ② 펼치기 | 안 됨. 칸을 나누고 모양만 바꿈 |
| ③ Patch Embedding W | 학습됨 |
| ④ [CLS] 벡터 | 학습됨 |
| ⑤ 위치 임베딩 | 학습됨 (기본 ViT) |
| ⑥ Encoder | 학습됨. W_q, W_k, W_v, MLP, LayerNorm |
| ⑦ CLS만 꺼내기 | 안 됨. 고르기만 함 |
| ⑧ MLP Head | 학습됨 |

### 꼬리질문
#### 파라미터는 어디에 몰려 있나요?
ViT-B/16 Backbone 85,798,656개 중 Encoder 12층이 약 8,505만 개로 99%를 차지합니다. 한 층이 Attention 약 236만 개, MLP 약 472만 개라서 MLP 쪽이 두 배 정도 큽니다. Patch Embedding은 약 59만 개, 위치 임베딩은 197 × 768 = 151,296개입니다.

#### Head Only 파인튜닝에서는 위치 임베딩도 고정되나요?
네. `model.vit`에 Patch Embedding, [CLS], 위치 임베딩, Encoder가 모두 들어 있어서 이 부분을 Freeze하면 함께 고정되고, 새 분류 Head만 학습됩니다.

## Q. [CLS] Attention 지도를 "모델이 본 곳"으로 해석해도 되나요?
id: vit-attention-map
type: 경험
level: 심화
tags: Attention 시각화, Rollout, Registers
source: 4. Hugging Face로 사전학습 ViT 쓰기 › 층마다의 출력과 Attention 꺼내 보기

### 30초 답변
조심해서 해석해야 합니다. Attention 표에서 [CLS] 행만 떼어 Head 평균을 내 14×14로 그려 보니, 층을 누적한 Rollout에서는 물체 쪽이 밝게 나왔지만, 8층과 12층에서는 물체가 없는 배경 칸 몇 개가 유난히 밝았습니다. 큰 ViT가 정보가 적은 배경 토큰을 계산용 메모리처럼 쓰는 현상으로 알려져 있어서, Attention 한 장만으로 판단 근거를 단정하면 안 됩니다.

### 자세히
- 꺼내기: `attn_implementation="eager"`, `output_attentions=True` → 층마다 (1, 12, 197, 197).
- [CLS] 행(Query = 0번)의 Patch 196칸을 Head 평균 → 14×14 지도.
- 관찰: 1층은 거의 고르게 퍼져 있고, 깊어질수록 특정 칸에 몰린다.
- Rollout(모든 층 누적)에서는 개, 버스 앞 사람들처럼 물체 쪽이 밝다.
- 개 사진의 바닥, 콩잎 사진의 흙 같은 배경 칸이 깊은 층에서 튀었다.
- 참고: Darcet et al., *Vision Transformers Need Registers*, 2023.

### 꼬리질문
#### Attention Rollout은 어떻게 계산하나요?
층마다 Head 평균 Attention에 잔차 연결을 반영하는 단위 행렬을 더하고 행 합이 1이 되게 정규화한 뒤, 1층부터 마지막 층까지 행렬을 곱해 누적합니다. 한 층의 Attention만 보면 이미 섞인 토큰끼리의 관계라서, 원래 Patch까지 거슬러 올라가려는 방법입니다.

#### Registers 논문은 이 문제를 어떻게 해결했나요?
입력에 이미지와 무관한 학습 가능한 토큰(register)을 몇 개 더 붙이고 출력에서는 버립니다. 모델이 계산용 메모리로 배경 Patch 대신 이 토큰을 쓰게 되어 Attention 지도가 깨끗해집니다.

## Q. ViT의 Attention Head들은 얼마나 먼 곳을 보나요? 직접 측정해 본 결과를 말해 보세요.
id: vit-attention-distance
type: 경험
level: 심화
tags: Attention distance, Receptive field, CNN 비교
source: 4. Hugging Face로 사전학습 ViT 쓰기 › 층마다의 출력과 Attention 꺼내 보기

### 30초 답변
Head마다 Query Patch와 Attention을 준 Patch 사이의 거리를 Attention 비중으로 평균 내서, 사진 8장에 대해 층별로 비교했습니다. 1층부터 Head마다 제각각이어서 0~22px 정도로 자기 주변만 보는 Head가 있는 반면, 110px 넘게 이미지 반대편까지 보는 Head도 있었습니다. 깊은 층에서는 모든 Head가 115~120px로 모여 이미지 전체를 봤습니다.

### 자세히
- 거리 = Σ (Attention 비중 × Patch 사이 픽셀 거리), Query Patch 전체에 대해 평균. 224 입력 기준.
- CNN의 3×3 커널은 첫 층에서 바로 옆 픽셀만 본다.
- ViT는 1층부터 전역을 볼 수 있다는 설명이 숫자로 확인된다.
- 동시에 앞쪽 층 일부 Head는 CNN처럼 지역적인 역할을 스스로 배운다.
- ViT 논문에도 같은 종류의 분석(mean attention distance)이 있다.

### 꼬리질문
#### Attention distance는 CNN의 어떤 개념과 비슷한가요?
수용 영역(receptive field)과 비슷합니다. 다만 CNN의 수용 영역은 구조로 정해지지만, Attention distance는 학습된 결과이고 입력마다 달라집니다.

#### 지역성 편향이 없는데 왜 일부 Head는 주변만 보게 되나요?
가까운 Patch끼리 관련이 깊은 이미지의 성질이 데이터에 있으니, 앞쪽 층에서 지역 특징을 모으는 것이 손실을 줄이는 데 유리하기 때문입니다. CNN이 구조로 넣은 가정을 ViT는 데이터에서 배운 셈입니다.

## Q. Hugging Face의 from_pretrained, Auto 클래스, Processor는 무엇이고, 왜 모델과 전처리기를 한 쌍으로 불러와야 하나요?
id: vit-hf-loading
type: 개념
level: 기본
tags: Hugging Face, from_pretrained, Processor
source: 4. Hugging Face로 사전학습 ViT 쓰기 › from_pretrained, Auto 클래스, Processor

### 30초 답변
`from_pretrained()`는 Hub 저장소에서 설정과 가중치를 내려받아 사전학습된 모델 객체를 만드는 함수이고, Auto 클래스는 모델 이름만 보고 알맞은 구현 클래스를 골라 줍니다. Processor는 리사이즈와 정규화 같은 전처리를 담당합니다. 모델은 학습할 때 쓴 전처리를 그대로 기대하므로, 같은 Model ID로 모델과 전처리기를 같이 불러와야 입력 분포가 맞습니다.

### 자세히
```python
model_id = "google/vit-base-patch16-224"
processor = AutoImageProcessor.from_pretrained(model_id)          # → ViTImageProcessor
model = AutoModelForImageClassification.from_pretrained(model_id) # → ViTForImageClassification
```

- 저장소 파일: `config.json`(구조), `model.safetensors`(가중치), `preprocessor_config.json`(이미지 크기, 정규화 값).
- `model.config`: hidden_size 768, num_hidden_layers 12, num_attention_heads 12, patch_size 16, num_labels 1000.
- 전처리기 출력 `pixel_values`는 (1, 3, 224, 224).
- 파일은 `~/.cache/huggingface/hub`에 캐시되어 두 번째부터는 내려받지 않는다.

### 꼬리질문
#### 전처리를 다르게 하면 어떤 일이 생기나요?
정규화 평균과 표준편차나 채널 순서가 다르면 모델이 학습 때 본 적 없는 분포의 입력을 받게 되어 성능이 크게 떨어집니다. ResNet에서 BGR 순서나 정규화 값을 틀렸을 때와 같은 문제입니다.

#### revision을 지정하는 이유는 무엇인가요?
Hub 저장소는 Git 기반이라 파일이 바뀔 수 있습니다. 브랜치, 태그, 커밋을 `revision`으로 고정하면 나중에 실행해도 같은 모델 파일을 써서 결과를 재현할 수 있습니다.

## Q. pipeline 한 줄 안에서는 어떤 단계가 일어나나요?
id: vit-hf-pipeline
type: 개념
level: 기본
tags: pipeline, logits, softmax
source: 4. Hugging Face로 사전학습 ViT 쓰기 › pipeline 안에서 일어나는 일

### 30초 답변
전처리, 모델 추론, 후처리 세 단계가 묶여 있습니다. 이미지를 `pixel_values` 텐서로 바꾸고, 모델이 클래스별 원시 점수인 logits를 낸 다음, softmax로 확률을 만들어 상위 클래스의 라벨 이름과 점수를 돌려줍니다. 실습에서 이 세 단계를 손으로 풀어 봤더니 pipeline 결과와 숫자가 똑같이 나왔습니다.

### 자세히
```python
inputs = processor(images=image, return_tensors="pt")   # 1. 전처리
with torch.no_grad():
    outputs = model(**inputs)                           # 2. logits (1, 1000)
probs = torch.softmax(outputs.logits, dim=-1)           # 3. 후처리
top_probs, top_ids = torch.topk(probs, k=5, dim=-1)
labels = [model.config.id2label[i.item()] for i in top_ids[0]]
```

결과(`google/vit-base-patch16-224`): 버스 사진은 minibus 0.868, 개 사진은 malamute 0.316, Siberian husky 0.305, Eskimo dog 0.153.

### 꼬리질문
#### 개 사진의 1위 확률이 0.316밖에 안 되는데, 모델이 못 맞힌 건가요?
그렇게 보기 어렵습니다. ImageNet-1k에는 개 품종만 100개가 넘어서, 품종이 섞여 보이는 개는 비슷한 견종끼리 확률을 나눠 가집니다. 상위 3개가 모두 썰매견 계열이었습니다.

#### logits와 확률은 어떻게 다른가요?
logits는 범위 제한이 없는 원시 점수이고, softmax를 거쳐야 합이 1인 확률이 됩니다. 1위 클래스만 고를 때는 softmax 전후 순서가 같아서 logits의 argmax로도 충분합니다.

## Q. 전이학습과 파인튜닝이란 무엇이고, 왜 사전학습 모델에서 시작하나요?
id: vit-transfer-learning
type: 개념
level: 기본
tags: 전이학습, Fine-tuning, 사전학습
source: 5. ViT 파인튜닝 › 왜 사전학습 모델에서 시작하나

### 30초 답변
전이학습은 큰 데이터로 미리 학습한 모델의 지식을 새 작업에 가져와 쓰는 것이고, 파인튜닝은 그 모델을 새 데이터로 추가 학습하는 과정입니다. 사전학습 모델은 일반적인 이미지 특징을 이미 배운 상태라서, 작은 데이터에서도 그 특징을 재사용하고 필요한 부분만 더 배우면 됩니다. 특히 ViT는 처음부터 학습하면 데이터가 많이 필요해서 이 방식이 거의 기본입니다.

### 자세히
- 순서: 큰 데이터로 사전학습 → 분류 Head를 떼고 새 클래스 수의 Head를 붙임 → 작은 데이터로 파인튜닝.
- 실습: ImageNet-21k로 사전학습한 `google/vit-base-patch16-224-in21k`에서 출발해 beans(콩잎 3클래스, train 1,034 / val 133 / test 128)로 학습.
- 3일차 YOLO 전이학습과 같은 이야기다.

| | 처음부터 학습 | 사전학습 모델에서 시작 |
|---|---|---|
| 초기 가중치 | 무작위 | 사전학습된 표현 |
| 필요한 데이터 | 많음 | 적어도 됨 |
| 수렴 속도 | 느림 | 빠름 |

### 꼬리질문
#### 왜 ImageNet-1k 분류 모델이 아니라 in21k 모델을 출발점으로 썼나요?
in21k 모델은 1,000개 클래스 분류에 맞춰 파인튜닝되지 않은 범용 Backbone이라, 새 Head를 붙여 다른 작업에 맞추는 출발점으로 쓰기 좋습니다.

#### 사전학습 데이터와 새 데이터의 분야가 많이 다르면 어떻게 되나요?
색, 모양, 질감 같은 저수준 특징은 여전히 쓸모가 있어서 보통 도움이 됩니다. 다만 분야가 멀수록 Head만 학습해서는 부족하고, Backbone까지 학습하는 쪽의 이득이 커집니다.

## Q. Head Only(Freeze)와 Full Fine-tuning은 어떻게 다른가요?
id: vit-freeze-vs-full
type: 비교
level: 기본
tags: Freeze, Fine-tuning, Head 교체
source: 5. ViT 파인튜닝 › Freeze, Head Only, Full Fine-tuning

### 30초 답변
Head Only는 Backbone을 Freeze해서 가중치를 고정하고 새 분류 Head만 학습하는 방식이고, Full Fine-tuning은 Backbone과 Head를 모두 학습하는 방식입니다. Head Only는 빠르고 안정적이며 과적합 위험이 작지만 새 데이터에 적응하는 범위가 좁습니다. Full은 적응력이 높은 대신 계산량과 메모리가 크고 과적합 위험이 있어서 작은 학습률을 씁니다.

### 자세히
| | Head Only | Full Fine-tuning |
|---|---|---|
| 학습 대상 | 분류 Head | Backbone + Head |
| 속도, 메모리 | 가벼움 | 무거움 |
| 적응력 | 낮음 | 높음 |
| 과적합 위험 | 작음 | 큼 |

```python
model = AutoModelForImageClassification.from_pretrained(
    "google/vit-base-patch16-224-in21k", num_labels=3,
    id2label=id2label, label2id=label2id,
    ignore_mismatched_sizes=True)          # 크기가 다른 분류층은 새로 만든다
for p in model.vit.parameters():
    p.requires_grad = False                # Head Only: Backbone 고정
```

### 꼬리질문
#### Head를 `nn.Linear`로 직접 바꿔 끼우면 안 되나요?
가능하지만 `config.num_labels`, `id2label`, `label2id`까지 손으로 고쳐야 해서 빠뜨리기 쉽습니다. `from_pretrained`에 같이 넘기면 이 정보가 한 번에 맞춰집니다.

#### 둘 사이의 중간 방법도 있나요?
Backbone의 앞쪽 층은 고정하고 뒤쪽 몇 층과 Head만 학습하는 부분 Freeze가 있습니다. 앞쪽 층은 범용적인 저수준 특징이라 그대로 두고, 작업에 가까운 뒤쪽 층만 맞추는 방식입니다.

## Q. Head Only와 Full Fine-tuning을 직접 비교해 본 결과는 어땠나요?
id: vit-finetune-strategy
type: 경험
level: 심화
tags: Fine-tuning, Freeze, 실험
source: 5. ViT 파인튜닝 › Freeze, Head Only, Full Fine-tuning

### 30초 답변
콩잎 질병 3클래스 데이터로 비교했을 때 Head Only는 학습 파라미터 2,307개로 test 정확도 0.875, Full은 약 8,580만 개로 0.961이 나왔습니다. 파라미터 수는 약 3만 7천 배 차이였지만 에폭당 시간은 1.4~1.7배 차이에 그쳤는데, Head Only도 순전파는 Backbone 전체를 지나야 하기 때문입니다. 대신 GPU 메모리는 Full 쪽이 훨씬 많이 씁니다.

### 자세히
설정: `google/vit-base-patch16-224-in21k`, beans, batch 16.

| | Head Only | Full |
|---|---|---|
| 학습률 · 에폭 | 1e-3 · 3 | 2e-5 · 2 |
| 학습 파라미터 | 2,307 (= 768×3 + 3) | 85,800,963 |
| 검증 정확도 | 0.812 → 0.850 → 0.880 | 첫 에폭 0.992 |
| test 정확도 | 0.875 | 0.961 |

- 실습 노트북에서도 Head Only 0.887, Full 0.985 → 0.970(검증), Full test 0.953으로 같은 그림이었다.
- Head Only가 아끼는 건 역전파와 가중치 갱신뿐이다.

### 꼬리질문
#### Head Only만으로 88%가 나온다는 건 무슨 의미인가요?
ImageNet-21k로 사전학습한 [CLS] 벡터에 이미 잎 상태를 가를 정보가 상당히 들어 있다는 뜻입니다. Full Fine-tuning은 그 위에서 반점 같은 세밀한 특징까지 맞춰 10%p 가까이 더 올렸습니다.

#### Full Fine-tuning에서 메모리는 왜 그렇게 많이 드나요?
8,580만 개 가중치 모두의 기울기와 AdamW의 모멘트 상태를 들고 있어야 하기 때문입니다. Head Only는 이것들이 Head의 2,307개분만 필요합니다.

## Q. Full Fine-tuning에서 학습률을 작게 하는 이유는 무엇인가요?
id: vit-finetune-lr
type: 이유
level: 기본
tags: Learning rate, Fine-tuning, 사전학습
source: 5. ViT 파인튜닝 › Freeze, Head Only, Full Fine-tuning

### 30초 답변
사전학습된 가중치는 이미 좋은 표현을 담고 있어서, 큰 학습률로 크게 움직이면 그 표현이 망가지기 때문입니다. 특히 새로 붙인 Head는 처음에 무작위라 초반 기울기가 크고 엉뚱한데, 이 기울기가 Backbone까지 그대로 흘러가면 사전학습 효과를 잃을 수 있습니다. 그래서 실습에서도 Head Only는 1e-3, Full은 2e-5로 50배 작게 썼습니다.

### 자세히
- 목표는 처음부터 다시 배우는 게 아니라 사전학습 가중치 근처에서 조금만 고치는 것이다.
- 데이터가 작으면 큰 걸음은 과적합도 키운다.
- Head만 학습할 때는 고정된 특징 위의 작은 Linear만 맞추면 되니 큰 학습률로 빨리 수렴시킨다.

| 방식 | 실습 학습률 |
|---|---|
| Head Only | 1e-3 |
| Full Fine-tuning | 2e-5 |

### 꼬리질문
#### 작은 학습률 말고 사전학습 표현을 지키는 다른 방법은?
학습 초반에 학습률을 서서히 올리는 warmup, Head를 먼저 학습한 뒤 전체를 파인튜닝하는 2단계 방식, 앞쪽 층일수록 학습률을 작게 주는 층별 학습률 감소(layer-wise lr decay)가 있습니다.

## Q. 파인튜닝 전에 Baseline을 재는 이유는 무엇이고, 학습 후 틀린 사진을 분석해 보니 어땠나요?
id: vit-baseline-errors
type: 경험
level: 기본
tags: Baseline, 오류 분석, Confusion Matrix
source: 5. ViT 파인튜닝 › 분류 Head 교체

### 30초 답변
학습 효과가 정말 있는지 판단할 기준점이 필요하기 때문입니다. 새 Head는 무작위라 학습 전 정확도가 제 실험에서 0.429, 실습 노트북에서는 0.218로 3클래스 무작위 기대값 0.33 근처였습니다. 학습 후 Full 모델이 test 128장 중 5장을 틀렸는데 그중 4장이 각반병과 녹병 사이의 혼동이었고, 둘 다 작은 갈색 반점이라 224로 줄이면 반점이 몇 픽셀만 남는 것이 원인으로 보였습니다.

### 자세히
- 0.218이 무작위보다 낮은 이유: 새 Head가 한쪽 클래스로 몰아서 예측했다.
- Baseline이 있어야 "88%"나 "96%"가 얼마나 올라간 숫자인지 말할 수 있다.
- 오류 분석: Full test 0.961, 틀린 5장 중 4장이 각반병 ↔ 녹병.
- 실습 노트북 검증셋에서 틀린 4장도 전부 각반병을 녹병으로 본 경우였고, 확신도는 0.61~0.71로 낮았다.

개선 방향:
- 두 클래스 사진을 더 모은다.
- 잎을 가까이 찍은 사진을 늘린다.
- 더 높은 해상도로 파인튜닝해 반점을 더 많은 픽셀로 보게 한다.

### 꼬리질문
#### 틀린 예측의 확신도가 낮았다는 건 어떻게 활용할 수 있나요?
확신도가 일정 값보다 낮은 예측은 사람이 다시 확인하게 하는 식으로 운영할 수 있습니다. 모델이 헷갈리는 경계에 있는 사진이라는 신호이기 때문입니다.

## Q. CLIP 모델에 대해 설명해 보세요.
id: clip-overview
type: 개념
level: 기본
tags: CLIP, Multimodal, 임베딩 공간
source: 6. CLIP › 인코더 두 개, 공간 하나

### 30초 답변
CLIP은 이미지 인코더와 텍스트 인코더 두 개를 함께 학습시켜 이미지와 문장을 같은 임베딩 공간에 놓는 모델입니다. 두 인코더가 같은 길이의 벡터를 내고, 길이를 1로 정규화한 뒤 내적하면 코사인 유사도가 되어 이미지와 문장이 얼마나 맞는지 점수로 나옵니다. 이 점수로 클래스 목록에 묶이지 않는 zero-shot 분류나 문장으로 이미지를 찾는 검색을 합니다.

### 자세히
- Image Encoder: ViT(논문에서는 ResNet도 사용). [CLS] 출력을 선형 투영한다.
- Text Encoder: Transformer. 문장 끝 토큰 출력을 선형 투영한다.
- 공통 차원: `clip-vit-base-patch32`는 512.
- 논문: Radford et al., 2021. 인터넷에서 모은 약 4억 개의 이미지-문장 쌍으로 학습.
- CLIP은 그림이나 문장을 생성하지 않는다. 임베딩과 점수를 검색과 분류에 쓸 뿐이다.
- 실험: 이미지 6장과 짝 문장 6개의 유사도 표에서 대각선(짝)이 0.26~0.35로 가장 컸고 나머지는 0.08~0.19였다. 6장 모두 자기 짝을 골랐다.

### 꼬리질문
#### 분류 데이터 대신 이미지-문장 쌍으로 학습하면 무엇이 좋은가요?
클래스 이름 하나는 "cat"밖에 알려 주지 않지만, 설명 문장은 자세, 장소, 색까지 담습니다. 사람이 라벨을 붙일 필요 없이 인터넷에서 대량으로 모을 수 있다는 점도 큽니다.

#### 이미지 임베딩과 문장 임베딩의 길이를 왜 같게 맞추나요?
내적으로 직접 비교하려면 같은 공간, 같은 차원에 있어야 하기 때문입니다. 각 인코더 끝의 선형 투영층이 서로 다른 Hidden size를 공통 차원으로 맞춥니다.

## Q. CLIP의 대조 학습 목표(loss)를 설명해 보세요.
id: clip-contrastive-loss
type: 개념
level: 기본
tags: Contrastive Learning, InfoNCE, Cross-entropy
source: 6. CLIP › 대조 학습 (Contrastive Learning)

### 30초 답변
한 배치에 이미지 N장과 각 이미지의 설명 문장 N개를 넣고, 모든 쌍의 유사도로 N×N 표를 만듭니다. i번째 이미지의 정답은 i번째 문장이라 대각선이 정답이고, 라벨은 0부터 N-1까지의 번호면 충분합니다. 행 방향으로 이미지에서 문장을 고르는 cross-entropy와 열 방향으로 문장에서 이미지를 고르는 cross-entropy를 구해 평균을 냅니다.

### 자세히
```python
img = F.normalize(img_emb, dim=-1)        # (N, 512)
txt = F.normalize(txt_emb, dim=-1)        # (N, 512)
logits = logit_scale * img @ txt.T        # (N, N), 행 = 이미지, 열 = 문장
labels = torch.arange(N)                  # 대각선이 정답
loss = (F.cross_entropy(logits, labels) + F.cross_entropy(logits.T, labels)) / 2
```

- 정답 쌍의 점수는 높이고, 같은 배치의 나머지 쌍은 상대적으로 낮추도록 두 인코더를 함께 학습한다.
- 따로 클래스 라벨이 필요 없다.
- Hugging Face `CLIPModel`은 `return_loss=True`를 주면 이 양방향 손실을 계산해 준다.
- 실험: 앞의 6쌍으로 계산하니 손실이 0.0001로 거의 0이었다. 이미 잘 구분하는 쌍들이라 그렇다.

### 꼬리질문
#### 왜 한 방향이 아니라 양방향으로 손실을 구하나요?
CLIP은 이미지로 문장을 찾는 쪽과 문장으로 이미지를 찾는 쪽을 모두 씁니다. 두 방향 모두 정답이 1등이 되도록 학습해야 양쪽 검색이 다 잘 됩니다.

#### 한 배치에 같은 상품 사진이 두 장 들어가면 어떻게 되나요?
사실상 맞는 쌍인데도 대각선이 아니라는 이유로 오답 취급을 받아 점수가 낮아지도록 학습됩니다. 그래서 데이터를 만들 때 중복을 걸러 내는 정제가 중요합니다.

## Q. CLIP의 Zero-shot 분류는 어떻게 동작하나요?
id: clip-zeroshot
type: 개념
level: 기본
tags: Zero-shot, CLIP, 분류
source: 6. CLIP › Zero-shot 분류와 검색

### 30초 답변
후보 클래스 이름을 "A photo of a {label}." 같은 문장으로 만들어 텍스트 임베딩을 구하고, 이미지 임베딩과 가장 유사한 문장을 고르는 방식입니다. 유사도에 logit_scale을 곱해 softmax하면 클래스별 확률처럼 쓸 수 있습니다. 새 클래스가 생겨도 모델을 다시 학습하지 않고 후보 문장만 바꾸면 됩니다.

### 자세히
```python
texts = [f"A photo of a {c}." for c in classes]
inputs = processor(text=texts, images=image, return_tensors="pt", padding=True)
probs = model(**inputs).logits_per_image.softmax(dim=1)   # (1, 클래스 수)
```

- 텍스트 임베딩을 미리 계산해 두면, 문장에서 만들어진 가중치를 가진 Linear 분류기와 같다.
- `pipeline("zero-shot-image-classification", ...)`에 `candidate_labels`를 넘겨도 된다.
- 논문 보충: ViT-L/14(336px) CLIP은 ImageNet zero-shot 76.2%로, ImageNet으로 학습한 ResNet-50과 비슷했다.

### 꼬리질문
#### "Zero-shot"은 정확히 무엇이 0개라는 뜻인가요?
그 작업의 라벨된 학습 예시를 하나도 쓰지 않았다는 뜻입니다. 다만 사전학습한 4억 쌍 안에 비슷한 개념이 있었을 수 있어서, 개념 자체를 처음 본다는 의미는 아닙니다.

#### 같은 모델로 문장 검색은 어떻게 하나요?
방향만 반대입니다. 이미지 임베딩을 미리 계산해 두고 검색 문장의 임베딩과 내적해 점수순으로 정렬합니다. 실습에서 "something you can ride"로 자전거 사진, "water"로 항구 사진을 라벨 없이 찾았습니다.

## Q. CLIP에서 logit_scale(temperature)은 왜 필요한가요?
id: clip-logit-scale
type: 이유
level: 심화
tags: Temperature, logit_scale, Softmax
source: 6. CLIP › 인코더 두 개, 공간 하나

### 30초 답변
코사인 유사도는 -1에서 1 사이라 후보끼리 차이가 작고, 그대로 softmax하면 확률이 거의 고르게 나오기 때문입니다. 실습에서 짝 문장 0.35, 다른 문장 0.16을 그대로 softmax하니 확률이 0.20 대 0.16 정도로 구분이 안 됐는데, logit_scale 100을 곱하면 차이 0.19가 19로 벌어져 짝 문장이 1.00을 가져갔습니다. 이 값은 고정이 아니라 학습되는 파라미터입니다.

### 자세히
| | 짝 문장 | 다른 문장 | softmax 결과 |
|---|---|---|---|
| 코사인 그대로 | 0.35 | 0.16 | 0.20 대 0.16 (6개 후보) |
| × logit_scale 100 | 35 | 16 | 짝 문장 1.00 |

- logit_scale = 1 / temperature. Hugging Face `logits_per_image`가 바로 `유사도 × logit_scale`이다.
- 논문 보충: log 값으로 학습하며, 초기값은 temperature 0.07에 해당하고 100을 넘지 않게 제한했다. 실습 모델 값이 100인 것도 이 상한이다.
- 짝 유사도가 0.3 근처인 것은 문제가 아니다. 점수는 절댓값보다 후보끼리의 비교로 의미가 있다.

### 꼬리질문
#### 값이 너무 크거나 작으면 어떻게 되나요?
너무 작으면 확률이 평평해져 정답과 오답을 가르는 학습 신호가 약해집니다. 너무 크면 가장 높은 점수 하나에 확률이 몰려 과하게 확신하고, 학습이 불안정해질 수 있습니다.

## Q. 대조 학습에서 배치 크기가 왜 중요한가요?
id: clip-batch-size
type: 이유
level: 심화
tags: Batch size, Negative, Contrastive Learning
source: 6. CLIP › 대조 학습 (Contrastive Learning)

### 30초 답변
대조 학습에서는 같은 배치의 다른 샘플이 오답(negative)이 되므로, 배치가 클수록 한 번에 구분해야 할 오답이 많아지기 때문입니다. 오답이 많으면 정답과 헷갈릴 만한 어려운 오답도 섞여 들어와 더 세밀하게 구분하도록 학습됩니다. CLIP이 32,768이라는 큰 배치를 쓴 이유입니다.

### 자세히
- 배치 N이면 샘플마다 오답 N-1개. N×N 표 전체가 학습 신호다.
- 작은 배치의 예: 6쌍으로 계산한 손실이 0.0001이었다. 오답 5개가 모두 쉬워서 배울 게 거의 없는 상태다.
- 비용: N×N 표와 큰 배치 활성값 때문에 메모리가 크고, 여러 GPU의 임베딩을 모아서 표를 만들어야 한다.
- 옷 검색 프로젝트는 배치 64였다.

### 꼬리질문
#### 큰 배치가 어려울 때는 어떻게 하나요?
SigLIP처럼 softmax 대신 쌍마다 독립적인 sigmoid 손실을 쓰면 배치 전체로 정규화할 필요가 없어 큰 배치 의존도가 줄어듭니다. 이전 배치의 임베딩을 큐에 저장해 오답으로 다시 쓰는 방법(MoCo)도 있습니다.

## Q. CLIP 임베딩 공간을 직접 들여다보니 어떤 점이 보였나요?
id: clip-modality-gap
type: 경험
level: 심화
tags: Modality gap, PCA, 임베딩
source: 6. CLIP › 임베딩 공간을 들여다보면

### 30초 답변
이미지 6장과 짝 문장 6개의 512차원 임베딩을 PCA로 2차원에 찍었더니, 짝끼리 붙어 있을 거라는 예상과 달리 이미지는 왼쪽, 문장은 오른쪽에 따로 모였습니다. 코사인 평균도 짝끼리 0.302인데 이미지끼리 0.406, 문장끼리 0.524로 오히려 같은 종류끼리 더 비슷했습니다. 이 현상을 modality gap이라고 부르고, 그래도 그룹 안의 상대적 순서는 맞아서 검색은 동작합니다.

### 자세히
| 비교 | 코사인 평균 |
|---|---|
| 짝 (이미지-문장) | 0.302 |
| 이미지끼리 | 0.406 |
| 문장끼리 | 0.524 |

- PCA 첫 번째 축이 이미지냐 문장이냐를 갈랐다.
- 이미지 하나를 기준으로 문장들을 줄 세우면 짝이 1등이 된다. 점수는 후보끼리 비교할 때 의미가 있다.
- 보충: 이 현상은 Liang et al., 2022, "Mind the Gap"에서 이름 붙여 분석되었다.

### 꼬리질문
#### modality gap이 실제로 문제가 되는 경우는 언제인가요?
이미지 하나로 문장 후보들을 비교하는 것처럼 같은 종류 후보끼리 순위를 매길 때는 괜찮습니다. 이미지와 문장을 한 검색 결과에 섞어서 순위를 매기거나, 고정된 유사도 기준값으로 맞다 아니다를 판정할 때는 같은 종류끼리 점수가 높게 나와 왜곡될 수 있습니다.

## Q. Zero-shot에서 "A photo of a {label}." 같은 프롬프트를 쓰는 이유는 무엇인가요?
id: clip-prompt
type: 개념
level: 기본
tags: Prompt engineering, Prompt ensembling, Zero-shot
source: 6. CLIP › Zero-shot 분류와 검색

### 30초 답변
CLIP이 학습한 인터넷 캡션은 단어 하나보다 문장 형태가 많아서, 학습 분포와 비슷한 틀을 쓰면 성능이 오르기 때문입니다. 단어 하나만 넣으면 뜻이 여러 개인 단어를 구분하기도 어렵습니다. CLIP 논문은 이 틀만으로 ImageNet 정확도를 1.3%p 올렸고, 여러 틀을 섞어 쓰는 프롬프트 앙상블로 더 올렸습니다.

### 자세히
- 프롬프트 엔지니어링: 작업에 맞는 문맥을 넣는다. 예: 반려동물 데이터면 "A photo of a {label}, a type of pet."
- 프롬프트 앙상블: 클래스마다 여러 틀의 텍스트 임베딩을 평균 내 하나의 클래스 임베딩으로 쓴다. 논문은 ImageNet에서 80개 틀로 3.5%p를 더 올렸다.
- 앙상블은 임베딩 단계에서 평균하므로 추론 비용이 늘지 않는다.
- 실험: 콩잎 데이터에서는 라벨 그대로든 "a photo of a ..." 틀이든 0.336으로 같았다. 모델이 개념 자체를 모르면 틀로 해결되지 않는다.

### 꼬리질문
#### 단어 하나만 넣으면 어떤 문제가 생기나요?
"crane"처럼 뜻이 여러 개인 단어는 새인지 기계인지 알 수 없습니다. "A photo of a ..." 같은 틀이 있으면 적어도 사진 속 대상이라는 문맥이 생깁니다.

## Q. CLIP zero-shot이 잘 안 됐던 사례를 말해 보세요.
id: clip-zeroshot-limit
type: 경험
level: 기본
tags: CLIP, Zero-shot, 도메인
source: 6. CLIP › 학습하지 않은 분야에서는

### 30초 답변
ViT 파인튜닝으로 0.961이 나온 콩잎 질병 데이터에 CLIP zero-shot을 그대로 써 보니 정확도가 0.336으로 무작위 수준이었고, 128장을 거의 전부 angular leaf spot으로 찍었습니다. 증상을 자세히 묘사한 문장으로 바꾸자 0.273으로 오히려 떨어졌고 정상 잎은 한 장도 예측하지 않았습니다. CLIP은 잎 사진이라는 건 알아도 각반병과 녹병의 차이 같은 전문 지식은 인터넷 이미지-문장에서 배우지 못한 것입니다.

### 자세히
| 후보 문장 | test 정확도 |
|---|---|
| 라벨 그대로 | 0.336 |
| "a photo of a ..." 틀 | 0.336 |
| 증상 묘사 문장 | 0.273 |
| (비교) ViT Full Fine-tuning | 0.961 |

- 후보 문장에 따라 답이 쉽게 바뀐다. 녹병 잎 한 장을 "healthy / rust / car"와 비교하면 rust 0.89로 맞히지만, "healthy / rust / angular leaf spot"이면 angular leaf spot 0.91을 골랐다.
- 문장 검색도 마찬가지였다. "갈색 반점이 있는 상한 콩잎"으로 9장을 검색하니 점수가 0.305~0.356에 몰려 정상 잎이 1, 2위였다.

### 꼬리질문
#### 일상 이미지에서도 점수를 믿기 어려운 경우가 있었나요?
있었습니다. "an animal"로 검색했을 때 개 사진이 1위이긴 했지만 0.241 대 0.238로 콩잎 사진과 거의 차이가 없었습니다. 1등과 2등의 점수 차이가 작으면 결과를 믿기 어렵습니다.

#### 이런 분야에서는 어떻게 해야 하나요?
라벨된 데이터가 있다면 분류 모델을 파인튜닝하거나, CLIP 자체를 그 분야의 이미지-문장 데이터로 다시 학습해야 합니다.

## Q. CLIP과 일반 이미지 분류 모델은 어떻게 다르고, 언제 무엇을 쓰나요?
id: clip-vs-classifier
type: 비교
level: 심화
tags: CLIP, Open vocabulary, 분류 모델
source: 6. CLIP › 분류 모델의 한계

### 30초 답변
일반 분류 모델은 학습한 클래스 목록 안에서만 답하는 닫힌 어휘 모델이라, 새 클래스가 생기면 Head를 바꿔 다시 학습해야 합니다. CLIP은 아무 문장과도 점수를 낼 수 있는 열린 어휘 모델이라 후보 문장만 바꾸면 되고, 문장으로 이미지를 찾는 검색에도 쓸 수 있습니다. 다만 전문 분야나 미세한 클래스 구분처럼 정확도가 중요하고 라벨 데이터가 있으면 분류 모델 파인튜닝이 낫습니다.

### 자세히
| | 일반 분류 모델 | CLIP |
|---|---|---|
| 학습 데이터 | 이미지 + 클래스 번호 | 이미지 + 설명 문장 |
| 출력 | 정해진 클래스의 확률 | 이미지와 문장의 유사도 |
| 새 클래스 | Head 교체 후 재학습 | 후보 문장만 추가 |
| 검색 | 불가 | 문장 ↔ 이미지 검색 가능 |
| 전문 분야 정확도 | 파인튜닝하면 높음 (콩잎 0.961) | 낮을 수 있음 (콩잎 0.336) |

- 절충안: CLIP 이미지 임베딩 위에 Linear 분류기만 학습하는 linear probe.

### 꼬리질문
#### 라벨 데이터가 거의 없는 새 분야라면 어떻게 시작하나요?
먼저 CLIP zero-shot으로 가능성을 확인하고, 1등과 2등 점수 차이가 작거나 한 클래스로 몰리면 소량이라도 라벨을 모아 linear probe나 파인튜닝으로 넘어갑니다. 콩잎처럼 무작위 수준이면 zero-shot만으로는 쓸 수 없습니다.

## Q. 이미지 검색 평가 지표인 Recall@K와 MRR을 설명해 보세요.
id: clip-retrieval-metrics
type: 개념
level: 기본
tags: Recall@K, MRR, 검색 평가
source: 7. 프로젝트: CLIP 파인튜닝으로 옷 검색하기

### 30초 답변
Recall@K는 질의마다 정답이 상위 K개 안에 들어 있는 비율이고, MRR은 정답 순위의 역수를 질의 전체에 대해 평균 낸 값입니다. 정답이 1등이면 1, 2등이면 0.5, 3등이면 약 0.33을 받습니다. Recall@K는 정답이 K등 안에만 들면 몇 등이든 같게 보고, MRR은 정답이 위에 있을수록 점수를 더 줍니다.

### 자세히
```python
ranks = np.array(...)            # 질의별 정답 순위 (1부터)
recall_at_k = (ranks <= k).mean()
mrr = (1 / ranks).mean()
```

| 질의 | 정답 순위 | Recall@1 | Recall@5 | 1/rank |
|---|---|---|---|---|
| A | 1 | 1 | 1 | 1.0 |
| B | 2 | 0 | 1 | 0.5 |
| C | 7 | 0 | 0 | 0.14 |

- K가 커질수록 Recall@K는 같거나 커진다.
- 프로젝트에서는 사람이 쓴 질의 27개, 후보 상품 71장으로 쟀다.

### 꼬리질문
#### 정답이 하나뿐일 때 Recall@1은 무엇과 같나요?
1등이 정답인 비율이므로 검색을 분류처럼 본 top-1 정확도와 같습니다.

#### 정답이 여러 개인 검색이라면 어떤 지표를 쓰나요?
정답 여러 개의 순위를 모두 반영하는 mAP나, 순위가 낮을수록 점수를 깎는 NDCG를 많이 씁니다.

## Q. CLIP을 옷 검색용으로 파인튜닝해 본 결과는 어땠나요?
id: clip-finetune-retrieval
type: 경험
level: 심화
tags: CLIP, Fine-tuning, 검색, 과적합
source: 7. 프로젝트: CLIP 파인튜닝으로 옷 검색하기

### 30초 답변
한국어와 영어를 이해하는 CLIP을 H&M 상품 531개의 사진과 영어 설명으로 8에폭 학습했더니, 검증 손실이 1.548에서 5에폭 1.008까지 내려갔다가 8에폭 1.053으로 다시 올라 과적합이 시작됐습니다. 질의 27개로 평가한 Recall@1은 영어가 0.852에서 0.815로 떨어지고 한국어는 0.778에서 0.815로 올랐습니다. 하지만 질의 하나가 0.037에 해당해서 이 차이는 질의 한 개 수준이라 좋아졌다거나 나빠졌다고 단정하기 어렵습니다.

### 자세히
설정: `Bingsu/clip-vit-base-patch32-ko`, 상품 단위로 train 531 / val 49 / test 71, 배치 64, lr 1e-5, 8에폭(64 step), 88초.

| 질의 | 모델 | Recall@1 | Recall@5 | MRR |
|---|---|---|---|---|
| 영어 | 학습 전 → 후 | 0.852 → 0.815 | 1.000 → 1.000 | 0.907 → 0.880 |
| 한국어 | 학습 전 → 후 | 0.778 → 0.815 | 0.963 → 0.963 | 0.865 → 0.875 |

- 학습 손실은 0.13까지 내려갔다. 그런데 노트북은 마지막 에폭을 저장해서, 저장된 모델은 검증 손실이 가장 낮은 모델이 아니었다.
- 질의별로는 "저녁 파티에 입을 검정 원피스"는 1, 2위가 모두 검정 원피스가 됐지만, "편안한 회색 맨투맨"은 회색 맨투맨이 2위에서 4위로 내려갔다.

### 꼬리질문
#### 다음에 해 볼 실험은 무엇인가요?
검증 손실이 가장 낮은 에폭을 저장하고, 이미지 인코더를 고정(`freeze_vision`)해 학습할 파라미터를 줄이며, 학습 데이터와 평가 질의를 늘리는 것입니다. 질의가 많아야 0.037 단위보다 작은 차이도 판단할 수 있습니다.

#### 데이터를 상품 단위로 나눈 이유는 무엇인가요?
한 상품에 사진과 문장 쌍이 여러 개 있어서, 쌍 단위로 나누면 학습한 상품이 시험에도 나와 성능이 부풀려지기 때문입니다.
