# 학습 기법
blog: blog_딥러닝심화/post.md
blog_title: 딥러닝 심화 정리: nn.Module, 옵티마이저, 정규화, 과적합 방지, 가중치 초기화

## Q. PyTorch의 nn.Module은 무엇이고, __init__과 forward에는 각각 무엇을 쓰나요?
id: train-nn-module
type: 개념
level: 기본
tags: PyTorch, nn.Module, forward, Sequential
source: 1. torch.nn › __init__과 forward

### 30초 답변
nn.Module은 PyTorch에서 모델을 클래스로 정의할 때 상속하는 기본 클래스입니다. `__init__`에는 쓸 레이어(부품)를 선언하고, `forward`에는 입력이 그 레이어들을 거치는 계산 순서를 적습니다. 레이어를 `self.xx`로 선언하면 파라미터가 자동으로 등록되어 `model.parameters()`, `model.to(device)`, `state_dict()`로 한 번에 다룰 수 있습니다.

### 자세히
- raw tensor로 짜면 층마다 weight, bias를 직접 만들고 `requires_grad`, GPU 이동, 저장을 하나씩 챙겨야 한다. nn.Module은 이걸 `model` 하나로 묶는다.
- `__init__`에서 `super().__init__()`을 먼저 호출해야 한다. 빠뜨리면 서브모듈을 등록할 때 에러가 난다.
- 실행은 `model(x)`로 한다. 내부에서 `forward`가 호출된다.

```python
class MLP(nn.Module):
    def __init__(self):
        super().__init__()
        self.fc1 = nn.Linear(2, 2)
        self.fc2 = nn.Linear(2, 1)
    def forward(self, x):
        return torch.sigmoid(self.fc2(torch.sigmoid(self.fc1(x))))
```

### 꼬리질문
#### model.forward(x)로 직접 부르면 안 되는 이유는 무엇인가요?
`model(x)`는 `__call__`을 거치면서 forward 전후에 등록된 hook 같은 부가 처리를 함께 실행합니다. forward를 직접 부르면 이 과정을 건너뛰기 때문에 `model(x)`로 부르는 것이 원칙입니다.

#### 클래스 대신 nn.Sequential을 쓰면 무엇이 다른가요?
레이어를 순서대로 이어 붙이기만 할 때는 Sequential이 간단합니다. 대신 파라미터 이름이 `0.weight`, `2.weight`처럼 번호로 붙고, 중간에 분기나 잔차 연결처럼 순서가 복잡한 구조는 표현할 수 없어서 그때는 클래스로 짭니다.

## Q. nn.Linear(2, 4)의 weight 모양은 어떻게 되고, 파라미터는 몇 개인가요?
id: train-linear-weight
type: 개념
level: 기본
tags: nn.Linear, 파라미터 수, Fully Connected
source: 1. torch.nn › nn.Linear: weight 모양이 뒤집혀 있다

### 30초 답변
nn.Linear(2, 4)는 입력 2개를 받아 출력 4개를 내는 완전연결층이고, weight 모양은 `(출력 수, 입력 수)`인 (4, 2)입니다. 내부에서 y = xWᵀ + b로 전치해서 곱하기 때문입니다. 파라미터는 가중치 8개와 편향 4개를 합쳐 12개입니다.

### 자세히
- 직접 만든 `W1`은 `(입력, 출력)` 모양이라 `x @ W1`로 계산했다. nn.Linear는 반대로 저장한다.
- 파라미터 수 = 입력 × 출력 + 출력(편향)

| 레이어 | weight | bias | 합 |
|---|---|---|---|
| Linear(2, 4) | 4×2 = 8 | 4 | 12 |
| Linear(4, 1) | 1×4 = 4 | 1 | 5 |

- 두 층을 이은 모델의 학습 파라미터는 총 17개이고, 하나도 직접 만들 필요가 없다.

### 꼬리질문
#### 모델 전체 파라미터 수는 코드로 어떻게 세나요?
`sum(p.numel() for p in model.parameters())`로 셉니다. 학습되는 것만 세려면 `if p.requires_grad` 조건을 붙입니다.

#### 활성화 함수 레이어에도 파라미터가 있나요?
ReLU, Sigmoid 같은 활성화 함수는 학습할 값이 없어서 파라미터가 0개입니다. 그래서 Sequential로 짠 모델의 파라미터 목록에서 ReLU 자리 번호는 빠집니다.

## Q. epoch, batch size, iteration을 각각 설명해 보세요.
id: train-epoch-batch-iteration
type: 개념
level: 기본
tags: epoch, batch, iteration, step
source: 2. 데이터 로딩 › epoch, batch, iteration

### 30초 답변
batch size는 한 번 업데이트할 때 쓰는 데이터 개수이고, iteration(step)은 파라미터 업데이트 1회, epoch는 전체 데이터를 한 바퀴 다 도는 것입니다. 1 epoch당 iteration 수는 데이터 수를 batch size로 나눈 값을 올림한 것입니다. 데이터 1000개를 batch 32로 자르면 32번이고, 마지막 batch에는 8개만 들어갑니다.

### 자세히
```
iteration/epoch = ceil(N / batch_size)
총 iteration    = iteration/epoch × epoch 수
```

- 1000 ÷ 32 = 31.25 → 32 step. 남은 8개는 버리지 않고 작은 batch 하나로 돈다.
- 1200개, batch 100, 20 epoch → epoch당 12번, 총 240번 업데이트
- 총 150번 업데이트, 10 epoch → epoch당 15 step
- "업데이트를 몇 번 했냐"가 iteration이라고 생각하면 헷갈리지 않는다.

### 꼬리질문
#### 마지막 작은 batch를 버리고 싶으면 어떻게 하나요?
DataLoader에 `drop_last=True`를 주면 됩니다. batch 크기가 들쭉날쭉하면 문제가 되는 경우, 예를 들어 BatchNorm이 크기 1인 batch를 받는 상황을 피할 때 씁니다.

## Q. Batch GD, SGD, Mini-batch GD를 비교해 보세요.
id: train-gd-variants
type: 비교
level: 기본
tags: Batch GD, SGD, Mini-batch, 경사하강법
source: 2. 데이터 로딩 › 세 가지 전략

### 30초 답변
Batch GD는 전체 데이터를 보고 한 번 업데이트해서 경로가 매끄럽지만 한 번 업데이트가 너무 느립니다. SGD는 데이터 1개마다 업데이트해서 빠르지만 경로가 들쭉날쭉하고 GPU 병렬 계산을 제대로 못 씁니다. Mini-batch GD는 그 중간으로, 적당한 노이즈와 병렬 효율을 함께 얻어서 가장 널리 씁니다.

### 자세히
| | Batch GD | SGD | Mini-batch |
|---|---|---|---|
| 업데이트당 데이터 | 전체 | 1개 | 32~256개 등 |
| 경로 | 매끄러움 | 많이 흔들림 | 중간 |
| 1 epoch 업데이트 수 | 1번 | N번 | N / batch |

- 직접 비교(데이터 64개, y = 2x + 1, 30 epoch): 학습률 0.01에서 Batch GD는 손실 1.215, SGD는 0.009. 같은 한 바퀴에 SGD는 64번, Batch GD는 1번 업데이트하기 때문이다.
- 학습률을 0.1로 올리면 Batch GD도 0.028까지 따라온다.
- 요즘 "SGD"라고 하면 대부분 mini-batch SGD를 뜻한다. `torch.optim.SGD`도 마찬가지다.

### 꼬리질문
#### SGD의 노이즈가 장점이 되기도 하나요?
네, 매번 정확한 기울기만 따라가면 local minimum이나 안장점에서 못 빠져나오기 쉬운데, 적당한 노이즈가 섞이면 거기서 튕겨 나와 더 좋은 지점을 찾기도 합니다. 이 노이즈가 일종의 암묵적 규제 역할을 한다는 해석도 있습니다.

## Q. batch size는 어떻게 정하나요? 32나 64를 많이 쓰는 이유는 무엇인가요?
id: train-batch-size
type: 이유
level: 심화
tags: batch size, 학습률, sharp minimum, GPU
source: 2. 데이터 로딩 › batch size는 왜 32일까

### 30초 답변
GPU는 2의 거듭제곱 단위로 계산할 때 효율적이고, 32는 그중 실용적인 가장 작은 크기입니다. 또 batch가 너무 작으면 노이즈가 커서 수렴이 불안정하고, 너무 크면 지나치게 안정적이라 sharp minimum에 빠져 일반화가 나빠질 수 있어서 중간 값이 균형이 맞습니다. AlexNet 이후 연구들이 32~128을 써 온 관행도 있습니다. 실제로는 GPU 메모리가 허용하는 범위에서 고르고, batch를 바꾸면 학습률도 함께 조정합니다.

