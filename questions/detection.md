# Object Detection
blog: detection_blog/detection_blog.md
blog_title: [CV 3일차] 무엇이 어디에 있는가: YOLO11로 처음 해 본 Object Detection

## Q. Classification, Localization, Detection, Segmentation은 각각 무엇을 출력하나요?
id: det-task-comparison
type: 비교
level: 기본
tags: Object Detection, Classification, Localization, Segmentation
source: 1. Object Detection이란

### 30초 답변
분류는 이미지 전체에 Class 하나를 붙이고, Localization은 거기에 그 객체 하나의 박스 위치를 더합니다. Detection은 이미지 안의 객체 여러 개를 각각 박스와 Class로 찾고, 객체가 없으면 빈 결과를 냅니다. Segmentation은 박스 대신 픽셀 단위로 영역을 나눠서 객체의 실제 윤곽까지 표현합니다.

### 자세히
| 작업 | 출력 | 결과 개수 | 위치 정보 |
|---|---|---|---|
| Classification | 이미지 전체에 class 1개 | 항상 1개 | 없음 |
| Localization | class 1개 + 박스 1개 | 1개 | 박스 |
| Detection | 객체마다 박스 4값 + class + confidence | 0개 이상 | 박스 |
| Segmentation | 픽셀마다 class (또는 객체별 mask) | 픽셀 단위 | 윤곽 |

- 같은 버스 사진에서 분류 모델은 minibus 0.57이라고만 답했고, 탐지 모델은 버스 1개와 사람 4명을 각각 박스로 찾았다.
- 위치를 알면 개수 세기, 객체 사이 거리 계산, 추적이나 분할의 입력으로 넘기기가 가능해진다.

### 꼬리질문
#### 분류 모델로 객체 여러 개를 찾을 수는 없나요?
이미지를 잘게 잘라 조각마다 분류하는 슬라이딩 윈도우 방식이 가능하지만, 위치와 크기를 바꿔 가며 수많은 조각을 분류해야 해서 매우 느립니다. 탐지 모델은 이 과정을 한 번의 forward로 처리하도록 만든 것입니다.

#### 탐지가 쓰이는 대표적인 분야는 어디인가요?
자율주행(차량, 보행자, 신호등), 제조 라인의 결함 위치 찾기, CCTV의 사람과 차량 탐지, 의료 영상의 병변 후보 탐지, 매대 상품 진열 확인 같은 곳입니다.

## Q. 탐지 결과 하나는 어떤 정보로 이루어지고, Confidence Score는 무엇을 뜻하나요?
id: det-output-confidence
type: 개념
level: 기본
tags: Bounding Box, Class, Confidence Score
source: 2. Bounding Box, Class, Confidence Score

### 30초 답변
검출된 객체마다 Bounding Box 좌표 4개, Class ID, Confidence Score가 한 세트로 나옵니다. Confidence Score는 그 박스에 그 Class의 객체가 있을 가능성을 모델이 0~1로 예측한 값입니다. 모델 스스로의 확신일 뿐 정답이라는 보장은 아니고, 맞는지는 평가 단계에서 GT와 비교해 따로 판정합니다.

### 자세히
```
cat (x1, y1, x2, y2, 0.93)
dog (x1, y1, x2, y2, 0.88)
```

- Bounding Box: 객체의 보이는 영역 전체를 포함하는 가장 작은 직사각형. 윤곽까지는 표현하지 못한다.
- Class: 모델 안에서는 정수 ID로 다루고, ID와 이름의 대응은 따로 둔다. 학습할 때 정의한 Class만 출력할 수 있다.
- Ultralytics에서 `boxes.conf`는 높은 순으로 정렬되어 있고, `boxes.cls`는 float Tensor라 이름을 찾을 때 `int()`로 바꿔야 한다.

### 꼬리질문
#### Confidence Score로 두 모델의 정확도를 비교해도 되나요?
안 됩니다. 같은 사진이라도 모델 크기와 가중치에 따라 점수 분포가 달라서, 점수가 높다고 더 정확한 모델이라는 뜻이 아닙니다. 모델 비교는 같은 데이터에서 잰 mAP로 합니다.

#### 모델이 모르는 Class의 객체가 들어오면 어떻게 되나요?
학습한 Class 중 하나로만 답할 수 있어서 가장 비슷해 보이는 Class로 잡거나 아예 놓칩니다. 실습에서 COCO 모델이 알약을 `cup`으로 잡은 것이 그 예입니다.

## Q. 박스를 표현하는 xyxy, xywh, YOLO 정규화 형식의 차이를 설명해 보세요.
id: det-box-format
type: 비교
level: 기본
tags: Bounding Box, xyxy, xywh, 좌표 정규화
source: 2. Bounding Box, Class, Confidence Score › Bounding Box와 좌표계

### 30초 답변
xyxy는 왼쪽 위와 오른쪽 아래 꼭짓점 좌표이고, xywh는 중심점과 너비, 높이입니다. YOLO 라벨은 xywh를 이미지 너비와 높이로 나눈 0~1 값을 씁니다. 같은 박스를 다르게 적은 것일 뿐이지만 라이브러리와 파일마다 쓰는 형식이 달라서 변환을 정확히 해야 하고, IoU처럼 꼭짓점이 필요한 연산은 xyxy로 합니다.

### 자세히
- 이미지 좌표계는 원점이 왼쪽 위, y는 아래로 갈수록 커진다.

| 형식 | 값 | 예 (bus.jpg 810×1080의 가운데 남자) |
|---|---|---|
| xyxy | (x1, y1, x2, y2) | [223, 409, 344, 860] |
| xywh | (cx, cy, w, h) | [284, 635, 121, 452] |
| xywhn | x값은 W로, y값은 H로 나눔 | [0.350, 0.588, 0.150, 0.418] |

```python
cx, cy = (x1 + x2) / 2, (y1 + y2) / 2
w, h = x2 - x1, y2 - y1
x1, y1 = cx - w / 2, cy - h / 2
```

### 꼬리질문
#### 좌표를 0~1로 정규화하면 어떤 점이 좋나요?
이미지를 resize해도 라벨 값이 바뀌지 않아서, 학습 해상도를 바꿀 때 라벨을 다시 계산할 필요가 없습니다. 대신 픽셀로 되돌릴 때 x는 W, y는 H를 곱해야 하고, 둘을 바꿔 쓰면 박스가 어긋납니다.

#### COCO 데이터셋의 박스 형식은 YOLO와 같나요?
다릅니다. COCO JSON은 왼쪽 위 꼭짓점과 너비, 높이를 픽셀 단위로 적는 `[x_min, y_min, w, h]` 형식이라, YOLO로 쓰려면 중심점으로 옮기고 정규화해야 합니다.

## Q. Two-stage detector와 One-stage detector의 차이는 무엇인가요?
id: det-two-vs-one-stage
type: 비교
level: 기본
tags: Two-stage, One-stage, R-CNN, YOLO
source: 3. YOLO와 탐지 모델의 흐름 › Two-stage와 One-stage

### 30초 답변
Two-stage는 먼저 객체가 있을 만한 영역 후보를 만들고, 그 후보마다 Class를 분류하고 박스를 보정하는 두 단계로 탐지합니다. One-stage는 후보 생성 단계 없이 신경망 출력이 곧바로 박스와 Class가 됩니다. 전통적으로 Two-stage가 더 정확하고 One-stage가 더 빠르다고 했지만, 지금 YOLO 계열은 실시간 속도에서 높은 정확도를 냅니다.

### 자세히
| | Two-stage | One-stage |
|---|---|---|
| 흐름 | 영역 후보 → 후보별 분류·박스 보정 | 이미지 전체 → 박스·Class 동시 예측 |
| 대표 모델 | R-CNN, Fast R-CNN, Faster R-CNN | YOLO 계열, SSD, RetinaNet |
| 장점 | 후보를 다시 보므로 위치가 정밀한 편 | forward 한 번, 단계가 적어 빠름 |
| 단점 | 단계가 많아 느림 | 배경 후보가 압도적으로 많아 학습이 까다로움 |

- YOLO(You Only Look Once)는 2016년 Joseph Redmon 등이 발표했다. 이름 그대로 이미지를 한 번만 통과시킨다.

### 꼬리질문
#### 실시간 서비스라면 어떤 쪽을 고르나요?
보통 One-stage를 고르고, 장비 성능에 맞춰 모델 크기(n~x)로 속도와 정확도를 조절합니다. 작은 객체를 아주 정밀하게 찾아야 하고 시간 제약이 느슨하면 Two-stage도 후보가 됩니다.

#### One-stage가 처음에 정확도가 낮았던 이유는 무엇인가요?
영역 후보 단계가 걸러 주지 않으니 이미지의 수많은 위치 대부분이 배경이라, 쉬운 배경 예측이 Loss를 지배했기 때문입니다. RetinaNet의 Focal Loss가 이 불균형을 줄여서 One-stage도 Two-stage에 가까운 정확도를 내게 됐습니다.

## Q. R-CNN, Fast R-CNN, Faster R-CNN은 어떻게 발전했고, RPN은 무엇인가요?
id: det-rcnn-rpn
type: 개념
level: 심화
tags: R-CNN, Faster R-CNN, RPN, RoI Pooling
source: 보충 (블로그 밖 내용)

### 30초 답변
R-CNN은 Selective Search로 영역 후보를 약 2,000개 만들고, 후보마다 이미지를 잘라 CNN에 따로 통과시켜서 매우 느렸습니다. Fast R-CNN은 이미지 전체를 CNN에 한 번만 통과시키고 Feature Map에서 후보 영역을 RoI Pooling으로 잘라 내서 계산을 공유했습니다. Faster R-CNN은 남은 병목인 Selective Search까지 신경망인 RPN(Region Proposal Network)으로 바꿔 전 과정을 하나의 신경망으로 학습하게 했습니다.

### 자세히
| 모델 | 영역 후보 | CNN 통과 | 분류·박스 |
|---|---|---|---|
| R-CNN (2014) | Selective Search | 후보마다 따로 | SVM + 회귀 |
| Fast R-CNN (2015) | Selective Search | 이미지당 1번 + RoI Pooling | 신경망 head |
| Faster R-CNN (2015) | RPN | 이미지당 1번 | 신경망 head |