### 자세히
| batch 작음 | batch 큼 |
|---|---|
| 기울기 노이즈 큼, 수렴 불안정 | 기울기 안정, 업데이트 횟수 적음 |
| GPU 병렬 효율 낮음 | 메모리 많이 씀 |
| 일반화에 유리하다는 보고 | sharp minimum 위험 |

- batch를 k배로 키우면 lr도 √k~k배 정도 키우는 게 보통이다. k배로 키우는 선형 규칙은 보통 warmup과 함께 쓴다.

### 꼬리질문
#### GPU 메모리 때문에 batch를 키울 수 없으면 어떻게 하나요?
gradient accumulation을 씁니다. 작은 batch로 여러 번 `backward()`를 해서 기울기를 누적한 다음 한 번만 `step()`하면 큰 batch와 비슷한 업데이트가 됩니다. 단, BatchNorm 통계는 여전히 작은 batch 기준으로 계산됩니다.

#### sharp minimum이 왜 일반화에 나쁜가요?
뾰족한 바닥은 파라미터가 조금만 어긋나도 손실이 크게 오릅니다. 학습 데이터와 테스트 데이터의 손실 지형은 조금씩 다르기 때문에, 평평한 바닥이 그 차이에 덜 민감하다는 설명입니다.

## Q. Dataset과 DataLoader는 각각 어떤 역할을 하나요?
id: train-dataloader
type: 개념
level: 기본
tags: Dataset, DataLoader, TensorDataset, shuffle
source: 2. 데이터 로딩 › TensorDataset과 DataLoader

### 30초 답변
Dataset은 데이터와 정답을 짝지어 i번째 샘플을 꺼낼 수 있게 묶는 역할이고, DataLoader는 그 Dataset을 batch 크기만큼 잘라 하나씩 꺼내 주는 역할입니다. `shuffle=True`로 두면 epoch마다 순서를 섞어 줍니다. DataLoader로 도는 for문 한 바퀴가 1 epoch입니다.

### 자세히
```python
dataset = TensorDataset(X_train_tensor, y_train_tensor)    # X, y를 한 묶음으로
loader = DataLoader(dataset, batch_size=32, shuffle=True)  # 32개씩 잘라서 꺼냄

for X_batch, y_batch in loader:    # 1 epoch
    ...
```

- 직접 Dataset을 만들 때는 `__len__`(전체 개수)과 `__getitem__`(i번째 샘플 반환)을 구현한다.
- 자주 쓰는 인자: `batch_size`, `shuffle`, `drop_last`, `num_workers`(병렬로 데이터를 불러올 프로세스 수)

### 꼬리질문
#### 학습 데이터는 왜 섞고, 검증 데이터는 왜 섞지 않나요?
학습 데이터가 정답 순서대로 정렬되어 있으면 batch마다 한쪽 클래스만 들어가 기울기가 치우치기 때문에 섞습니다. 검증은 업데이트를 하지 않고 전체 성능만 재므로 섞을 필요가 없고, 순서를 고정해 두면 결과 비교도 쉽습니다.

#### 이미지처럼 데이터가 커서 메모리에 다 못 올리면 어떻게 하나요?
`__getitem__` 안에서 파일 경로만 들고 있다가 필요한 샘플만 그때그때 읽어 오게 짭니다. 그러면 전체를 메모리에 올리지 않고 batch 단위로 불러올 수 있습니다.

## Q. 기본 경사하강법이 잘 안 되는 손실 지형에는 어떤 것들이 있나요?
id: train-loss-landscape
type: 개념
level: 기본
tags: local minimum, saddle point, plateau, ravine
source: 3. 옵티마이저 › 경사하강법이 막히는 지형

### 30초 답변
대표적으로 네 가지입니다. 주변보다만 낮은 가짜 바닥인 local minimum, 한 방향은 내리막이고 다른 방향은 오르막인 saddle point, 넓고 평평해 기울기가 거의 0인 plateau, 그리고 한 방향만 가팔라서 좌우로 진동만 하는 좁고 긴 골짜기(ravine)입니다. 기본 경사하강법은 w ← w − lr·g라 기울기가 작아지면 걸음도 작아져서 이런 곳에서 멈추거나 헤맵니다.

### 자세히
| 지형 | 문제 |
|---|---|
| Local Minimum | 기울기 0이라 그냥 멈춤 |
| Saddle Point | 기울기 0. 고차원에서는 local minimum보다 훨씬 많음 |
| Plateau | 기울기 거의 0, 학습이 사실상 멈춤 |
| Ravine | 가파른 축으로 진동, 완만한 축으로는 거의 못 감. 실제 신경망에서 가장 흔함 |

- 해결 방향: 과거 이동 방향을 기억하거나(Momentum), 파라미터마다 보폭을 다르게 주거나(AdaGrad, RMSprop), 둘 다(Adam).

### 꼬리질문
#### 고차원에서는 왜 local minimum보다 saddle point가 더 많나요?
기울기가 0인 지점이 local minimum이 되려면 모든 방향이 동시에 오르막이어야 하는데, 차원이 수백만 개면 그럴 가능성이 매우 낮습니다. 대부분은 일부 방향이 내리막인 안장점이라, 실제로는 local minimum보다 안장점과 평지에서 느려지는 문제가 더 큽니다.

## Q. Momentum 옵티마이저를 설명해 보세요.
id: train-momentum
type: 개념
level: 기본
tags: Momentum, NAG, 옵티마이저
source: 3. 옵티마이저 › Momentum: 굴러오던 방향을 기억한다

### 30초 답변
Momentum은 언덕을 굴러가는 공처럼 이전 이동 속도를 일정 비율 유지한 채 이번 기울기를 더해서 움직이는 방법입니다. 같은 방향의 힘은 쌓이고 반대 방향의 힘은 상쇄되기 때문에, 골짜기에서 좌우 진동은 줄고 가야 할 방향으로는 속도가 붙습니다. 관성 계수 β는 보통 0.9를 씁니다.

### 자세히
```
v ← β·v − lr·g
w ← w + v
```

- β = 0이면 그냥 SGD, 0.9면 최근 약 10걸음(1/(1−β)) 평균 방향, 0.99면 약 100걸음 평균
- β가 너무 크면 방향 전환이 굼뜨고 최저점을 지나쳐 출렁인다.
- PyTorch: `torch.optim.SGD(params, lr=0.01, momentum=0.9)`

### 꼬리질문
#### NAG(Nesterov)는 Momentum과 무엇이 다른가요?
지금 자리가 아니라 관성으로 도착할 자리(w + β·v)에서 기울기를 잽니다. 도착지가 오르막이면 미리 덜 가게 되어 과속(overshooting)이 줄고, PyTorch에서는 `nesterov=True`로 켭니다.

#### 관성이 있으면 local minimum을 넘을 수 있나요?
작은 턱 정도는 굴러오던 힘으로 넘을 수 있습니다. 다만 항상 보장되는 것은 아니고 β와 학습률에 따라 달라집니다.

## Q. Momentum의 β 값을 바꿔 보면 어떤 차이가 나던가요?
id: train-momentum-beta-exp
type: 경험
level: 심화
tags: Momentum, 관성 계수, 실험
source: 3. 옵티마이저 › Momentum: 굴러오던 방향을 기억한다

### 30초 답변
가운데에 작은 턱이 있는 언덕에서 학습률 0.1은 그대로 두고 β만 바꿔 100번 업데이트해 봤습니다. β가 0, 0.5, 0.6일 때는 턱 앞의 작은 웅덩이(x ≈ −1.88)에 갇혔고, 0.7부터는 관성으로 턱을 넘어 진짜 바닥(x = 3)까지 갔습니다. 반대로 β = 0.99는 관성이 너무 세서 100번 뒤에도 바닥을 지나친 x = 5.50 근처에서 출렁이고 있었습니다.

### 자세히
| β | 100번 업데이트 뒤 |
|---|---|
| 0, 0.5, 0.6 | x ≈ −1.88 가짜 바닥에서 멈춤 |
| 0.7 이상 | 턱을 넘어 x = 3 도달 |
| 0.99 | x ≈ 5.50, 바닥을 지나쳐 출렁임 |

- 관성이 부족하면 갇히고, 지나치면 멈추지 못한다. 그래서 적당한 값인 0.9를 기본으로 쓴다.
- 같은 골짜기(f = 0.05x² + y²) 경로 비교에서도 Momentum은 시원하게 가지만 최저점을 지나쳤다가 돌아왔다.

### 꼬리질문
#### β를 키우면 학습률도 함께 조정해야 하나요?
관성이 1/(1−β)걸음만큼 쌓이므로 β를 키우면 실제 보폭이 커지는 효과가 있습니다. 그래서 β를 크게 잡을 때는 학습률을 조금 낮춰야 발산이나 큰 출렁임을 피할 수 있습니다.

## Q. AdaGrad와 RMSProp의 차이를 설명해 보세요.
id: train-adagrad-rmsprop
type: 비교
level: 심화
tags: AdaGrad, RMSProp, 적응적 학습률
source: 3. 옵티마이저 › 적응적 학습률: 파라미터마다 보폭을 다르게

### 30초 답변
둘 다 기울기를 그 크기로 나눠서 파라미터마다 보폭을 다르게 주는 적응적 학습률 방법입니다. AdaGrad는 기울기 제곱을 처음부터 계속 더하기만 해서 시간이 지나면 분모가 커져 보폭이 0으로 죽고 학습이 멈춥니다. RMSProp은 이를 지수이동평균으로 바꿔 최근 기울기 크기만 반영하므로 보폭이 죽지 않고, 깊은 신경망에서도 쓸 수 있습니다.

### 자세히
```
AdaGrad: s ← s + g²
RMSProp: s ← β·s + (1−β)·g²
공통:    w ← w − lr / (√s + ε) · g
```

- 직접 측정(골짜기, lr 0.2): SGD는 첫 걸음에 가파른 축 0.80, 완만한 축 0.036으로 20배 넘게 차이 났다. RMSProp은 두 축 모두 0.63을 움직였다.
- 첫 걸음에 s = 0.1·g²이라 보폭이 lr/√0.1 ≈ 0.63으로 기울기와 상관없이 같아진다.
- RMSProp은 보폭만 조절하고 방향 관성은 없다.

### 꼬리질문
#### AdaGrad가 여전히 유리한 경우가 있나요?
드물게 나오는 특성의 기울기는 누적값이 작아 상대적으로 큰 보폭을 받기 때문에, 희소한 특성이 많은 선형 모델 같은 문제에서는 쓸 만합니다. 깊은 신경망처럼 오래 학습하는 경우에는 보폭이 죽는 문제가 커서 잘 쓰지 않습니다.

#### ε은 왜 들어가나요?
s가 0에 가까울 때 0으로 나누는 것을 막기 위한 아주 작은 값입니다. Adam의 기본값은 1e-8입니다.

## Q. Adam 옵티마이저의 동작 원리를 설명해 보세요.
id: train-adam
type: 개념
level: 기본
tags: Adam, Momentum, RMSProp, 하이퍼파라미터
source: 3. 옵티마이저 › Adam: 둘 다

### 30초 답변
Adam은 Momentum처럼 기울기의 지수이동평균(1차 모멘트 m)으로 방향을 기억하고, RMSProp처럼 기울기 제곱의 지수이동평균(2차 모멘트 v)으로 파라미터마다 보폭을 조절합니다. 즉 방향과 보폭을 둘 다 챙기는 방법이고, 2014년에 나온 뒤 사실상 기본 옵티마이저가 되었습니다. 기본값은 lr 1e-3, β1 0.9, β2 0.999, ε 1e-8입니다.

### 자세히
```
m ← β1·m + (1−β1)·g        # 방향
v ← β2·v + (1−β2)·g²       # 크기
m̂ = m / (1−β1ᵗ),  v̂ = v / (1−β2ᵗ)
w ← w − lr · m̂ / (√v̂ + ε)
```

| 이름 | 기본값 | 의미 |
|---|---|---|
| β1 | 0.9 | 방향 관성, 최근 약 10걸음 평균 |
| β2 | 0.999 | 크기 기억, 약 1000스텝 평균 |

- 골짜기 경로 실험에서 Adam은 고른 보폭으로 가다가 관성 때문에 끝에서 살짝 맴돌았다.

### 꼬리질문
#### Adam은 옵티마이저 상태로 무엇을 저장하나요?
파라미터마다 m과 v를 하나씩 들고 있어서 파라미터 수의 두 배만큼 메모리를 더 씁니다. 그래서 학습을 끊었다 이어 하려면 `optimizer.state_dict()`도 모델과 함께 저장해야 합니다.

## Q. Adam에서 bias correction은 왜 필요한가요?
id: train-adam-bias-correction
type: 이유
level: 심화
tags: Adam, bias correction, 모멘트
source: 보충 (블로그 밖 내용)

### 30초 답변
m과 v를 0으로 시작하기 때문에 학습 초반에는 지수이동평균이 실제 값보다 0 쪽으로 크게 치우칩니다. 특히 β2가 0.999로 커서 v는 초반에 거의 0에 가깝습니다. 그래서 1−βᵗ로 나눠서 이 치우침을 보정하고, t가 커지면 이 값이 1에 가까워져 보정 효과가 자연스럽게 사라집니다.

### 자세히
- 첫 step: m₁ = (1−β1)·g = 0.1·g → 실제 기울기의 10%밖에 안 된다.
- 보정: m̂₁ = m₁ / (1−0.9) = g

| t | 1−0.9ᵗ | 1−0.999ᵗ |
|---|---|---|
| 1 | 0.1 | 0.001 |
| 10 | 약 0.65 | 약 0.01 |
| 1000 | 약 1 | 약 0.63 |

- 보정이 없으면 m과 v가 모두 작게 잡히는데, 둘의 치우침 정도가 달라서 초반 보폭이 의도와 다르게 나온다.

### 꼬리질문
#### 보정을 하는데도 Adam에 warmup을 쓰는 이유는 무엇인가요?
bias correction은 평균의 치우침만 고칠 뿐, 초반에 몇 개 안 되는 기울기로 추정한 v 자체가 불안정하다는 문제는 남습니다. 그래서 초반 몇 step은 작은 학습률로 시작하는 warmup을 함께 쓰는 경우가 많습니다.

## Q. Adam이 항상 SGD보다 좋은가요?
id: train-adam-vs-sgd
type: 비교
level: 심화
tags: Adam, SGD, 일반화, 옵티마이저 선택
source: 보충 (블로그 밖 내용)

### 30초 답변
그렇지는 않습니다. Adam은 학습률에 덜 민감하고 초반 수렴이 빨라서 기본 선택으로 무난하고, Transformer 계열에서는 사실상 표준입니다. 반면 이미지 분류 CNN에서는 학습률 스케줄을 잘 맞춘 SGD + Momentum이 최종 일반화 성능에서 비슷하거나 더 낫다는 보고가 많습니다. 그래서 빠르게 실험할 때는 Adam(AdamW), 검증된 레시피가 있는 비전 모델은 SGD + Momentum을 많이 씁니다.

### 자세히
| | SGD + Momentum | Adam / AdamW |
|---|---|---|
| 학습률 민감도 | 큼, 스케줄 튜닝 필요 | 작음 |
| 수렴 속도 | 느린 편 | 빠름 |
| 메모리 | 파라미터당 상태 1개 | 2개 |
| 주 사용처 | CNN 이미지 분류 레시피 | Transformer, LLM, 빠른 실험 |

- 학습률 기본값도 다르다: Adam 1e-3, SGD 1e-2 ~ 1e-1. Adam에 맞춘 학습률을 SGD에 그대로 쓰면 너무 작고, 반대로 SGD 값을 Adam에 쓰면 발산하기 쉽다.

### 꼬리질문
#### 옵티마이저를 바꿀 때 무엇을 같이 바꿔야 하나요?
학습률을 다시 찾아야 하고, weight decay 값도 옵티마이저마다 적정 범위가 다릅니다. 옵티마이저만 바꾸고 나머지를 그대로 두면 공정한 비교가 되지 않습니다.

## Q. Adam의 L2 규제와 AdamW의 weight decay는 무엇이 다른가요?
id: train-adamw
type: 비교
level: 심화
tags: AdamW, weight decay, L2 규제, Adam
source: 4. torch.optim › 많이 쓰는 세 가지 (보충 포함)

### 30초 답변
Adam에 L2 규제를 걸면 λw가 기울기에 더해진 뒤 √v로 나눠지기 때문에, 기울기가 컸던 파라미터일수록 감쇠가 약해져 규제가 의도대로 걸리지 않습니다. AdamW는 감쇠를 기울기와 분리해서 적응적 보폭 계산과 상관없이 가중치에 직접 lr·λ·w만큼 줄입니다. SGD에서는 두 방식이 같지만 Adam에서는 달라지고, 그래서 Adam 계열에서는 AdamW를 쓰는 게 일반적입니다.