- RPN: Feature Map의 위치마다 Anchor 여러 개(대표적으로 3가지 크기 × 3가지 비율 = 9개)를 두고, Anchor마다 객체 여부(objectness)와 박스 보정값을 예측한다.
- 점수가 높은 후보를 NMS로 추려 2단계 head로 넘긴다.

### 꼬리질문
#### RoI Pooling은 왜 필요한가요?
후보 영역마다 크기가 다른데 뒤의 FC 층은 고정 크기 입력을 받기 때문입니다. RoI Pooling은 영역을 정해진 칸 수(예: 7×7)로 나눠 칸마다 max pooling해서 크기를 맞춥니다.

#### RoIAlign은 RoI Pooling과 무엇이 다른가요?
RoI Pooling은 좌표를 정수로 반올림해서 위치가 조금씩 어긋나는데, RoIAlign은 반올림 없이 bilinear 보간으로 값을 뽑습니다. 픽셀 단위 정밀도가 필요한 Mask R-CNN에서 도입됐습니다.

## Q. YOLOv1은 어떤 방식으로 탐지했고, 어떤 한계가 있었나요?
id: det-yolov1-grid
type: 개념
level: 기본
tags: YOLOv1, Grid, 작은 객체
source: 3. YOLO와 탐지 모델의 흐름 › YOLOv1: 7×7 Grid

### 30초 답변
YOLOv1은 입력 이미지를 7×7 격자로 나누고, 객체의 중심점이 들어간 칸이 그 객체의 예측을 맡게 했습니다. 칸마다 박스 2개와 Class 확률을 예측하니 이미지 한 장에서 최대 98개 박스만 낼 수 있습니다. 칸 하나가 Class를 하나만 고르기 때문에 작은 객체 여러 개의 중심이 같은 칸에 들어가면 하나밖에 못 찾아서, 작거나 붙어 있는 객체에 약했습니다.

### 자세히
```
입력 448×448 → 출력 7 × 7 × (B·5 + C) = 7 × 7 × 30   (B=2, Pascal VOC C=20)
박스 하나 = (x, y, w, h, confidence)
최대 박스 수 = 49칸 × 2 = 98
```

- 한계 정리
  - 칸당 Class 1개 → 한 칸에 여러 객체가 몰리면 놓친다.
  - 마지막 격자가 거칠어서 작은 객체의 위치가 부정확하다.
  - 박스 모양을 기준 없이 직접 회귀해서 특이한 비율의 객체에 약하다.

### 꼬리질문
#### 이후 YOLO는 이 한계를 어떻게 풀었나요?
YOLOv2는 Anchor Box로 다양한 모양을 기준 삼아 예측하게 했고, 이후 버전들은 해상도가 다른 여러 Feature Map에서 예측해서 작은 객체를 고해상도 격자가 맡게 했습니다. YOLO11은 80×80, 40×40, 20×20 세 격자를 씁니다.

#### "중심점이 들어간 칸이 담당한다"는 규칙은 학습에서 어떤 역할을 하나요?
GT 하나를 어느 예측이 맡을지 정하는 할당(assignment) 규칙입니다. 담당 칸의 예측만 그 GT에 대해 박스와 Class Loss를 받고, 나머지 칸은 객체가 없다고 학습됩니다.

## Q. Anchor Box란 무엇이고, 왜 도입했나요?
id: det-anchor-box
type: 개념
level: 기본
tags: Anchor Box, YOLOv2, 하이퍼파라미터
source: 3. YOLO와 탐지 모델의 흐름 › YOLOv2: Anchor Box

### 30초 답변
Anchor Box는 각 격자 위치에 미리 정해 둔 여러 크기와 비율의 기준 박스입니다. 모델은 박스를 처음부터 맞히는 대신, 객체와 모양이 가장 비슷한 Anchor에서 출발해 위치와 크기를 조금 보정하는 값을 예측합니다. 2017년 YOLOv2가 도입했고, 크기와 비율이 제각각인 객체를 찾는 학습이 쉬워졌습니다.

### 자세히
- 실습에서 정답 박스와 넓이는 같고 비율만 다른 Anchor를 겹쳐 봤다.

| 객체 | 비율 | 가장 잘 맞은 Anchor | IoU |
|---|---|---|---|
| 버스 | 가로가 1.6배 | 2:1 가로형 | 0.80 |
| 사람 | 세로가 3.7배 | 1:3 세로형 | 0.81 |

- 넓이가 같아도 비율이 안 맞으면 IoU가 0.2~0.3대로 떨어진다.
- 단점: 데이터셋에 맞는 Anchor 크기·비율을 골라야 하고, 개수와 형태가 하이퍼파라미터가 되며, 위치마다 여러 개라 후보 박스가 많아지고, 어떤 Anchor가 어떤 GT를 맡을지 정하는 매칭이 복잡하다.

### 꼬리질문
#### Anchor 크기는 어떻게 정하나요?
YOLOv2는 학습 데이터의 GT 박스 크기를 k-means로 군집화해서 대표 크기를 골랐고, 거리 기준으로 유클리드 거리 대신 1 - IoU를 썼습니다. 그래야 큰 박스가 거리 계산을 지배하지 않습니다.

#### 데이터셋과 Anchor가 안 맞으면 어떤 일이 생기나요?
아주 길쭉하거나 작은 객체처럼 어떤 Anchor와도 IoU가 낮은 GT는 담당 Anchor가 제대로 정해지지 않아 학습이 잘 안 되고 Recall이 떨어집니다.

## Q. Anchor-based와 Anchor-free 방식의 차이는 무엇인가요?
id: det-anchor-free
type: 비교
level: 심화
tags: Anchor-free, Anchor Box, YOLO11, Head
source: 4. YOLO11 › Backbone, Neck, Head

### 30초 답변
Anchor-based는 위치마다 미리 정한 기준 박스를 두고 그 Anchor에서의 보정값을 예측합니다. Anchor-free는 기준 박스 없이 격자점 자체에서 박스를 직접 예측합니다. YOLO11은 각 격자점에서 왼쪽, 위, 오른쪽, 아래 경계까지의 거리 (l, t, r, b) 4개를 예측하는 Anchor-free 방식이라, Anchor 크기나 비율을 데이터셋마다 고를 필요가 없습니다.

### 자세히
| | Anchor-based | Anchor-free |
|---|---|---|
| 예측 대상 | Anchor 대비 오프셋 (dx, dy, dw, dh) | 격자점에서 네 경계까지 거리, 또는 중심점과 크기 |
| 위치당 예측 수 | Anchor 개수만큼 | 1개 |
| 하이퍼파라미터 | Anchor 크기·비율·개수 | 없음 |
| 예 | Faster R-CNN, YOLOv2~v5, RetinaNet | FCOS, CenterNet, YOLOv8, YOLO11 |

```
x1 = px - l,  y1 = py - t,  x2 = px + r,  y2 = py + b   (px, py: 격자점 좌표)
```

### 꼬리질문
#### Anchor가 없으면 어떤 위치가 어떤 객체를 맡는지는 어떻게 정하나요?
정해 둔 Anchor와의 IoU 대신, 예측 자체의 품질로 담당을 고릅니다. Ultralytics YOLOv8 계열은 Class 점수와 예측 박스 IoU를 함께 본 점수가 높은 위치를 그 GT의 담당으로 고르는 Task-Aligned Assigner를 씁니다.

#### Anchor-free가 항상 더 좋은가요?
설정이 단순하고 데이터셋에 덜 민감하다는 장점이 크지만, 정확도는 할당 방식과 구조 전체에 좌우됩니다. Anchor를 없앤 것만으로 성능이 오르는 것은 아니고, 대신 튜닝해야 할 요소가 줄어든다고 보는 게 맞습니다.

## Q. 탐지 모델의 Backbone, Neck, Head는 각각 어떤 역할을 하나요?
id: det-backbone-neck-head
type: 개념
level: 기본
tags: Backbone, Neck, Head, YOLO11
source: 4. YOLO11 › Backbone, Neck, Head

### 30초 답변
Backbone은 Convolution 층을 쌓아 입력 이미지에서 Feature Map을 뽑습니다. Neck은 Backbone의 여러 단계에서 나온, 해상도가 다른 Feature Map을 합칩니다. Head는 합친 Feature Map 각각에서 박스 좌표와 Class 점수를 예측합니다.

### 자세히
| 부분 | 하는 일 | YOLO11에서 |
|---|---|---|
| Backbone | Feature 추출. 깊어질수록 해상도↓, 채널↑ | C3k2 블록, SPPF, C2PSA(attention) |
| Neck | 다른 해상도의 Feature 결합 | FPN(top-down) + PAN(bottom-up) |
| Head | 3개 Feature Map에서 예측 | 박스 branch와 Class branch 분리, Anchor-free |

- 얕은 층은 경계·질감 같은 저수준 정보와 정확한 위치를, 깊은 층은 객체 단위의 의미 정보를 담는다.
- Fine-tuning할 때 Backbone 가중치는 다른 데이터에도 잘 통해서 그대로 쓰거나 고정하기도 한다.

### 꼬리질문
#### Head에서 박스 branch와 Class branch를 나누는 이유는 무엇인가요?
위치 회귀와 분류는 필요한 Feature가 다르기 때문입니다. 분류는 객체의 의미를, 박스는 경계의 정확한 위치를 봐야 해서, branch를 나눈(decoupled) Head가 하나로 합친 Head보다 학습이 잘 된다고 알려져 있습니다.

#### Backbone으로 분류용 CNN을 그대로 쓸 수 있나요?
쓸 수 있습니다. 초기 탐지 모델은 VGG, ResNet 같은 ImageNet 분류 모델의 FC 층을 떼고 Backbone으로 썼습니다. 분류로 배운 Feature가 탐지에도 통하기 때문입니다.

## Q. Neck의 FPN과 PAN은 무엇이고, 왜 필요한가요?
id: det-fpn-pan
type: 개념
level: 심화
tags: FPN, PAN, Neck, Multi-scale
source: 4. YOLO11 › Backbone, Neck, Head

### 30초 답변
Backbone의 깊은 Feature Map은 의미 정보는 풍부하지만 해상도가 낮고, 얕은 Feature Map은 위치는 정확하지만 의미 정보가 약합니다. FPN은 깊은 층의 Feature를 키워서 얕은 층과 합치는 top-down 경로로, 고해상도 Feature Map에도 의미 정보를 넣어 줍니다. PAN은 그 결과를 다시 줄여 깊은 층으로 보내는 bottom-up 경로를 더해서, 저해상도 Feature Map에도 정확한 위치 정보를 전달합니다.