### 자세히
```
Adam + L2:  g' = g + λw  →  m, v 계산에 g' 사용  →  w ← w − lr·m̂/(√v̂+ε)
AdamW:      m, v는 g로만 계산  →  w ← w − lr·(m̂/(√v̂+ε) + λw)
```

- PyTorch `Adam(weight_decay=...)`은 L2 방식, `AdamW(weight_decay=...)`가 분리 방식이다.
- 권장 범위: Adam의 weight decay 0 ~ 0.01, AdamW 0.01 ~ 0.1. AdamW 기본값은 0.01이다.

### 꼬리질문
#### SGD에서는 왜 L2와 weight decay가 같은가요?
SGD는 기울기에 학습률만 곱하므로, λw를 기울기에 더하든 가중치에서 직접 빼든 업데이트 식이 w ← (1 − lr·λ)w − lr·g로 같아집니다. Adam은 기울기를 √v̂로 나누는 단계가 끼어 있어서 두 방식이 갈라집니다.

## Q. 학습률 스케줄러는 무엇이고 왜 쓰나요?
id: train-lr-scheduler
type: 개념
level: 기본
tags: 학습률, 스케줄러, param_groups
source: 보충 (블로그 밖 내용)

### 30초 답변
학습률 스케줄러는 학습이 진행되는 동안 학습률을 정해진 규칙에 따라 바꿔 주는 도구입니다. 초반에는 큰 학습률로 빠르게 내려가고, 후반에는 작은 학습률로 최저점 근처에서 튕기지 않고 세밀하게 수렴하게 하려는 목적입니다. 대표적으로 StepLR, CosineAnnealing, 검증 손실이 정체되면 줄이는 ReduceLROnPlateau가 있습니다.

### 자세히
| 스케줄러 | 방식 |
|---|---|
| StepLR | 정해진 epoch마다 일정 비율로 감소 |
| CosineAnnealingLR | 코사인 곡선을 따라 0 근처까지 부드럽게 감소 |
| ReduceLROnPlateau | 검증 지표가 patience 동안 안 좋아지면 감소 |
| OneCycleLR | 올렸다가 내리는 한 번의 주기 |

```python
scheduler = torch.optim.lr_scheduler.StepLR(optimizer, step_size=10, gamma=0.1)
for epoch in range(epochs):
    train_one_epoch()        # 안에서 optimizer.step()
    scheduler.step()
```

- 내부적으로는 `optimizer.param_groups[0]["lr"]` 값을 바꾸는 것이다. 이 값을 직접 바꿔도 다음 `step()`부터 적용된다.

### 꼬리질문
#### 학습률이 너무 크거나 작으면 각각 어떤 증상이 보이나요?
너무 크면 손실이 출렁이거나 NaN으로 발산하고, 너무 작으면 손실이 매우 천천히 줄거나 평지에서 멈춘 것처럼 보입니다. 학습률은 가장 중요한 하이퍼파라미터라 보통 먼저 맞춥니다.

#### 층마다 학습률을 다르게 줄 수도 있나요?
옵티마이저에 파라미터 그룹을 나눠 넘기면 그룹마다 lr을 따로 줄 수 있습니다. 파인튜닝에서 사전학습된 앞쪽 층은 작게, 새로 붙인 뒤쪽 층은 크게 줄 때 씁니다.

## Q. 학습률 warmup은 왜 하나요?
id: train-warmup
type: 이유
level: 심화
tags: warmup, 학습률, Transformer, large batch
source: 보충 (블로그 밖 내용)

### 30초 답변
학습 초반에는 가중치가 무작위 상태라 기울기가 크고 방향도 불안정하며, Adam의 v 같은 통계도 아직 제대로 쌓이지 않았습니다. 이때 큰 학습률로 시작하면 초반 몇 step에 모델이 크게 망가지거나 발산할 수 있습니다. 그래서 처음 일정 step 동안 학습률을 0 근처에서 목표값까지 서서히 올리는 warmup을 씁니다.

### 자세히
```
step < warmup:  lr = base_lr × step / warmup
이후:           cosine 등으로 감소
```

- Transformer, 큰 batch 학습에서 특히 중요하다. 원래 Transformer 논문도 4000 step warmup을 썼다.
- batch를 키우면서 학습률을 비례해 키울 때, warmup 없이 큰 학습률로 바로 시작하면 정확도가 떨어진다는 보고가 있다.
- 흔한 조합: 선형 warmup + cosine decay

### 꼬리질문
#### warmup 길이는 어떻게 정하나요?
보통 전체 학습 step의 몇 퍼센트 정도로 잡는 경우가 많고, 초반 손실이 튀지 않을 만큼이면 충분합니다. 너무 길면 큰 학습률 구간이 짧아져 학습이 느려집니다.

## Q. model.train()과 model.eval()은 무엇이 다르고, torch.no_grad()와는 어떻게 다른가요?
id: train-train-eval
type: 비교
level: 기본
tags: train, eval, no_grad, Dropout, BatchNorm
source: 4. torch.optim › train 모드와 eval 모드

### 30초 답변
`model.train()`과 `model.eval()`은 BatchNorm과 Dropout처럼 학습과 평가에서 다르게 동작하는 레이어의 모드를 바꿉니다. eval 모드에서 Dropout은 꺼지고, BatchNorm은 현재 batch 통계 대신 학습 중 모아 둔 running 통계를 씁니다. `torch.no_grad()`는 레이어 동작과 상관없이 기울기 계산 그래프를 만들지 않게 해서 메모리와 시간을 아끼는 것이라, 평가할 때는 둘을 함께 씁니다.

### 자세히
| | 바꾸는 것 | 기울기 계산 |
|---|---|---|
| `model.eval()` | Dropout 끔, BN은 running 통계 사용 | 여전히 계산됨 |
| `torch.no_grad()` | 레이어 동작은 그대로 | 계산 안 함 |

```python
model.eval()
with torch.no_grad():
    val_loss = criterion(model(X_val), y_val)
model.train()   # 다음 epoch 학습 전에 되돌리기
```

### 꼬리질문
#### eval()을 빼먹으면 어떤 일이 생기나요?
평가 중에도 Dropout이 뉴런을 꺼서 예측이 매번 달라지고, BatchNorm은 평가 batch의 통계를 써서 결과가 batch 구성에 따라 흔들립니다. 평가가 끝나고 train()으로 되돌리지 않으면 반대로 학습 때 Dropout이 꺼진 채로 돌게 됩니다.

## Q. 입력 데이터를 정규화(표준화)하는 이유는 무엇인가요?
id: train-input-normalization
type: 이유
level: 기본
tags: 입력 정규화, Standardization, Min-Max, 스케일링
source: 5. 정규화 › 입력 정규화

### 30초 답변
특성마다 숫자 범위가 다르면 큰 특성의 가중치는 조금만 움직여도 손실이 크게 바뀌어서, 손실 지형이 좁고 긴 골짜기 모양이 됩니다. 그러면 경사하강법이 가파른 축으로 지그재그를 그리며 느리게 수렴합니다. 평균 0, 표준편차 1로 맞추면 등고선이 둥글어져 거의 곧장 내려갈 수 있어서 학습이 빠르고 안정적이 됩니다.

### 자세히
- 예: 방 개수(1~5)와 넓이(30~200㎡)로 집값 예측 → 넓이 축이 훨씬 민감하다.

| 방법 | 수식 | 언제 |
|---|---|---|
| Min-Max | (x − min) / (max − min) | 이미지 픽셀처럼 범위가 명확할 때 |
| Standardization | (x − μ) / σ | 가장 기본 |
| Robust Scaling | (x − 중앙값) / IQR | 이상치가 많을 때 |
| Log 변환 | log(1 + x) | 한쪽으로 심하게 치우친 분포 |

- 같은 경사하강법, 같은 20걸음으로 비교하면 정규화 전에는 지그재그로 헤매고 후에는 거의 곧장 내려간다.

### 꼬리질문
#### Normalization과 Regularization은 어떻게 다른가요?
한국어로는 둘 다 "정규화"라고 부르지만, Normalization은 입력이나 중간층 값의 눈금을 맞춰 학습을 안정시키는 것이고, Regularization(규제)은 모델에 제약을 걸어 과적합을 막는 것입니다. L2와 Dropout은 규제, BatchNorm과 입력 표준화는 정규화입니다.

## Q. 정규화의 평균과 표준편차는 왜 학습 데이터에서만 구하나요?
id: train-scaler-leakage
type: 이유
level: 기본
tags: 데이터 누수, fit_transform, 전처리, 표준화
source: 5. 정규화 › 입력 정규화

### 30초 답변
검증·테스트 데이터는 모델이 처음 보는 데이터라는 가정으로 성능을 재는 것인데, 그 통계까지 섞어 쓰면 평가 데이터의 정보가 학습 과정에 새어 들어가는 데이터 누수가 됩니다. 그러면 성능이 실제보다 좋게 나옵니다. 그래서 데이터를 먼저 나누고, 학습 데이터에서만 평균과 표준편차를 구해 나머지에는 그 값을 그대로 적용합니다.

### 자세히
1. train / val / test로 먼저 나눈다.
2. 학습 데이터에서만 μ, σ를 계산한다.
3. 그 값을 검증·테스트 데이터에 그대로 적용한다.
4. 배포할 때도 그 값을 파일로 저장해 같이 내보낸다.

```python
scaler.fit_transform(X_train)   # 통계 계산 + 변환
scaler.transform(X_val)         # 변환만
```

- scikit-learn의 `fit_transform` / `transform` 구분이 이 규칙을 코드로 강제한 것이다.

### 꼬리질문
#### 배포할 때 통계를 같이 저장해야 하는 이유는 무엇인가요?
실제 서비스 입력도 학습 때와 같은 기준으로 변환해야 모델이 배운 분포와 맞기 때문입니다. 서비스 데이터로 새로 통계를 구하면 같은 값이라도 다른 숫자로 바뀌어 예측이 틀어집니다.

## Q. Batch Normalization은 어떻게 동작하나요?
id: train-batchnorm
type: 개념
level: 기본
tags: BatchNorm, 정규화 층, γ, β
source: 5. 정규화 › Batch Normalization

### 30초 답변
Batch Normalization은 각 층의 값을 현재 batch의 평균과 분산으로 정규화해서 층을 지나도 값의 범위가 적절하게 유지되게 하는 층입니다. 특성(채널)별로 batch 평균을 빼고 표준편차로 나눈 뒤, 학습되는 γ를 곱하고 β를 더합니다. 동시에 추론 때 쓸 전체 평균·분산을 지수이동평균으로 모아 둡니다. 2015년에 제안됐습니다.

### 자세히
```
x̂ = (x − μ_B) / √(σ²_B + ε)
y = γ·x̂ + β                  # γ, β는 역전파로 학습
running_mean, running_var ← 지수이동평균으로 갱신
```

```python
model = nn.Sequential(
    nn.Linear(10, 64),
    nn.BatchNorm1d(64),   # 특성 수
    nn.ReLU(),
    nn.Linear(64, 1),
)
```

- MLP는 Linear → BN → ReLU, CNN은 Conv → BN → ReLU가 관례다. BN의 β가 bias 역할을 하므로 앞의 Conv는 `bias=False`로 둔다.

### 꼬리질문
#### 평균 0, 분산 1로 맞춘 뒤 γ와 β를 다시 곱하고 더하는 이유는 무엇인가요?
모든 층을 강제로 평균 0, 분산 1로 묶으면 표현력이 제한될 수 있습니다. 예를 들어 sigmoid 앞에서는 거의 선형 구간만 쓰게 됩니다. γ와 β를 학습하게 두면 필요하면 원래 분포로도 되돌릴 수 있어서, 정규화의 안정성과 표현력을 함께 얻습니다.

## Q. Batch Normalization은 학습할 때와 추론할 때 어떻게 다르게 동작하나요?
id: train-bn-train-inference
type: 비교
level: 기본
tags: BatchNorm, running statistics, eval
source: 5. 정규화 › Batch Normalization

### 30초 답변
학습할 때는 현재 batch의 평균과 분산으로 정규화하고, 그와 동시에 running mean과 running variance를 지수이동평균으로 갱신합니다. 추론할 때는 batch 통계 대신 이렇게 모아 둔 running 통계를 고정해서 씁니다. 그래야 같은 입력이면 함께 들어온 다른 샘플과 상관없이 항상 같은 출력이 나옵니다.

### 자세히
| | train 모드 | eval 모드 |
|---|---|---|
| 정규화 통계 | 현재 batch의 μ, σ² | running mean / var |
| running 통계 갱신 | 함 | 안 함 |
| 출력 | batch 구성에 따라 달라짐 | 샘플마다 결정적 |

- PyTorch는 `model.eval()`로 전환한다. 안 하면 평가 결과가 batch 구성에 따라 흔들린다.
- running 통계 갱신 비율은 `momentum` 인자로 정하며 기본값은 0.1이다.

### 꼬리질문
#### 학습 데이터와 실제 서비스 데이터의 분포가 다르면 어떤 문제가 생기나요?
running 통계는 학습 데이터 기준이라, 서비스 데이터의 분포가 다르면 정규화 결과가 어긋나 성능이 떨어집니다. 이럴 때 새 도메인 데이터로 BN 통계만 다시 추정하는 방법을 쓰기도 합니다.

#### batch 크기가 1이면 학습 때 BN은 어떻게 되나요?
샘플이 하나면 batch 분산을 제대로 구할 수 없어서, PyTorch의 BatchNorm1d는 학습 모드에서 에러를 냅니다. 작은 batch에서는 통계가 불안정하므로 Group Norm이나 Layer Norm을 대신 고려합니다.

## Q. Batch Normalization이 학습에 효과적인 이유는 무엇인가요?
id: train-bn-why
type: 이유
level: 심화
tags: BatchNorm, internal covariate shift, 기울기 소실, 규제
source: 5. 정규화 › Batch Normalization

### 30초 답변
층을 지날 때마다 값의 분포가 넓어지거나 한쪽으로 쏠리는데, 층이 깊을수록 이 변화가 쌓여 학습이 느려지거나 멈춥니다. BN은 매 층의 값을 적절한 범위로 되돌려서 신호가 사라지거나 포화 구간에 갇히는 것을 막습니다. 그 결과 깊은 망도 학습되고, 큰 학습률을 안전하게 쓸 수 있고, 초기화에 덜 민감해지며, batch마다 통계가 조금씩 달라서 생기는 잡음이 가벼운 규제 효과도 냅니다.

### 자세히
- 원 논문은 효과를 internal covariate shift(앞 층이 바뀌면서 뒤 층 입력 분포가 계속 바뀌는 현상)를 줄이는 것으로 설명했다.
- 이후 연구에서는 분포 변화 감소보다 손실 지형을 매끄럽게 만들어 기울기가 더 예측 가능해지는 것이 핵심이라는 반론이 나왔다.

| 장점 | 단점 |
|---|---|
| 깊은 망 학습, 큰 학습률 가능 | batch가 작으면 통계 불안정 |
| 초기화에 덜 민감 | train/eval 동작이 달라 실수 여지 |
| 약한 규제 효과 | 시퀀스 모델에 적용이 까다로움 |

### 꼬리질문
#### BN을 쓰면 Dropout이 필요 없나요?
BN에도 약한 규제 효과가 있어서 Dropout을 줄이거나 빼는 경우가 많습니다. 다만 BN의 규제 효과는 부수적인 것이라 과적합이 심하면 다른 규제가 여전히 필요합니다.

## Q. BatchNorm을 넣고 뺀 실험에서 어떤 차이를 봤나요?
id: train-bn-exp
type: 경험
level: 심화
tags: BatchNorm, sigmoid, 기울기 소실, 실험
source: 5. 정규화 › Batch Normalization

### 30초 답변
8층 sigmoid MLP에 입력 스케일을 일부러 5배로 키우고 학습률 0.5로, BatchNorm만 넣고 빼서 비교했습니다. BN이 없으면 loss가 처음부터 0.6929에 붙어 200 step 내내 움직이지 않았고 정확도는 51%로 찍는 수준이었습니다. BN을 넣으면 같은 모델이 0.0075까지 내려가고 정확도 100%가 됐습니다.

### 자세히
| | BN 없음 | BN 있음 |
|---|---|---|
| 200 step 뒤 loss | 0.6929 (그대로) | 0.0075 |
| 정확도 | 51% | 100% |

- 0.693 = ln 2는 무엇을 넣든 0.5라고 답할 때의 BCE 값이다. 모델이 아무것도 못 배운 상태라는 뜻이다.
- 슬라이드에는 이때 마지막 층 활성값의 표준편차가 0.0006까지 떨어져 신호가 사실상 죽었다고 나와 있었다.

### 꼬리질문
#### loss가 ln 2에 고정되어 있다는 걸 보고 무엇을 의심해야 하나요?
이진분류에서 모델이 모든 입력에 0.5만 출력하고 있다는 뜻이라, 신호가 층을 지나며 사라졌거나 출력이 포화되어 기울기가 흐르지 않는지 의심해야 합니다. 층별 활성값과 기울기 크기를 찍어 보고, 초기화, 입력 스케일, 학습률, 정규화 층을 차례로 점검합니다.