### 자세히
```
Backbone:   P3(80×80) ── P4(40×40) ── P5(20×20)
FPN  ↓ :    P5 upsample → P4와 concat → upsample → P3와 concat
PAN  ↑ :    P3' downsample → P4'와 concat → downsample → P5'와 concat
Head :      P3'', P4'', P5'' 각각에서 예측
```

- 결과적으로 Neck을 지난 세 Feature Map이 모두 위치 정보와 의미 정보를 함께 갖는다.
- FPN은 2017년, PAN(PANet)은 2018년에 제안됐다.

### 꼬리질문
#### FPN 없이 고해상도 Feature Map에서 바로 예측하면 무슨 문제가 있나요?
얕은 층의 Feature는 경계나 질감 수준이라, 작은 객체의 위치는 잡아도 그게 무엇인지 구분할 의미 정보가 부족합니다. 그래서 작은 객체의 분류가 틀리거나 배경과 헷갈립니다.

#### Feature를 합칠 때 더하기와 concat의 차이는?
FPN 원 논문은 채널을 맞춘 뒤 원소별로 더했고, YOLO 계열 Neck은 채널 방향으로 이어 붙인(concat) 뒤 Convolution으로 섞습니다. concat은 채널이 늘어 계산이 늘지만 두 정보를 따로 보존한 채 섞을 수 있습니다.

## Q. 왜 해상도가 다른 여러 Feature Map에서 예측하나요?
id: det-multiscale-feature
type: 이유
level: 심화
tags: Multi-scale, Feature Map, stride, 작은 객체
source: 4. YOLO11 › 3개 Feature Map, 8,400개 위치

### 30초 답변
한 이미지 안에도 객체 크기가 제각각이라, 한 해상도의 Feature Map으로는 큰 객체와 작은 객체를 함께 잘 잡기 어렵기 때문입니다. 해상도가 높은 Feature Map은 칸이 촘촘해서 작은 객체를, 해상도가 낮은 Feature Map은 칸 하나가 넓은 영역을 봐서 큰 객체를 맡습니다. 실습에서 최종 박스가 어느 Feature Map에서 나왔는지 확인해 보니 실제로 이렇게 나뉘었습니다.

### 자세히
| stride | 격자 (640 입력) | 주로 맡는 객체 |
|---|---|---|
| 8 | 80×80 | 작은 객체 |
| 16 | 40×40 | 중간 객체 |
| 32 | 20×20 | 큰 객체 |

- bus.jpg 결과: 버스와 크게 찍힌 사람 3명은 전부 stride 32, 왼쪽 끝에 반쯤 잘려 작게 보이는 사람만 stride 16에서 나왔다. 작은 객체가 없어서 stride 8 후보는 없었다.
- 출력 순서가 stride 8 위치 6,400개 → stride 16 1,600개 → stride 32 400개라서, 인덱스로 출처를 알 수 있다.

### 꼬리질문
#### stride 32 격자에서 아주 작은 객체를 찾기 어려운 이유는?
칸 하나가 원본의 32×32 픽셀을 덮어서, 그보다 작은 객체는 한 칸 안에서 Feature가 배경과 뒤섞입니다. 여러 번 downsampling되는 동안 작은 객체의 정보가 거의 사라지는 것도 이유입니다.

#### 아주 작은 객체가 많은 데이터라면 어떻게 하나요?
입력 해상도(`imgsz`)를 키우거나, stride 4 수준의 더 촘촘한 Feature Map을 쓰는 Head를 추가하는 방법이 있습니다. 큰 이미지를 겹치게 잘라 조각마다 추론하는 방법(SAHI 같은 슬라이싱 추론)도 씁니다.

## Q. YOLO11은 640×640 입력에서 어떻게 8,400개 후보를 만들고 최종 결과를 내나요?
id: det-yolo11-8400
type: 개념
level: 심화
tags: YOLO11, 추론 과정, 후처리, 8400
source: 4. YOLO11 › 3개 Feature Map, 8,400개 위치

### 30초 답변
YOLO11은 stride 8, 16, 32의 Feature Map 3개를 쓰는데, 640×640 입력이면 80×80, 40×40, 20×20 격자가 되어 합치면 8,400개 위치에서 예측합니다. 위치마다 박스 4개 값과 Class별 점수를 내므로 COCO 모델 출력은 (1, 84, 8400)입니다. 여기서 점수가 낮은 후보를 지우고, NMS로 중복을 지운 뒤, 좌표를 원본 크기로 되돌려 최종 결과를 냅니다.

### 자세히
```
80² + 40² + 20² = 6,400 + 1,600 + 400 = 8,400
출력 (1, 84, 8400):  84 = 박스 xywh 4 + COCO Class 80
```

1. 비율을 유지하며 640×640으로 resize하고 남는 영역은 padding
2. forward 한 번 → 후보 8,400개와 Class별 점수
3. Confidence Threshold로 낮은 후보 제거
4. NMS로 중복 박스 제거
5. 좌표를 원본 이미지 크기로 복원

- 직접 풀어 본 결과(bus.jpg): conf 0.25 이상 후보 47개 → NMS(IoU 0.7) 후 5개(버스 1, 사람 4). 라이브러리 결과와 같았다.

### 꼬리질문
#### 입력 크기를 1280으로 바꾸면 후보 수는 어떻게 되나요?
격자가 160×160, 80×80, 40×40이 되어 25,600 + 6,400 + 1,600 = 33,600개로 4배가 됩니다. 작은 객체에는 유리하지만 계산량과 NMS 부담도 같이 늘어납니다.

#### 출력에 objectness 점수가 따로 없는데 confidence는 무엇인가요?
YOLO11은 별도 objectness 없이 Class별 점수만 내고, 후보의 confidence로는 가장 높은 Class 점수를 씁니다. YOLOv5처럼 objectness를 따로 두고 Class 점수와 곱하던 버전과 다른 점입니다.

## Q. YOLO 모델 크기(n, s, m, l, x)는 어떻게 고르나요?
id: det-model-size
type: 비교
level: 기본
tags: 모델 크기, 속도, 정확도, YOLO11
source: 4. YOLO11 › 모델 크기와 사전학습 모델

### 30초 답변
다섯 가지는 같은 구조에서 층의 반복 횟수(depth)와 채널 수(width)만 다르게 한 것이고, 클수록 정확하지만 느립니다. 실시간 처리나 CPU, 임베디드 장비라면 n이나 s를, 정확도가 우선이고 GPU가 넉넉하면 m, l, x를 고릅니다. 학습 데이터가 적으면 큰 모델이 과적합되기 쉽다는 점도 함께 봐야 합니다.

### 자세히
| | 파라미터 (실습) | bus.jpg 결과 | 왼쪽 끝 사람 conf |
|---|---|---|---|
| yolo11n | 262만 | 버스 1, 사람 4 | 0.62 |
| yolo11s | 946만 (3.6배) | 버스 1, 사람 4 | 0.66 |

- 찾은 객체 수는 같았고, 차이는 신뢰도와 박스 위치에서 났다.
- 선택 순서: 목표 장비에서 필요한 FPS를 먼저 정하고, 그 안에서 가장 큰 모델을 고른 뒤 데이터 양을 보고 조정한다.

### 꼬리질문
#### 큰 모델의 이득은 주로 어디서 나타났나요?
실습의 IoU 임계값별 AP에서 s가 모든 구간에서 n보다 높았고, 차이는 IoU 0.85~0.90 구간에서 가장 컸습니다. 객체를 찾는 것보다 박스를 정확한 자리에 치는 데서 이득이 더 컸습니다.

#### 속도를 비교할 때는 무엇을 봐야 하나요?
같은 장비, 같은 입력 크기, 같은 batch에서 잰 지연 시간(ms)이나 FPS를 봐야 합니다. 파라미터 수나 FLOPs는 참고일 뿐이고, 실제 속도는 장비와 런타임(TensorRT, ONNX 등)에 따라 달라집니다.

## Q. Confidence Threshold란 무엇이고, 값을 바꾸면 결과가 어떻게 달라지나요?
id: det-conf-threshold
type: 개념
level: 기본
tags: Confidence Threshold, 후처리, Precision, Recall
source: 5. 탐지 결과 꺼내 쓰기 › Confidence Threshold

### 30초 답변
Confidence Threshold는 결과에 남길 박스의 최소 점수이고, 이보다 낮은 박스는 버립니다. 낮추면 놓치는 객체는 줄지만 잘못된 검출이 늘고, 높이면 잘못된 검출은 줄지만 놓치는 객체가 늘어납니다. Ultralytics 기본값은 0.25입니다.

### 자세히
| conf | 검출 수 | 바뀐 것 (bus.jpg) |
|---|---|---|
| 0.10 | 6 | 신발을 `skateboard 0.12`로 잘못 검출 |
| 0.25 | 5 | 버스 1, 사람 4 |
| 0.50 | 5 | 그대로 |
| 0.80 | 4 | 몸이 잘린 사람(0.62)이 빠짐 |

```python
model("image.jpg", conf=0.5)
```

### 꼬리질문
#### 실제 서비스에서 Threshold는 어떻게 정하나요?
Validation 데이터에서 Threshold별 Precision과 Recall을 그려 보고, 서비스에서 무엇이 더 치명적인지로 고릅니다. 놓치면 위험한 결함 검사는 낮게, 오경보가 비싼 알림 시스템은 높게 잡습니다.

#### mAP를 잴 때는 왜 Threshold를 아주 낮게 두나요?
mAP는 모든 Threshold에서의 Precision과 Recall을 PR Curve로 요약하는 지표라, 낮은 점수의 예측까지 다 있어야 곡선 끝까지 그릴 수 있기 때문입니다. 그래서 평가할 때는 conf 0.001 같은 값을 씁니다.

## Q. NMS(Non-Maximum Suppression)는 무엇이고 어떻게 동작하나요?
id: det-nms
type: 개념
level: 기본
tags: NMS, 후처리, IoU
source: 6. 중복 박스와 NMS