## Q. Batch Norm, Layer Norm, Instance Norm, Group Norm은 무엇이 다른가요?
id: train-norm-types
type: 비교
level: 심화
tags: LayerNorm, InstanceNorm, GroupNorm, RMSNorm
source: 5. 정규화 › 다른 정규화 층들

### 30초 답변
네 가지 모두 평균과 분산으로 정규화하지만 평균을 내는 범위가 다릅니다. Batch Norm은 batch 전체에서 같은 채널끼리, Layer Norm은 샘플 하나의 모든 채널, Instance Norm은 샘플 하나의 채널 하나, Group Norm은 샘플 하나 안의 채널 묶음 단위로 계산합니다. Batch Norm만 batch 크기의 영향을 받고, 나머지는 샘플 하나 안에서 계산하므로 batch가 작아도 안정적입니다.

### 자세히
| 기법 | 평균 내는 범위 | batch 영향 | 주 사용처 |
|---|---|---|---|
| Batch Norm | batch 전체 × 같은 채널 | 있음 | CNN 이미지 분류 |
| Layer Norm | 샘플 하나의 모든 채널 | 없음 | Transformer, RNN |
| Instance Norm | 샘플 하나 × 채널 하나 | 없음 | 스타일 변환, 생성 |
| Group Norm | 샘플 하나 × 채널 묶음 | 없음 | batch를 크게 못 잡는 탐지·분할 |
| RMS Norm | Layer Norm에서 평균 빼기 생략 | 없음 | 최신 LLM, 더 빠름 |

- batch가 작으면 BN의 평균이 batch마다 크게 출렁인다. 학습지에서 batch 크기를 바꿔 batch 10개의 평균을 찍어 보면 작을수록 흔들림이 컸다.

### 꼬리질문
#### Instance Norm이 스타일 변환에 맞는 이유는 무엇인가요?
샘플 하나의 채널별 평균과 분산에는 색감이나 대비 같은 이미지 고유의 스타일 정보가 많이 담겨 있습니다. 이걸 채널마다 정규화해서 지우면 원래 스타일을 걷어 내고 새 스타일을 입히기 쉬워집니다.

#### Group Norm은 Layer Norm, Instance Norm과 어떤 관계인가요?
그룹 수를 1로 두면 모든 채널을 한 번에 묶으니 Layer Norm과 같고, 그룹 수를 채널 수와 같게 두면 Instance Norm과 같습니다. 그 사이에서 채널 묶음 크기를 조절하는 방법입니다.

## Q. 과적합이란 무엇이고, 어떤 신호로 알아차리나요?
id: train-overfitting
type: 개념
level: 기본
tags: 과적합, 검증 손실, 일반화
source: 6. 규제 › 과적합의 신호

### 30초 답변
과적합은 모델이 학습 데이터의 규칙이 아니라 잡음까지 외워 버려서 새 데이터에서 성능이 떨어지는 상태입니다. 학습 loss는 계속 줄어드는데 검증 loss는 어느 시점부터 다시 올라가는 것이 대표적인 신호입니다. 데이터는 적은데 파라미터가 많거나 입력 특성이 지나치게 많을 때 잘 생깁니다.

### 자세히
- 다항식 실험: sin 곡선에 잡음을 섞은 train 12개, validation 40개

| 차수 | train 오차 | validation 오차 |
|---|---|---|
| 1차 | 0.192 | 0.240 |
| 3차 | 낮음 | 0.037 (최저) |
| 9차 | 0.011 | 5.411 |

- 9차는 train 점을 거의 다 지나가지만 점 사이에서 제멋대로 꺾인다. train 성능만 보면 9차가 제일 좋아 보이는 게 과적합의 무서운 점이다.

### 꼬리질문
#### 과소적합은 어떻게 구분하나요?
학습 성능부터 낮고 검증 성능도 비슷하게 낮으면 과소적합입니다. 이때는 규제를 걸면 오히려 나빠지므로 모델을 키우거나 더 오래 학습시키는 쪽으로 갑니다.

## Q. bias-variance trade-off를 설명해 보세요.
id: train-bias-variance
type: 개념
level: 심화
tags: bias, variance, 일반화, 모델 복잡도
source: 보충 (블로그 밖 내용)

### 30초 답변
예측 오차는 편향(bias), 분산(variance), 줄일 수 없는 잡음으로 나눠 볼 수 있습니다. 편향은 모델이 너무 단순해서 진짜 패턴을 못 따라가는 오차이고, 분산은 학습 데이터가 조금만 바뀌어도 예측이 크게 달라지는 오차입니다. 모델을 복잡하게 하면 편향은 줄지만 분산이 커지기 때문에, 둘의 합이 가장 작은 복잡도를 찾는 것이 trade-off입니다.

### 자세히
```
기대 오차 = bias² + variance + 잡음
```

| | 높은 bias | 높은 variance |
|---|---|---|
| 상태 | 과소적합 | 과적합 |
| 증상 | train, val 모두 오차 큼 | train 오차 작고 val 오차 큼 |
| 대응 | 모델 키우기, 더 학습 | 데이터 늘리기, 규제, 모델 줄이기 |

- 다항식 실험에 대입하면 1차가 high bias, 9차가 high variance, 3차가 균형점이다.

### 꼬리질문
#### 큰 신경망에도 이 U자 곡선이 그대로 맞나요?
파라미터가 데이터 수보다 훨씬 많은 영역에서는 검증 오차가 다시 내려가는 double descent 현상이 보고되어 있어서, 고전적인 설명이 그대로 맞지 않는 경우가 있습니다. 그래도 규제와 데이터가 분산을 줄인다는 직관은 여전히 유효합니다.

## Q. L2 규제(weight decay)는 어떻게 과적합을 막나요?
id: train-l2
type: 개념
level: 기본
tags: L2 규제, weight decay, Ridge
source: 6. 규제 › L2 규제 (Weight Decay)

### 30초 답변
L2 규제는 손실에 가중치 제곱합에 비례하는 벌점을 더해서 가중치가 커지지 못하게 누릅니다. 업데이트 식으로 풀면 매 step마다 가중치를 일정 비율로 줄이는 꼴이라 weight decay라고도 부릅니다. 큰 가중치들이 서로 상쇄하며 억지로 데이터를 맞추는 걸 막아서 함수가 매끄러워지고 과적합이 줄어듭니다.

### 자세히
```
L_total = L + λ·Σw²
w ← (1 − 2·lr·λ)·w − lr·g
```

- 9차 다항식에 λ만 바꿔 본 결과 (계수 크기 = √Σw²)

| λ | 계수 크기 | validation 오차 |
|---|---|---|
| 0 | 393 | 5.411 |
| 0.001 | 5.0 | 0.068 |
| 0.1 | 2.1 | 0.079 |

- λ가 너무 크면 덜 맞추게 되어 오히려 나빠진다. 과소적합 상태에서는 쓰지 않는다.
- PyTorch는 옵티마이저 `weight_decay` 인자로 건다. 기울기 0에서 lr 0.1, weight_decay 0.1로 `step()`하면 1.0이 0.99가 된다.

### 꼬리질문
#### bias에도 weight decay를 거나요?
bias는 입력과 곱해지지 않아 함수의 복잡도에 주는 영향이 작아서, 보통 bias와 정규화 층의 γ, β에는 weight decay를 빼는 경우가 많습니다. 파라미터 그룹을 나눠 그룹별로 weight_decay를 다르게 줍니다.

## Q. L1 규제와 L2 규제의 차이는 무엇인가요?
id: train-l1-vs-l2
type: 비교
level: 기본
tags: L1, L2, Lasso, Ridge, 희소성
source: 6. 규제 › L2 규제 (Weight Decay)

### 30초 답변
L1은 가중치 절댓값 합을, L2는 제곱합을 벌점으로 더합니다. L2의 벌점 기울기는 2λw라 가중치가 작아질수록 미는 힘도 약해져서 0에 가까워질 뿐 정확히 0이 되지는 않습니다. L1의 벌점 기울기는 크기와 상관없이 λ로 일정해서 작은 가중치를 끝까지 밀어 정확히 0으로 만들기 때문에 특성 선택 효과가 있습니다.

### 자세히
| | L1 (Lasso) | L2 (Ridge) |
|---|---|---|
| 벌점 | λ·Σ\|w\| | λ·Σw² |
| 결과 | 일부 가중치가 정확히 0 (희소) | 모두 골고루 작아짐 |
| 쓸모 | 특성 선택, 압축, 해석 | 일반적인 과적합 방지 |
| 단점 | 0에서 미분 불가 | 불필요한 특성을 완전히 못 지움 |

- 신경망은 레이어가 알아서 특징을 거르기 때문에 희소성이 크게 중요하지 않아, L1은 선형 모델에서 더 많이 쓴다.

### 꼬리질문
#### 둘을 섞어 쓸 수도 있나요?
네, L1과 L2 벌점을 함께 거는 것을 Elastic Net이라고 합니다. 서로 상관된 특성이 많을 때 L1만 쓰면 그중 하나만 임의로 남기는 문제를 L2가 완화해 줍니다.

## Q. Dropout이란 무엇이고, 왜 과적합을 줄이나요?
id: train-dropout
type: 개념
level: 기본
tags: Dropout, 규제, 앙상블
source: 6. 규제 › Dropout

### 30초 답변
Dropout은 학습 중 매 step마다 뉴런의 일부를 무작위로 꺼서 0으로 만드는 규제 기법입니다. 특정 뉴런 몇 개에 지나치게 의존할 수 없게 되므로 여러 뉴런이 골고루 유용한 특징을 배우게 됩니다. 매번 다른 부분 신경망을 학습시키는 셈이라, 여러 모델을 평균 내는 앙상블과 비슷한 효과로도 설명합니다. 평가할 때는 끕니다.

### 자세히
```python
model = nn.Sequential(
    nn.Linear(10, 64),
    nn.ReLU(),
    nn.Dropout(p=0.5),   # 뉴런 절반을 무작위로 끔
    nn.Linear(64, 1),
)
```

- 보통 p는 0.2~0.5. 너무 크면 너무 많이 꺼져서 학습이 잘 안 된다.
- 학습할 때만 적용하고 `model.eval()`이면 자동으로 꺼진다.

### 꼬리질문
#### Dropout을 켜면 학습 loss가 더 높게 나오는 건 정상인가요?
정상입니다. 학습 중에는 일부 뉴런이 꺼진 모델로 loss를 재기 때문에 더 높게 나오고, eval 모드로 잰 검증 loss가 학습 loss보다 낮게 나오기도 합니다.

#### 추론 때 Dropout을 일부러 켜는 경우도 있나요?
Monte Carlo Dropout은 추론 때도 Dropout을 켠 채 여러 번 예측해서, 예측이 얼마나 흔들리는지로 불확실성을 추정합니다.

## Q. Dropout에서 남은 뉴런 값에 1/(1−p)를 곱하는 이유는 무엇인가요?
id: train-inverted-dropout
type: 이유
level: 심화
tags: Dropout, inverted dropout, 스케일링
source: 6. 규제 › Dropout

### 30초 답변
학습 때는 일부 뉴런만 켜지고 평가 때는 전부 켜지기 때문에, 그대로 두면 다음 층이 받는 값의 평균 크기가 학습과 평가에서 달라집니다. 그래서 학습 때 살아남은 뉴런 값을 1/(1−p)배로 미리 키워서 기대값을 맞춥니다. 이렇게 하면 평가 때는 아무 처리 없이 그대로 통과시키면 됩니다. 이를 inverted dropout이라고 하고, PyTorch가 이 방식입니다.

### 자세히
```python
drop = nn.Dropout(p=0.5)
x = torch.ones(1, 8)
drop.train(); drop(x)   # tensor([[0., 0., 2., 0., 0., 0., 2., 2.]])
drop.eval();  drop(x)   # tensor([[1., 1., 1., 1., 1., 1., 1., 1.]])
```

- 기대값: (1−p) × x/(1−p) + p × 0 = x
- 원래 방식은 학습 때 그대로 두고 평가 때 출력에 (1−p)를 곱했다. 결과는 같지만 추론 코드를 건드리지 않아도 되는 inverted 방식이 표준이 됐다.

### 꼬리질문
#### 1/(1−p)를 곱하지 않으면 어떤 문제가 생기나요?
p = 0.5라면 평가 때 다음 층이 받는 값이 학습 때의 약 두 배가 됩니다. 다음 층 가중치는 학습 때 크기에 맞춰져 있으므로 출력이 틀어지고, 특히 BatchNorm이나 활성화 함수의 포화 구간과 겹치면 성능이 크게 떨어집니다.

## Q. Early Stopping은 어떻게 동작하나요?
id: train-early-stopping
type: 개념
level: 기본
tags: Early Stopping, patience, 검증 손실
source: 6. 규제 › Early Stopping

### 30초 답변
Early Stopping은 epoch마다 검증 loss를 확인하다가 더 이상 좋아지지 않으면 학습을 일찍 멈추는 방법입니다. 검증 loss가 최저를 갱신할 때마다 모델을 저장하고, 정해 둔 patience 동안 개선이 없으면 멈춘 뒤 저장해 둔 가장 좋은 모델을 최종 모델로 씁니다. 너무 오래 학습해서 생기는 과적합을 막습니다.

### 자세히
```python
best_loss, wait, patience = float("inf"), 0, 5
for epoch in range(100):
    train_one_epoch()
    val_loss = validate()
    if val_loss < best_loss:
        best_loss, wait = val_loss, 0
        torch.save(model.state_dict(), "best.pt")
    else:
        wait += 1
        if wait >= patience:
            break
model.load_state_dict(torch.load("best.pt", weights_only=True))
```

- 멈춘 시점의 모델이 아니라 best 시점의 모델을 불러오는 것이 핵심이다.

### 꼬리질문
#### Early Stopping 기준으로 테스트 데이터를 쓰면 안 되는 이유는 무엇인가요?
멈출 시점을 고르는 것도 모델 선택이라, 테스트 데이터로 고르면 그 데이터에 맞춰진 결과가 되어 최종 성능이 부풀려집니다. 기준은 검증 데이터로 잡고 테스트는 마지막에 한 번만 씁니다.

## Q. Early Stopping의 patience를 바꿔 보면 결과가 어떻게 달라지던가요?
id: train-patience-exp
type: 경험
level: 심화
tags: Early Stopping, patience, 실험
source: 6. 규제 › Early Stopping

### 30초 답변
작은 신경망의 실제 손실 곡선에 patience만 바꿔 적용해 봤습니다. 검증 loss가 초반에 한 번 크게 출렁였는데, patience 5는 그 출렁임에 epoch 10에서 멈춰서 epoch 5의 0.375가 최종 모델이 됐습니다. patience 20은 출렁임을 견디고 계속 가서 epoch 87에서 0.198까지 내려간 모델을 얻고 epoch 107에서 멈췄습니다. 손실이 거의 절반 차이였습니다.

### 자세히
| patience | 멈춘 epoch | best epoch | best val loss |
|---|---|---|---|
| 5 | 10 | 5 | 0.375 |
| 20 | 107 | 87 | 0.198 |

- 검증 loss는 교과서 그림처럼 매끄럽게 내려가지 않는다.
- 너무 작으면 일시적인 출렁임에 멈추고, 너무 크면 과적합 구간을 쓸데없이 오래 돈다. 슬라이드에서는 10 epoch 정도로 시작하라고 했다.

### 꼬리질문
#### patience 말고 출렁임에 덜 흔들리게 하는 방법이 있나요?
일정 크기 이상 좋아져야 개선으로 치는 최소 개선폭(min_delta)을 두거나, 검증 loss를 몇 epoch 이동평균해서 판단할 수 있습니다. 학습률 스케줄러로 후반 학습률을 낮추면 곡선 자체의 출렁임도 줄어듭니다.

## Q. 과적합을 막는 방법 중 데이터가 가장 강력하다고 하는 이유는 무엇인가요?
id: train-data-regularization
type: 이유
level: 기본
tags: 과적합, 데이터, 규제 순서
source: 6. 규제 › 사실 제일 강력한 건 데이터

### 30초 답변
과적합은 모델이 데이터 수에 비해 너무 많은 것을 외울 수 있을 때 생기는데, 데이터가 많아지면 잡음만 외워서는 전체를 맞출 수 없어 진짜 규칙을 배울 수밖에 없습니다. 다른 규제들은 모델의 자유도를 줄여 우회적으로 막는 반면, 데이터는 문제 자체를 줄입니다. 진짜 데이터를 더 못 모으면 정답이 바뀌지 않는 선에서 기존 데이터를 변형하는 Data Augmentation을 씁니다.

### 자세히
과적합이 생겼을 때 손댈 순서:
1. 학습 데이터를 충분히 확보할 수 있는지 본다.
2. 부족하면 Data Augmentation
3. Early Stopping으로 쓸데없이 오래 학습하는 걸 막는다.
4. Weight Decay로 가중치가 커지는 걸 제한한다.
5. 그래도 계속되면 Dropout을 넣거나 모델 크기를 줄인다.