### 30초 답변
같은 객체 주변의 여러 위치가 비슷한 박스를 예측하기 때문에, Confidence Threshold만 적용하면 한 객체에 박스가 여러 개 남습니다. NMS는 점수가 가장 높은 박스를 고르고 그 박스와 IoU가 기준보다 큰 박스를 지우는 일을 반복해서, 객체마다 박스 하나만 남기는 후처리입니다. Ultralytics는 기본적으로 같은 Class의 박스끼리만 NMS를 합니다.

### 자세히
1. 남은 박스를 점수 내림차순으로 정렬
2. 가장 높은 박스를 결과로 선택
3. 선택한 박스와 IoU가 IoU Threshold보다 큰 박스 제거
4. 남은 박스가 없을 때까지 2~3 반복

```python
def nms(boxes, scores, iou_thr):
    order = scores.argsort()[::-1]
    keep = []
    while len(order) > 0:
        i = order[0]; keep.append(i)
        order = [j for j in order[1:] if iou(boxes[i], boxes[j]) <= iou_thr]
    return keep
```

- 직접 짠 함수로 가운데 남자 주변 후보 9개(점수 0.72~0.83)를 넣었더니, 0.83 박스와 나머지 8개의 IoU가 0.95~0.99라 한 번에 1개로 줄었다.

### 꼬리질문
#### Class를 구분하지 않고 NMS를 하면 어떻게 되나요?
겹쳐 있는 다른 Class 객체, 예를 들어 말 위의 사람 박스가 말 박스와 많이 겹치면 지워질 수 있습니다. 반대로 같은 객체에 다른 Class 박스가 둘 붙는 경우는 정리할 수 있어서, Ultralytics는 `agnostic_nms=True` 옵션으로 고를 수 있게 합니다.

#### NMS의 계산량은 어떤가요?
박스 수가 N이면 최악의 경우 쌍마다 IoU를 계산해 O(N²)입니다. 그래서 먼저 Confidence Threshold로 후보를 크게 줄인 뒤 NMS를 하고, 남길 최대 박스 수도 제한합니다.

## Q. NMS의 IoU Threshold를 너무 낮거나 높게 잡으면 왜 문제가 되나요?
id: det-nms-iou-threshold
type: 이유
level: 심화
tags: NMS, IoU Threshold, 겹친 객체
source: 6. 중복 박스와 NMS › IoU Threshold

### 30초 답변
IoU Threshold는 두 박스를 같은 객체로 볼 겹침 기준입니다. 너무 낮으면 조금만 겹쳐도 지워져서 서로 가까이 있는 다른 객체의 박스까지 지워지고, 너무 높으면 같은 객체의 중복 박스가 남습니다. 실습에서 두 사람이 겹쳐 선 사진에 0.1을 주었더니, 다른 사람 박스 때문에 가장 좋은 박스가 지워지는 것을 확인했습니다.

### 자세히
- Ultralytics 추론 기본값은 `iou=0.7`

| 설정 (zidane.jpg, 두 사람 박스 IoU 0.333) | 결과 |
|---|---|
| iou=0.7 | 지단 0.78, 안첼로티 0.84 두 박스가 그대로 남음 |
| iou=0.1 | 지단의 0.78 박스가 안첼로티 박스와 0.333 겹친다는 이유로 제거됨 |

- 0.1에서도 사람 수는 2명이었지만, 우연히 살아남은 0.34짜리 다른 후보 덕분이었다. 개수만 보고 괜찮다고 판단하면 안 된다.

### 꼬리질문
#### 군중처럼 객체가 빽빽한 장면에서는 어떻게 설정하나요?
서로 다른 객체끼리 IoU가 높으니 Threshold를 높여서 겹친 객체를 살립니다. 그래도 한계가 있어서 Soft-NMS처럼 박스를 지우지 않고 점수만 깎는 방법을 쓰기도 합니다.

#### 평가의 IoU Threshold와 NMS의 IoU Threshold는 같은 것인가요?
다릅니다. NMS의 값은 예측끼리 중복인지 판단하는 기준이고, 평가의 값은 예측과 GT가 맞았는지 판정하는 기준입니다. 같은 IoU 계산을 쓰지만 비교하는 대상과 목적이 다릅니다.

## Q. NMS의 한계와 이를 보완하거나 없앤 방법(Soft-NMS, DETR)을 설명해 보세요.
id: det-nms-alternatives
type: 개념
level: 심화
tags: Soft-NMS, DETR, NMS-free, Transformer
source: 보충 (블로그 밖 내용)

### 30초 답변
기본 NMS는 IoU가 기준을 넘으면 무조건 지우기 때문에 실제로 겹쳐 있는 다른 객체를 놓칠 수 있고, Threshold 하나에 결과가 민감합니다. Soft-NMS는 박스를 지우는 대신 겹친 정도에 따라 점수를 깎아서, 많이 겹친 다른 객체도 점수가 충분하면 살아남게 합니다. DETR은 Transformer로 객체 수만큼의 박스 집합을 직접 예측하고 학습 때 GT와 일대일로 매칭해서, 중복 박스가 생기지 않아 NMS 자체가 필요 없습니다.

### 자세히
| | NMS | Soft-NMS (2017) | DETR (2020) |
|---|---|---|---|
| 중복 처리 | IoU > 기준이면 삭제 | IoU에 따라 점수 감소 (선형 또는 가우시안) | 일대일 매칭으로 중복 자체를 학습에서 억제 |
| 후처리 | 필요 | 필요 | 불필요 |
| 특징 | 단순, 빠름 | 겹친 객체 Recall↑ | Anchor·NMS 없음, 수렴이 느리고 초기 버전은 작은 객체에 약함 |

- DETR: CNN Backbone + Transformer encoder-decoder. 정해진 개수의 object query가 각각 박스 하나를 내고, 학습 때 헝가리안 알고리즘으로 예측과 GT를 일대일로 짝지어 Loss를 계산한다.

### 꼬리질문
#### DETR은 왜 NMS가 필요 없나요?
GT 하나에 예측 하나만 짝지어 학습하고, 짝이 없는 예측은 "객체 없음"으로 학습되기 때문입니다. 같은 객체를 두 query가 예측하면 하나는 손해를 보니, 모델이 스스로 중복을 피하도록 배웁니다.

#### NMS가 없으면 배포에서 어떤 점이 좋나요?
후처리 단계가 없어 지연 시간이 일정하고, 하드웨어 가속기로 옮길 때 NMS 구현 차이로 결과가 달라지는 문제가 줄어듭니다.

## Q. IoU란 무엇이고 어떻게 계산하나요?
id: det-iou
type: 개념
level: 기본
tags: IoU, 평가 지표, NMS
source: 7. 탐지 모델 평가 › IoU

### 30초 답변
IoU(Intersection over Union)는 두 박스의 겹친 면적을 두 박스를 합친 면적으로 나눈 값으로, 0이면 전혀 안 겹치고 1이면 완전히 같습니다. 겹친 영역은 두 왼쪽 위 꼭짓점 중 큰 값과 두 오른쪽 아래 꼭짓점 중 작은 값으로 구하고, 합집합은 두 넓이의 합에서 교집합을 한 번 뺍니다. 평가에서 예측과 GT의 일치를 판정할 때, NMS에서 중복을 판단할 때 모두 씁니다.

### 자세히
```python
inter_w = max(0, min(b1[2], b2[2]) - max(b1[0], b2[0]))
inter_h = max(0, min(b1[3], b2[3]) - max(b1[1], b2[1]))
intersection = inter_w * inter_h
union = area1 + area2 - intersection
iou = intersection / union
```

- 가로나 세로가 음수면 안 겹친 것이므로 0으로 둔다.
- 작은 박스가 큰 박스 안에 완전히 들어가도 크기 차이가 크면 IoU는 0.111처럼 낮다. 다 포함했다고 잘 맞힌 것이 아니다.

### 꼬리질문
#### 두 박스가 전혀 안 겹치면 IoU를 Loss로 쓸 때 무슨 문제가 있나요?
얼마나 떨어져 있든 IoU가 0이라 기울기가 0이 되고, 어느 쪽으로 움직여야 할지 알려 주지 못합니다. 그래서 두 박스를 감싸는 최소 박스를 이용하는 GIoU, 중심 거리까지 보는 DIoU와 CIoU가 나왔습니다.

#### 두 박스가 xywh 형식이면 바로 계산할 수 있나요?
먼저 xyxy로 바꿔야 합니다. 교집합 계산에 꼭짓점 좌표가 필요하기 때문입니다.

## Q. 작은 객체 탐지는 왜 어렵나요?
id: det-small-object
type: 이유
level: 심화
tags: 작은 객체, IoU, Multi-scale, Augmentation
source: 7. 탐지 모델 평가 › IoU, 보충 (블로그 밖 내용)

### 30초 답변
작은 객체는 픽셀 수가 적어서 Feature 자체가 빈약하고, downsampling을 거치면서 정보가 거의 사라집니다. 평가 쪽에서도 불리한데, IoU는 같은 픽셀 오차가 작은 박스에 훨씬 가혹해서 몇 픽셀만 밀려도 FP가 됩니다. 실습에서 재 보니 20×20 박스는 4px만 밀려도 IoU가 0.5 아래로 떨어졌습니다.

### 자세히
| 박스 크기 | IoU 0.5 아래로 떨어지는 이동 거리 (실습) |
|---|---|
| 200×200 | 37px |
| 100×100 | 19px |
| 50×50 | 10px |
| 20×20 | 4px |

- 원인: 적은 픽셀, 깊은 층에서 정보 소실, 라벨링 오차의 영향이 상대적으로 큼, 학습 데이터에서 작은 객체 비중이 적은 경우가 많음.
- 대응: 입력 해상도 키우기, 고해상도 Feature Map(FPN) 활용, 이미지 타일링·슬라이싱 추론, 작은 객체를 늘리는 Augmentation(copy-paste 등).
- COCO는 넓이 32² 미만을 small로 분류하고 APs를 따로 보고한다.

### 꼬리질문
#### 해상도를 키우는 것 말고 데이터 쪽에서 할 수 있는 일은?
작은 객체가 있는 이미지를 더 모으거나, 객체를 잘라 다른 이미지에 붙이는 copy-paste 같은 Augmentation으로 작은 객체의 학습 빈도를 늘릴 수 있습니다. 라벨링 기준에서 너무 작은 객체를 어떻게 처리할지 미리 정해 두는 것도 중요합니다.