- 다른 규제는 모델을 덜 맞추게 만드는 대가가 있어서 세게 걸면 과소적합으로 넘어간다. 데이터를 늘리는 쪽은 그런 대가가 적다.

### 꼬리질문
#### 데이터를 늘렸는데도 성능이 안 오른다면 무엇을 의심해야 하나요?
학습 성능부터 낮다면 과적합이 아니라 과소적합이라 모델 용량이나 학습 설정을 봐야 합니다. 또 늘린 데이터가 기존과 거의 같거나 라벨 잡음이 많으면 효과가 작습니다.

## Q. 가중치를 모두 0(또는 같은 값)으로 초기화하면 안 되는 이유는 무엇인가요?
id: train-zero-init
type: 이유
level: 기본
tags: 가중치 초기화, symmetry breaking, 대칭성
source: 7. 가중치 초기화 › 0으로 초기화하면 안 된다

### 30초 답변
같은 층의 뉴런들이 모두 같은 가중치로 시작하면 같은 입력에 대해 똑같은 출력을 내고, 역전파로 받는 기울기도 똑같습니다. 그러면 업데이트도 똑같이 되어 아무리 학습해도 뉴런들이 영원히 같은 값을 유지합니다. 뉴런을 100개 둬도 1개와 다를 게 없어지는 대칭성 문제라서, 무작위 초기화로 대칭을 깨야 합니다.

### 자세히
- 같은 값 → 같은 출력 → 같은 기울기 → 같은 업데이트 → 다시 같은 값
- 이 대칭은 학습만으로는 깨지지 않는다. 시작부터 다른 값을 줘야 한다(symmetry breaking).
- bias는 0으로 시작해도 된다. 가중치가 무작위면 뉴런 출력이 이미 달라서 대칭이 생기지 않는다.

```python
nn.init.zeros_(layer.bias)    # bias는 0으로 괜찮음
```

### 꼬리질문
#### Dropout을 쓰면 대칭이 깨지지 않나요?
학습 중 무작위로 다른 뉴런이 꺼지면 기울기가 달라져 대칭이 조금 깨질 수는 있습니다. 하지만 이걸 기대하는 것은 불안정한 우회책이고, 초기화 단계에서 무작위 값으로 대칭을 깨는 것이 정석입니다.

## Q. 무작위 초기화의 크기(표준편차)를 바꿔 보면 깊은 망에서 어떤 일이 생기던가요?
id: train-init-scale-exp
type: 경험
level: 심화
tags: 가중치 초기화, 기울기 소실, 기울기 폭발, 실험
source: 7. 가중치 초기화 › 무작위로만 뽑으면 될까?

### 30초 답변
층당 뉴런 128개인 ReLU 10층 신경망에 표준편차 1인 입력을 넣고 층마다 활성값의 표준편차를 재 봤습니다. 가중치를 표준편차 0.01로 뽑으면 10층 뒤 약 0.000000000012로 신호가 사라졌고, 표준편차 1로 뽑으면 13억 가까이 폭발했습니다. He 초기화는 10층을 지나도 0.7 근처를 유지했고, Xavier는 서서히 줄어들었습니다.

### 자세히
| 초기화 | 10층 뒤 활성값 표준편차 |
|---|---|
| std 0.01 | 약 10⁻¹¹ (소실) |
| std 1 | 약 10⁹ (폭발) |
| Xavier | 서서히 감소 |
| He | 0.7 근처 유지 |

- 층마다 곱하고 더하는 일이 반복되므로 크기가 조금만 어긋나도 기하급수적으로 불어나거나 줄어든다.
- 역전파 기울기도 같은 원리로 누적되어 기울기 소실·폭발이 된다.
- Xavier가 줄어드는 건 sigmoid/tanh 기준으로 설계되어 ReLU가 절반을 0으로 만드는 것을 고려하지 않았기 때문이다.

### 꼬리질문
#### 실제로 학습을 시작하기 전에 초기화가 괜찮은지 어떻게 확인하나요?
이 실험처럼 무작위 batch를 한 번 통과시켜 층별 활성값과 기울기의 표준편차를 찍어 봅니다. 층을 지나며 크기가 급격히 줄거나 커지면 초기화나 정규화 층을 다시 봅니다.

## Q. LeCun, Xavier, He 초기화를 비교하고, 활성화 함수와 어떻게 짝지어야 하는지 설명해 보세요.
id: train-xavier-he
type: 비교
level: 기본
tags: Xavier, He, LeCun, 가중치 초기화, 활성화 함수
source: 7. 가중치 초기화 › LeCun, Xavier, He

### 30초 답변
세 방법 모두 평균 0인 분포에서 뽑되, 층을 지나도 순전파 활성값과 역전파 기울기의 분산이 일정하게 유지되도록 분산을 정합니다. LeCun은 1/n_in, Xavier는 순전파와 역전파를 함께 고려해 2/(n_in + n_out), He는 ReLU가 절반을 0으로 만드는 것을 감안해 2/n_in을 씁니다. 그래서 ReLU 계열이면 He, sigmoid나 tanh면 Xavier를 씁니다.

### 자세히
| 이름 | 분산 | 짝 |
|---|---|---|
| LeCun | 1 / n_in | 순전파만 고려 |
| Xavier (Glorot) | 2 / (n_in + n_out) | sigmoid, tanh |
| He (Kaiming) | 2 / n_in | ReLU 계열 |

```python
nn.init.kaiming_uniform_(layer.weight, nonlinearity="relu")   # He
nn.init.xavier_uniform_(layer.weight)                         # Xavier
model.apply(init_weights)    # 여러 층에 한 번에 적용
```

- nn.Linear, nn.Conv2d는 기본으로 kaiming_uniform 계열 초기화가 들어 있다. 다만 `a=√5` 옵션이 붙어 교과서의 ReLU용 He 초기화와는 값이 조금 다르다.

### 꼬리질문
#### He 초기화는 왜 분산을 2배로 잡나요?
ReLU는 음수 입력을 0으로 만들어 출력의 절반 정도를 버리므로, 다음 층으로 전달되는 신호의 분산이 대략 절반으로 줄어듭니다. 이 감소를 메우려고 1/n_in 대신 2/n_in을 씁니다.

#### BatchNorm을 쓰면 초기화는 신경 쓰지 않아도 되나요?
BN이 층마다 값의 범위를 다시 맞춰 주므로 초기화에 덜 민감해지는 것은 맞습니다. 하지만 0 초기화의 대칭성 문제는 BN으로 해결되지 않고, BN이 없는 층이나 잔차 경로에서는 초기화가 여전히 중요합니다.

## Q. Gradient clipping은 무엇이고 언제 쓰나요?
id: train-gradient-clipping
type: 개념
level: 심화
tags: gradient clipping, 기울기 폭발, RNN
source: 보충 (블로그 밖 내용)

### 30초 답변
Gradient clipping은 기울기가 정해 둔 크기를 넘으면 잘라서 한 번의 업데이트가 지나치게 커지지 않게 막는 기법입니다. 기울기 폭발로 손실이 갑자기 튀거나 NaN이 되는 것을 막는 데 쓰고, 긴 시퀀스를 다루는 RNN이나 Transformer 학습에서 자주 씁니다. 보통 전체 기울기 벡터의 노름을 기준으로 잘라서 방향은 유지하고 크기만 줄입니다.

### 자세히
```
‖g‖ > c 이면  g ← g · c / ‖g‖
```

```python
loss.backward()
torch.nn.utils.clip_grad_norm_(model.parameters(), max_norm=1.0)
optimizer.step()
```

| 방식 | 동작 | 특징 |
|---|---|---|
| norm 기준 | 전체 노름이 c를 넘으면 비율로 축소 | 방향 유지, 주로 사용 |
| value 기준 | 각 원소를 [−c, c]로 자름 | 방향이 바뀔 수 있음 |

### 꼬리질문
#### clipping은 학습 루프 어디에 넣어야 하나요?
`backward()`로 기울기를 계산한 뒤, `step()`으로 업데이트하기 전에 넣어야 합니다. step 뒤에 넣으면 이미 큰 기울기로 업데이트가 끝난 뒤라 효과가 없습니다.

#### 기울기 소실에도 clipping이 도움이 되나요?
아닙니다. clipping은 큰 기울기를 줄이기만 하므로 작아진 기울기를 키우지는 못합니다. 기울기 소실은 ReLU 계열 활성화, 적절한 초기화, 정규화 층, 잔차 연결로 대응합니다.