## Q. 탐지에서 TP, FP, FN은 어떻게 정의하고, TN은 왜 쓰지 않나요?
id: det-tp-fp-fn
type: 개념
level: 기본
tags: TP, FP, FN, 평가 지표, Ground Truth
source: 7. 탐지 모델 평가 › TP, FP, FN

### 30초 답변
예측 박스가 GT와 Class가 같고 IoU가 기준값 이상이면 TP이고, GT 하나에는 TP가 최대 하나만 매칭됩니다. GT와 매칭되지 않은 예측은 FP, 어떤 예측과도 매칭되지 않은 GT는 FN으로 모델이 놓친 객체입니다. TN은 "객체가 없는 곳에 아무것도 예측하지 않은 경우"인데, 이미지에서 배경 위치는 셀 수 없이 많아 정의할 수 없어서 쓰지 않습니다.

### 자세히
| 판정 | 조건 |
|---|---|
| TP | 같은 Class, IoU ≥ 기준, 아직 매칭 안 된 GT |
| FP | 객체 없는 곳, Class가 다름, IoU 부족, 이미 매칭된 GT에 중복 예측 |
| FN | 매칭된 예측이 없는 GT (Threshold에 걸려 지워진 예측의 GT 포함) |

- 같은 예측도 IoU 기준에 따라 판정이 바뀐다. IoU 0.6짜리 예측은 기준 0.5에서 TP, 0.75에서 FP다. 예측이 나빠진 게 아니라 채점 기준이 까다로워진 것이다.
- 그래서 TN이 필요한 Accuracy, Specificity, ROC Curve 대신 Precision, Recall, PR Curve를 쓴다.

### 꼬리질문
#### IoU가 기준에 못 미친 예측은 FP와 FN을 동시에 만드나요?
그렇습니다. 그 예측은 FP가 되고, 그 GT를 맞힌 다른 예측이 없다면 GT는 FN이 됩니다. 실습 예제의 P5와 GT2가 이 경우였습니다.

#### GT 품질이 평가에 왜 중요한가요?
모든 판정이 GT와의 비교로 이뤄지기 때문입니다. 라벨이 빠진 객체를 모델이 맞게 찾으면 FP로 세어지고, 박스가 부정확하면 IoU가 낮아져 TP가 FP로 바뀝니다.

## Q. 평가할 때 예측 박스와 GT는 어떤 절차로 매칭하나요?
id: det-matching
type: 개념
level: 심화
tags: 매칭, TP, FP, 평가 절차
source: 7. 탐지 모델 평가 › TP, FP, FN

### 30초 답변
예측을 Confidence Score 내림차순으로 정렬하고, 각 예측마다 같은 Class의 GT 중 IoU가 가장 큰 GT를 찾습니다. 그 IoU가 기준 이상이고 그 GT가 아직 매칭되지 않았으면 TP, 아니면 FP로 판정합니다. 모든 예측을 처리한 뒤 남은 GT는 FN입니다. 점수 높은 예측이 먼저 GT를 차지하기 때문에, 같은 객체에 대한 중복 예측은 FP가 됩니다.

### 자세히
- 실습 예제: GT 3개, 예측 6개, IoU 기준 0.5. FP 네 가지 경우를 모두 넣었다.

| 예측 | 판정 | 이유 |
|---|---|---|
| P2 | FP | P1과 같은 dog를 잡았지만 점수가 낮아 늦게 처리됨. GT1은 이미 P1과 매칭 |
| P3 | FP | 위치는 GT1과 맞지만 Class를 cat으로 예측 |
| P5 | FP | GT2 근처지만 IoU가 기준 미달 → GT2는 FN |

- 결과: TP 2, FP 4, FN 1

### 꼬리질문
#### 점수 순서대로 매칭하는 이유는 무엇인가요?
PR Curve는 점수 순서대로 예측을 하나씩 포함하며 그리기 때문에, 높은 점수의 예측이 먼저 GT를 차지해야 Threshold를 바꿨을 때의 결과와 일치합니다. 점수가 높은 예측을 낮은 예측의 중복으로 치면 곡선이 왜곡됩니다.

#### 매칭 방식이 다르면 결과도 달라지나요?
조금 달라질 수 있습니다. 실습에서 점수순 매칭으로 직접 계산한 mAP50-95는 0.801, IoU가 큰 짝부터 정하는 Ultralytics는 0.793이었습니다. 기준 IoU가 높을 때 짝이 몇 개 달라진 것으로 보입니다.

## Q. 탐지에서 Precision과 Recall은 무엇이고, 둘은 어떤 관계인가요?
id: det-precision-recall
type: 개념
level: 기본
tags: Precision, Recall, Trade-off, Confidence Threshold
source: 7. 탐지 모델 평가 › Precision과 Recall

### 30초 답변
Precision은 모델이 예측한 박스 중 맞은 비율로, 낮으면 잘못된 검출이 많다는 뜻입니다. Recall은 실제 객체 중 찾아낸 비율로, 낮으면 놓친 객체가 많다는 뜻입니다. 두 값은 Confidence Threshold에 따라 반대로 움직여서, Threshold를 높이면 Precision은 오르고 Recall은 내려갑니다.

### 자세히
```
Precision = TP / (TP + FP)      예측한 것 중 맞은 비율
Recall    = TP / (TP + FN)      실제 객체 중 찾은 비율
```

- 매칭 예제(TP 2, FP 4, FN 1): Precision 2/6 = 0.33, Recall 2/3 = 0.67
- 실습 노트북 문제(TP 6, FP 2, FN 4): Precision 0.75, Recall 0.6
- Ultralytics 로그의 P, R은 특정 Confidence Threshold 하나에서 잰 값이라, 모델 비교에는 mAP를 쓴다.

### 꼬리질문
#### Recall이 낮으면 무엇부터 확인하나요?
작은 객체, 가려진 객체, 데이터가 적은 Class에서 주로 생기므로 그 객체들의 학습 데이터 양과 라벨 누락부터 봅니다. 너무 높은 Confidence Threshold 때문에 지워진 것은 아닌지도 확인합니다.

#### Precision과 Recall을 하나로 합친 지표는 없나요?
둘의 조화평균인 F1 Score가 있습니다. Ultralytics는 Threshold별 F1 곡선을 그려 F1이 최대가 되는 Confidence를 알려 주는데, 이것도 Threshold 하나의 값이라 모델 전체 비교에는 mAP를 씁니다.

## Q. PR Curve와 AP는 무엇이고, 어떻게 계산하나요?
id: det-pr-curve-ap
type: 개념
level: 기본
tags: PR Curve, AP, 보간
source: 7. 탐지 모델 평가 › PR Curve, AP, mAP

### 30초 답변
PR Curve는 Confidence Threshold를 바꿔 가며 얻은 (Recall, Precision) 점을 이은 곡선으로, Class마다 하나씩 그리고 오른쪽 위에 가까울수록 좋습니다. AP는 이 곡선 아래 면적으로, 모든 Threshold에서의 성능을 0~1 값 하나로 요약합니다. 실제 곡선은 지그재그라서 각 Recall의 Precision을 그 뒤 구간의 최대값으로 바꿔 계단 모양으로 만든 다음 넓이를 잽니다.

### 자세히
1. 한 Class의 모든 예측을 점수 내림차순 정렬
2. 각 예측을 TP 또는 FP로 판정
3. 위에서부터 하나씩 포함하며 누적 TP, 누적 FP를 셈
4. 매 시점의 Precision, Recall을 점으로 찍음
5. Precision을 오른쪽 최대값으로 보간하고 넓이 계산

```python
prec = np.maximum.accumulate(prec[::-1])[::-1]     # 계단형 보간
ap = np.mean(np.interp(np.linspace(0, 1, 101), rec, prec))   # COCO식 101점
```

### 꼬리질문
#### 왜 Precision을 오른쪽 최대값으로 보간하나요?
Threshold를 조금 낮춰서 Recall과 Precision이 함께 오르는 지점이 있다면, 그 Recall 수준에서는 더 좋은 Precision을 얻을 수 있다는 뜻이기 때문입니다. 보간하면 곡선이 단조 감소가 되어 작은 흔들림에 AP가 덜 민감해집니다.

#### Pascal VOC와 COCO의 AP 계산은 무엇이 다른가요?
VOC 2007은 Recall 11개 점에서 보간한 Precision을 평균했고 IoU 0.5 하나만 썼습니다. COCO는 101개 점에서 보간하고, IoU 0.50~0.95의 열 개 기준으로 계산한 값을 평균합니다.

## Q. mAP란 무엇이고, 왜 탐지 모델 비교에 mAP를 쓰나요?
id: det-map
type: 개념
level: 기본
tags: mAP, AP, 평가 지표
source: 7. 탐지 모델 평가 › PR Curve, AP, mAP

### 30초 답변
mAP(mean Average Precision)는 Class별 AP의 평균입니다. Precision과 Recall은 Confidence Threshold 하나에서 잰 값이라 Threshold를 어떻게 잡느냐에 따라 모델 순위가 바뀔 수 있는데, AP는 모든 Threshold를 아우르니 그 영향을 받지 않습니다. 그래서 탐지 모델 비교에 가장 많이 쓰는 지표입니다.

### 자세히
```
mAP = (1 / C) × Σ AP_c        C: Class 수
```

- Class마다 같은 비중으로 평균한다. 데이터가 적은 Class 하나의 AP가 낮으면 mAP도 같이 내려간다.
- 문헌에 따라 "AP"가 이미 Class 평균을 뜻하기도 한다(COCO는 AP와 mAP를 구분 없이 쓴다).
- 해석: Class별 AP를 펼쳐 놓고 낮은 Class를 찾은 뒤, 그 Class의 이미지 수, 라벨 품질, 객체 크기를 확인한다.

### 꼬리질문
#### mAP가 높으면 실제 서비스에서도 잘 동작하나요?
꼭 그렇지는 않습니다. mAP는 모든 Threshold의 평균적인 성능이라, 서비스는 결국 Threshold 하나로 운영되므로 그 지점의 Precision과 Recall을 따로 확인해야 합니다. 평가 데이터가 실제 환경과 다르면 숫자 자체도 믿기 어렵습니다.

#### Validation mAP와 Test mAP가 크게 다르면 무엇을 의심하나요?
Validation으로 모델과 하이퍼파라미터를 고르다 보니 Validation에 맞춰진 모델일 수 있습니다. 두 데이터의 분포가 다른지도 확인합니다.

## Q. mAP50과 mAP50-95는 어떻게 다르고, 각각 무엇을 말해 주나요?
id: det-map50-vs-5095
type: 비교
level: 기본
tags: mAP50, mAP50-95, COCO, 위치 정확도
source: 7. 탐지 모델 평가 › mAP50과 mAP50-95

### 30초 답변
mAP50은 IoU 0.5에서 계산한 mAP라서 객체를 찾았는지를 중심으로 평가합니다. mAP50-95는 IoU 기준을 0.50부터 0.05 간격으로 0.95까지 바꿔 계산한 mAP 10개의 평균이라, 박스 위치가 얼마나 정확한지까지 평가하고 COCO 벤치마크의 대표 지표입니다. mAP50은 높은데 mAP50-95가 낮으면 객체는 찾지만 박스를 정확한 자리에 치지 못한다는 뜻입니다.

### 자세히
| | mAP50 | mAP50-95 |
|---|---|---|
| IoU 기준 | 0.5 하나 | 0.50, 0.55, ..., 0.95 (10개) 평균 |
| 보는 것 | 찾았는가 | 찾았는가 + 위치 정확도 |
| 대표 사용 | Pascal VOC | COCO |

- 같은 모델에서는 항상 mAP50 ≥ mAP50-95다. 기준이 높아지면 TP가 FP로 바뀌기만 하고 그 반대는 없다.
- 실습의 scratch 학습: mAP50 0.954, mAP50-95 0.535. 알약 위치는 찾지만 박스가 부정확한 전형적인 경우였다.

### 꼬리질문
#### 어떤 지표를 기준으로 삼아야 하나요?
위치 정밀도가 중요하면(로봇 집기, 측정) mAP50-95를, 객체 존재 여부만 중요하면(개수 세기, 경보) mAP50을 중심으로 봅니다. Ultralytics는 `best.pt`를 고를 때 둘을 1:9로 섞은 fitness를 써서 mAP50-95에 무게를 둡니다.

#### mAP75는 언제 보나요?
IoU 0.75 기준 mAP로, mAP50보다 엄격하게 위치를 보는 중간 지표입니다. COCO는 AP50, AP75를 mAP50-95와 함께 보고해서 위치 정확도의 차이를 따로 보여 줍니다.

## Q. 모델 크기에 따라 IoU 기준별 AP가 어떻게 달라지는지 직접 비교해 본 적이 있나요?
id: det-exp-n-vs-s
type: 경험
level: 심화
tags: mAP50-95, 모델 크기, 실험
source: 7. 탐지 모델 평가 › mAP50과 mAP50-95

### 30초 답변
coco8-seg 검증 4장의 박스 라벨로 yolo11n과 yolo11s를 IoU 임계값별로 평가해 봤습니다. n은 mAP50 0.854, mAP50-95 0.631이었고 s는 0.935, 0.745로 모든 구간에서 s가 위였습니다. 차이는 IoU 0.85~0.90 구간에서 가장 크게 벌어져서, 큰 모델의 이득이 객체를 찾는 것보다 박스를 정확한 자리에 치는 데서 더 컸습니다.

### 자세히
| 모델 | mAP50 | mAP50-95 |
|---|---|---|
| yolo11n | 0.854 | 0.631 |
| yolo11s | 0.935 | 0.745 |

- 임계값이 올라갈수록 두 모델 모두 내려가고, 0.95에서는 0.2 안팎까지 떨어졌다.
- 검증 이미지가 4장뿐인 연습용 데이터라 숫자 자체보다는 곡선 모양만 의미 있게 봤다.

### 꼬리질문
#### 이 결과에서 mAP50만 봤다면 무엇을 놓쳤을까요?
mAP50 차이(0.08)만 보면 두 모델의 차이가 작아 보이지만, 높은 IoU 구간에서 벌어지는 위치 정확도 차이는 보이지 않습니다. 박스 정밀도가 중요한 작업이면 그 차이가 모델 선택을 바꿀 수 있습니다.

#### 평가 이미지가 4장이면 어떤 문제가 있나요?
객체 수가 적어 예측 하나의 판정이 바뀌면 AP가 크게 출렁입니다. 모델 선택 같은 결론을 내리려면 충분한 크기의 평가 데이터가 필요합니다.

## Q. 사전학습된 탐지 모델을 내 데이터로 Fine-tuning하면 내부에서 무슨 일이 일어나나요?
id: det-finetune-process
type: 개념
level: 기본
tags: 전이학습, Fine-tuning, Freeze, COCO
source: 8. 전이학습과 Fine-tuning › Fine-tuning

### 30초 답변
COCO로 학습한 `yolo11n.pt`를 불러오면, 새 데이터셋의 Class 수에 맞게 Head의 Class 예측 부분을 새로 만들고, 구조와 크기가 같은 나머지 Backbone, Neck, Head 가중치는 사전학습 값을 복사합니다. 그 상태에서 새 데이터로 학습해 가중치를 갱신합니다. Ultralytics는 기본적으로 전체 층을 학습하고, `freeze` 인자로 앞쪽 층을 고정할 수 있습니다.

### 자세히
1. 사전학습 모델 로드
2. Class 수에 맞춰 Head의 Class branch 재생성 (COCO 80 → 내 Class 수)
3. 크기가 맞는 가중치 복사 → `Transferred N/M items from pretrained weights` 로그
4. 새 데이터로 학습

```python
model = YOLO("yolo11n.pt")
model.train(data="data.yaml", epochs=100, imgsz=640, freeze=10)   # 앞 10개 층 고정
```

- Fine-tuning한 Custom 모델은 내가 정의한 Class만 찾고, 학습에 넣지 않은 COCO Class는 더 이상 찾지 않는다.

### 꼬리질문
#### 박스 branch는 왜 그대로 재사용할 수 있나요?
박스 예측은 Class 수와 관계없이 출력 크기가 같아서 가중치 모양이 그대로 맞기 때문입니다. "객체의 경계가 어디인가"는 새 Class에도 통하는 지식이라 재사용하면 학습이 빨라집니다.

#### 기존 COCO Class를 유지하면서 새 Class를 추가하려면 어떻게 하나요?
새 데이터만으로 학습하면 기존 Class를 잊어버리므로, COCO 데이터 일부를 함께 넣어 학습하거나 기존 모델과 새 모델을 따로 두고 결과를 합칩니다. 기존 Class 라벨이 새 데이터에 빠져 있으면 그 객체가 배경으로 학습되는 점도 주의해야 합니다.

## Q. 탐지 모델 학습에서 주요 하이퍼파라미터는 무엇이고, 과적합은 어떻게 알아보고 막나요?
id: det-overfitting-hparams
type: 개념
level: 기본
tags: 하이퍼파라미터, 과적합, Loss 곡선, patience
source: 8. 전이학습과 Fine-tuning › 하이퍼파라미터와 과적합

### 30초 답변
Epoch는 데이터 전체를 몇 번 쓰는지, Batch Size는 한 번 갱신에 쓰는 이미지 수, Learning Rate는 한 번 갱신할 때 바꾸는 크기입니다. Train Loss는 계속 주는데 Validation Loss가 늘거나 Validation mAP가 멈추거나 떨어지면 과적합입니다. 데이터를 더 모으거나 Augmentation을 쓰고, `patience`로 Validation 성능이 한동안 안 오르면 학습을 일찍 멈춥니다.

### 자세히
| 하이퍼파라미터 | 너무 작으면 / 크면 | Ultralytics 기본값 |
|---|---|---|
| Epoch | 과소적합 / 과적합 | `epochs=100` |
| Batch Size | 클수록 GPU 메모리 많이 씀 | `batch=16` (`-1`이면 자동) |
| Learning Rate | 느리게 수렴 / 진동·발산 | `lr0`, `lrf`로 지정, 점점 감소 |

| Loss 곡선 | 해석 |
|---|---|
| Train, Val 같이 감소 | 정상 학습 |
| Train 감소, Val 증가 | 과적합 |
| 둘 다 높은 값에서 정체 | 과소적합이거나 데이터·라벨 문제 |

- `optimizer='auto'`는 총 반복 횟수에 따라 Optimizer와 LR을 자동으로 고른다. 실습에서도 `lr0=0.01`을 무시하고 AdamW(lr=0.002)를 골랐다는 로그가 떴다.

### 꼬리질문
#### 탐지에서 자주 쓰는 Augmentation은 무엇인가요?
뒤집기, 크기 조절, 색상 변화 외에 이미지 4장을 이어 붙이는 Mosaic를 많이 씁니다. 한 장에 다양한 크기와 배경의 객체가 들어가서 작은 객체와 문맥 학습에 도움이 되고, Ultralytics는 마지막 몇 에폭에서 Mosaic를 꺼서 실제 이미지 분포에 맞춥니다.

#### 데이터가 적으면 모델 크기는 어떻게 고르나요?
큰 모델은 파라미터가 많아 적은 데이터를 외우기 쉬우므로 작은 모델부터 시작하고, Validation 성능을 보며 키웁니다. 사전학습 가중치에서 시작하는 것도 과적합 위험을 줄여 줍니다.

## Q. 탐지 데이터셋은 어떻게 구성하고, YOLO 라벨 형식과 data.yaml에는 무엇을 적나요?
id: det-dataset-format
type: 개념
level: 기본
tags: 데이터셋, YOLO 라벨, data.yaml, Annotation
source: 9. 탐지 데이터셋 만들기 › 데이터셋의 구성과 라벨 형식, data.yaml

### 30초 답변
탐지 데이터셋은 이미지, 이미지별 라벨 파일, Class ID와 이름의 대응, Train/Val/Test 분할 정보로 이뤄집니다. YOLO 라벨은 이미지마다 이름이 같은 `.txt` 파일 하나이고, 한 줄에 객체 하나를 `class_id x_center y_center width height` 형식의 0~1 정규화 값으로 적습니다. `data.yaml`에는 데이터셋 경로, train/val/test 이미지 폴더, Class 이름 목록을 적고, `names` 순서가 라벨의 class_id와 일치해야 합니다.

### 자세히
```
labels/train/img001.txt
0 0.481 0.633 0.212 0.418
2 0.150 0.720 0.180 0.240
```

```yaml
path: ./dataset
train: images/train
val: images/val
test: images/test
names:
  0: helmet
  1: head
```

- `images/`와 `labels/`의 하위 구조가 같아야 한다. Ultralytics가 이미지 경로의 `images`를 `labels`로 바꿔 라벨을 찾는다.
- 객체가 없는 이미지는 라벨 파일을 비우거나 만들지 않으면 배경 이미지로 학습에 쓰인다.
- 알약 라벨 한 줄 `0 0.177344 0.804167 0.110937 0.150926`을 1920×1080으로 되돌리면 xyxy `[234, 787, 447, 950]`이었다.

### 꼬리질문
#### `names` 순서와 라벨의 class_id가 어긋나면 어떻게 되나요?
에러 없이 학습은 되지만 모델이 helmet을 head라는 이름으로 배우는 식으로 이름이 뒤바뀝니다. CVAT 등에서 내보낸 뒤 Class ID 순서와 `names` 순서가 맞는지 꼭 확인해야 합니다.

#### 배경 이미지를 일부러 넣는 이유는 무엇인가요?
객체가 없는 장면에서 헛것을 검출하는 FP를 줄이기 위해서입니다. 실제 환경에서 자주 보이는 배경을 넣어 두면 그것을 객체로 착각할 가능성이 줄어듭니다.

## Q. Train, Validation, Test 데이터는 각각 무슨 역할을 하고 왜 나누나요?
id: det-train-val-test
type: 비교
level: 기본
tags: Train, Validation, Test, 데이터 분할
source: 9. 탐지 데이터셋 만들기 › Train / Validation / Test

### 30초 답변
Train은 가중치를 학습하고, Validation은 학습 중 에폭마다 성능을 재서 하이퍼파라미터와 `best.pt`를 고르는 데 쓰고, Test는 모든 선택이 끝난 뒤 최종 성능을 한 번 잽니다. 학습에 쓴 데이터로 평가하면 성능이 실제보다 높게 나옵니다. Validation도 모델을 고르는 데 썼으니 조금 낙관적일 수 있어서, 한 번도 쓰지 않은 Test로 최종 성능을 잽니다.

### 자세히
| | 쓰임 | 가중치 갱신 |
|---|---|---|
| Train | 학습 | O |
| Validation | 에폭별 평가, 모델·하이퍼파라미터 선택 | X |
| Test | 최종 성능 | X |

- 보통 7:2:1이나 8:1:1로 나눈다.
- 저장된 모델로 Test 평가: `model.val(data="data.yaml", split="test")`

### 꼬리질문
#### 영상에서 뽑은 프레임으로 데이터셋을 만들 때 주의할 점은?
연속 프레임은 거의 같은 이미지라 무작위로 나누면 Train과 Test에 사실상 같은 장면이 들어가 성능이 부풀려집니다. 영상이나 촬영 장소 단위로 나눠야 합니다.

## Q. 라벨링 기준을 미리 정하고 일관되게 지키는 것이 왜 중요한가요?
id: det-labeling-criteria
type: 이유
level: 기본
tags: 라벨링, Annotation, CVAT, 데이터 품질
source: 9. 탐지 데이터셋 만들기 › 라벨링 기준과 CVAT

### 30초 답변
모델은 라벨 파일을 그대로 정답으로 배우기 때문입니다. 라벨링하지 않은 객체는 배경으로 학습되어 Recall이 내려가고, 박스 경계가 들쭉날쭉하면 위치 예측이 부정확해져 mAP50-95가 떨어지고, Class를 잘못 붙이면 Class끼리 헷갈립니다. 평가 데이터의 라벨이 틀리면 잰 성능 자체를 믿을 수 없습니다.

### 자세히
- 미리 정할 것: 보이는 부분만 그릴지 가려진 부분까지 추정할지, 이미지 경계에서 잘린 객체 처리, 몇 픽셀 이하는 라벨링하지 않을지
- 좋은 박스: 네 변이 객체 바깥 경계에 맞닿고, 배경을 불필요하게 넣지 않고, 객체를 잘라 먹지 않는다. 객체 하나에 박스 하나.
- CVAT: 오픈소스 웹 라벨링 도구. Project > Task > Job 구조로 작업을 나누고 검수한다. YOLO 형식으로 내보낸 뒤 폴더 구조와 Class 순서를 확인한다.
- YOLO 탐지 형식은 Label만 저장하고 Attribute(color, occluded 등)는 내보내지 않는다. 모델이 구분해야 하는 건 Label로 정의해야 한다.

### 꼬리질문
#### 라벨링 기준이 평가 숫자까지 바꾼 사례가 있나요?
알약 모델에서 다른 포장에 반쯤 가려진 알약에 0.32, 0.56 같은 낮은 점수의 박스가 붙었습니다. 이런 애매한 객체를 라벨에 넣었느냐에 따라 같은 박스가 TP도 FP도 될 수 있습니다.

#### 여러 사람이 라벨링할 때 일관성은 어떻게 확인하나요?
같은 이미지 일부를 여러 작업자가 라벨링하게 해서 박스 IoU나 Class 일치율을 비교하고, 검수 단계를 둬서 기준과 다른 라벨을 고칩니다. CVAT에는 작업 배정과 검수 기능이 있습니다.

## Q. YOLO 학습 로그의 box_loss, cls_loss, dfl_loss는 각각 무엇인가요?
id: det-loss
type: 개념
level: 심화
tags: Loss, CIoU, BCE, DFL
source: 10. 내 데이터로 학습해 보기 › 학습 로그와 Loss

### 30초 답변
box_loss는 예측 박스와 GT 박스의 위치 차이로, IoU를 변형한 CIoU를 씁니다. cls_loss는 Class 예측 오차로 Binary Cross Entropy를 쓰고, dfl_loss는 박스 경계 위치를 하나의 값 대신 확률 분포로 예측하게 하는 Distribution Focal Loss입니다. 전체 Loss는 세 값에 가중치를 곱해 더한 것이고, 셋 다 작을수록 GT에 가깝습니다.

### 자세히
| Loss | 다루는 것 | 방식 |
|---|---|---|
| box_loss | 박스 위치·크기 | CIoU = IoU + 중심점 거리 + 가로세로 비율 일치도 |
| cls_loss | Class 점수 | Class마다 sigmoid + BCE |
| dfl_loss | 경계 거리 (l, t, r, b) | 여러 구간에 대한 분포로 예측, 정답 근처 구간에 확률이 모이게 학습 |

```
CIoU loss = 1 - IoU + ρ²(중심점 거리) / c²(감싸는 박스 대각선) + αv(비율 차이)
```

- Train Loss는 가중치 갱신에 쓰이고, Validation Loss는 갱신에 쓰지 않는다. 둘을 같이 봐야 한다.

### 꼬리질문
#### Class 점수에 softmax 대신 sigmoid를 쓰는 이유는?
Class마다 독립적으로 0~1 점수를 내서 합이 1일 필요가 없게 하려는 것입니다. 배경처럼 어느 Class도 아닌 위치는 모든 점수를 낮게 낼 수 있고, 겹치는 Class 체계에서도 여러 Class가 함께 높을 수 있습니다.

#### 경계 위치를 분포로 예측하면 무엇이 좋나요?
가려지거나 흐릿해서 경계가 애매한 경우, 값 하나로 억지로 맞추는 대신 여러 위치에 확률을 나눠 둘 수 있습니다. 최종 경계는 분포의 기대값으로 계산해서 더 부드럽고 정밀한 위치를 얻습니다.

## Q. 탐지에서 클래스 불균형 문제는 어떤 형태로 나타나고, 어떻게 다루나요?
id: det-class-imbalance
type: 이유
level: 심화
tags: 클래스 불균형, Focal Loss, 배경, mAP
source: 보충 (블로그 밖 내용)

### 30초 답변
탐지에는 두 종류의 불균형이 있습니다. 첫째는 전경과 배경의 불균형으로, 8,400개 후보 위치 중 객체를 맡는 위치는 몇 개뿐이라 쉬운 배경 예측이 Loss를 지배합니다. 둘째는 Class 사이의 불균형으로, 데이터가 적은 Class는 AP가 낮게 나오고 mAP가 Class를 같은 비중으로 평균하니 전체 점수도 끌어내립니다.

### 자세히
| 불균형 | 대응 |
|---|---|
| 전경 vs 배경 | Focal Loss, 학습 샘플 할당 방식 개선, Two-stage의 후보 단계 |
| Class 사이 | 적은 Class 데이터 추가 수집, 오버샘플링, Augmentation, Class별 Loss 가중치 |

```
Focal Loss = -α (1 - p_t)^γ · log(p_t)        보통 γ = 2
```

- Focal Loss는 2017년 RetinaNet에서 제안됐다. 이미 잘 맞히는 쉬운 예시(p_t가 1에 가까움)의 Loss를 (1 - p_t)^γ로 크게 줄여서, 어려운 예시에 학습이 집중되게 한다.

### 꼬리질문
#### Class 불균형이 있을 때 결과는 어떻게 확인하나요?
전체 mAP만 보지 말고 Class별 AP와 Confusion Matrix를 봅니다. 데이터가 적은 Class가 background 행에 몰리면 자주 놓친다는 뜻이고, 다른 Class와 혼동이 크면 그 둘의 라벨링 기준부터 다시 봅니다.

## Q. 사전학습 가중치, 처음부터 학습, Backbone 고정을 직접 비교해 본 결과를 설명해 주세요.
id: det-exp-pretrain-scratch-freeze
type: 경험
level: 심화
tags: 전이학습, Training from Scratch, Freeze, 실험
source: 10. 내 데이터로 학습해 보기 › 사전학습 가중치 vs 처음부터 vs Backbone 고정

### 30초 답변
알약 데이터(train 92장)로 같은 설정에 시작 가중치만 바꿔 세 번 학습했습니다. 사전학습에서 시작한 모델은 첫 에폭부터 mAP50-95 0.38이었고 마지막에 0.792였는데, 처음부터 학습한 모델은 15 에폭까지 0에 붙어 있다가 마지막에 0.535에 그쳤습니다. Backbone 앞 10개 층을 고정한 모델도 0.780으로 사전학습과 거의 같아서, COCO에서 배운 Feature가 알약에도 그대로 통한다는 것을 확인했습니다.

### 자세히
- 설정: medical-pills (Class 1개, train 92장, val 23장), 30 에폭, batch 16, imgsz 640

| 시작 가중치 | 마지막 mAP50 | 마지막 mAP50-95 |
|---|---|---|
| pretrained (`yolo11n.pt`, 전체 학습) | 0.993 | 0.792 |
| freeze10 (앞 10개 층 고정) | 0.994 | 0.780 |
| scratch (`yolo11n.yaml`, 무작위) | 0.954 | 0.535 |

- 첫 에폭 Validation box_loss: pretrained 1.16, scratch 5.8. "Loss가 낮은 상태에서 출발한다"는 말이 숫자로 보였다.
- scratch는 mAP50은 높고 mAP50-95는 낮았다. 알약을 찾긴 하지만 박스 위치가 부정확했다.

### 꼬리질문
#### Backbone을 고정하면 어떤 장점이 있나요?
학습할 가중치가 적어 메모리와 계산이 줄고, 데이터가 적을 때 Backbone이 과적합되는 것도 막을 수 있습니다. 대신 새 데이터가 COCO와 많이 다르면(의료 영상, 적외선 등) 고정된 Feature가 안 맞아 성능이 떨어질 수 있습니다.

#### 처음부터 학습한 모델에 더 많은 에폭을 주면 따라잡을까요?
이번에는 30 에폭에서 끝나서 확인하지 못했습니다. 다만 92장으로는 일반적인 시각 Feature를 처음부터 배우기에 데이터가 턱없이 부족하므로, 에폭만 늘리면 따라잡기 전에 과적합될 가능성이 큽니다.

## Q. 학습 결과의 best.pt와 last.pt는 무엇이 다르고, 각각 언제 쓰나요?
id: det-best-vs-last
type: 비교
level: 기본
tags: best.pt, last.pt, fitness, 체크포인트
source: 10. 내 데이터로 학습해 보기 › best.pt와 last.pt

### 30초 답변
last.pt는 마지막 에폭이 끝난 시점의 가중치로, 중단된 학습을 이어 갈 때 씁니다. best.pt는 Validation 성능이 가장 좋았던 에폭의 가중치로, 추론과 배포에는 이걸 씁니다. Ultralytics의 '가장 좋음' 기준은 0.1 × mAP50 + 0.9 × mAP50-95인 fitness입니다.

### 자세히
| | last.pt | best.pt |
|---|---|---|
| 저장 시점 | 매 에폭 끝 (덮어씀) | fitness가 최고일 때 |
| 쓰임 | `resume=True`로 학습 재개 | 추론, 배포, 최종 평가 |

- 알약 pretrained 실행: 29 에폭이 fitness 0.813으로 best.pt, 30 에폭(0.812)은 last.pt로만 남았다.
- mAP50은 10 에폭에 0.975를 넘긴 뒤 거의 그대로였고, mAP50-95는 끝까지 조금씩 올랐다. 찾는 건 빨리 배우고 정확한 박스는 늦게 배운다.

### 꼬리질문
#### fitness에서 mAP50-95에 9배 비중을 두는 이유는?
mAP50은 학습 초반에 금방 포화되어 에폭 사이 차이를 잘 구분하지 못하고, 위치 정확도까지 반영하는 mAP50-95가 모델 품질 차이를 더 잘 보여 주기 때문입니다.

#### best.pt를 Validation으로 골랐는데 성능 보고도 Validation 숫자로 해도 되나요?
Validation으로 모델을 골랐기 때문에 그 숫자는 조금 낙관적입니다. 최종 성능은 학습과 선택에 쓰지 않은 Test 데이터로 재서 보고합니다.

## Q. 같은 알약 사진에 COCO 사전학습 모델과 Fine-tuning 모델을 돌려 보니 어땠나요?
id: det-exp-coco-vs-finetuned
type: 경험
level: 기본
tags: Fine-tuning, Custom 모델, 실험
source: 10. 내 데이터로 학습해 보기 › 사전학습 모델과 Fine-tuning 모델 비교

### 30초 답변
학습에 안 쓴 val 사진 한 장에 두 모델을 돌렸는데, COCO 사전학습 모델은 알약 두 개를 `cup`(0.38, 0.27)으로 잡았습니다. COCO에 알약 Class가 없으니 가장 비슷한 컵으로 답한 것입니다. Fine-tuning 모델은 `{0: 'pill'}` 하나만 아는 모델이 되어 17개를 찾았지만, 대신 사람이나 버스는 더 이상 찾지 못합니다.

### 자세히
| 모델 | 아는 Class | 알약 사진 결과 |
|---|---|---|
| COCO 사전학습 | 80개 | `cup` 2개 (0.38, 0.27) |
| Fine-tuning | `pill` 1개 | 17개 |

- 모델은 학습한 Class 중에서만 답한다. 모르는 객체는 엉뚱한 Class로 잡거나 놓친다.
- 두 모델에 공통 Class가 있다면 같은 Test 데이터에서 mAP를 계산해 숫자로 비교하면 된다.

### 꼬리질문
#### 사전학습 모델이 알약을 컵으로 잡은 것을 어떻게 해석하나요?
모델이 "작고 둥근 물체가 있다"는 위치와 형태 정보는 이미 잘 뽑는다는 뜻입니다. Class 이름만 새로 알려 주면 되니, 적은 데이터와 30 에폭으로도 Fine-tuning이 잘 된 것과 맞닿아 있습니다.

## Q. Precision, Recall, AP를 직접 계산해서 라이브러리 값과 맞춰 본 경험을 설명해 주세요.
id: det-exp-manual-ap
type: 경험
level: 심화
tags: AP, PR Curve, mAP50-95, 실험
source: 10. 내 데이터로 학습해 보기 › Precision, Recall, AP를 직접 계산해서 맞춰 보기

### 30초 답변
알약 val 23장(알약 399개)에 conf 0.001로 예측 1,451개를 모은 뒤, 직접 짠 매칭 함수로 TP/FP를 판정하고 PR Curve를 그려 101점 보간으로 AP를 계산했습니다. AP50은 0.993으로 `model.val()`과 같았고, mAP50-95는 0.801로 라이브러리의 0.793과 조금 달랐습니다. 제 함수는 점수 높은 예측부터 짝을 짓고 Ultralytics는 IoU가 큰 짝부터 정하는 방식이라, 높은 IoU 기준에서 짝이 몇 개 달라진 것으로 봤습니다.

### 자세히
| IoU 기준 | AP (직접 계산) |
|---|---|
| 0.50 | 0.993 |
| 0.75 | 0.983 |
| 0.85 | 0.734 |
| 0.95 | 0.031 |

- IoU 0.95에서는 곡선이 Recall 0.1에서 끝났다. 위치는 거의 다 맞히지만 0.95 이상으로 딱 맞게 친 박스는 열 개 중 하나꼴이었다.
- Threshold별 그래프에서 0.25~0.6 구간은 Precision, Recall 모두 0.95 이상이었고, 0.8을 넘기면 Recall이 급격히 무너졌다.

### 꼬리질문
#### 이 실험에서 Threshold를 고른다면 어디로 하겠나요?
Precision과 Recall이 모두 0.95 이상인 0.25~0.6 구간이면 큰 차이가 없어서, 기본값 0.25 근처를 쓰고 놓치는 것과 헛검출 중 무엇이 더 비싼지에 따라 이 구간 안에서 조정하겠습니다. 0.8 이상은 Recall이 급격히 떨어지므로 피합니다.

#### 직접 계산해 보면서 새로 알게 된 점은 무엇인가요?
mAP50-95가 mAP50보다 낮은 이유가 곡선으로 보였습니다. IoU 기준이 올라갈수록 같은 예측이 TP에서 FP로 바뀌면서 곡선이 안쪽으로 말려 들어오고, 0.85를 넘기면서 급격히 떨어졌습니다.

## Q. Confusion Matrix로 탐지 모델의 문제를 어떻게 진단했나요?
id: det-exp-confusion-matrix
type: 경험
level: 심화
tags: Confusion Matrix, 오류 분석, FP, FN
source: 10. 내 데이터로 학습해 보기 › Confusion Matrix, 결과 해석 정리

### 30초 답변
탐지의 Confusion Matrix에는 Class 칸 외에 background 행과 열이 붙는데, background 열은 객체가 없는 곳에 친 박스(FP), background 행은 검출하지 못한 GT(FN)입니다. 알약 모델은 399개 중 396개를 맞게 찾았고, 3개를 놓쳤고, 알약이 아닌 곳에 17번 박스를 쳤습니다. FN보다 FP가 많으니, 이 모델을 고친다면 놓치는 쪽보다 헛것을 보는 쪽부터 확인해야 한다고 판단했습니다.

### 자세히
- conf 0.25, IoU 0.5 기준: TP 396, FN 3, FP 17 → Precision 0.959, Recall 0.992

| 증상 | 의미 | 볼 곳 |
|---|---|---|
| Precision 낮음 | 배경이나 다른 객체를 잘못 검출 | background 열, Class 간 혼동 |
| Recall 낮음 | 놓치는 객체가 많음 | background 행, 라벨 누락, 작은·가려진 객체 |
| mAP50 높고 mAP50-95 낮음 | 찾지만 박스 위치가 부정확 | 박스 라벨 품질, 학습량 |
| 둘 다 낮음 | 검출 자체가 부족 | 데이터 양, 모델, 설정 전반 |

### 꼬리질문
#### Class가 여러 개라면 무엇을 더 보나요?
대각선 밖의 칸을 봅니다. 두 Class 사이 값이 크면 그 둘을 헷갈리는 것이니, 두 Class의 라벨링 기준이 명확한지와 각 Class의 학습 데이터 양부터 다시 봅니다.

#### FP 17개는 모두 모델의 잘못인가요?
꼭 그렇지는 않습니다. 반쯤 가려진 알약처럼 라벨에서 빠진 애매한 객체에 박스를 친 경우는, 라벨 기준에 따라 TP가 될 수도 있습니다. 그래서 FP 이미지를 직접 열어 모델 문제인지 라벨 문제인지 나눠 봐야 합니다.
