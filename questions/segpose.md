# Segmentation·Pose
blog: seg_pose_blog/seg_pose_blog.md
blog_title: [CV 4일차] 픽셀을 칠하고 관절을 찍는 법: YOLO11 Segmentation과 Pose Estimation

## Q. Detection, Segmentation, Pose Estimation은 출력이 어떻게 다르고, YOLO11에서는 구조가 어떻게 다른가요?
id: seg-three-tasks
type: 비교
level: 기본
tags: Detection, Segmentation, Pose, Head, YOLO11
source: 1. 같은 YOLO, 세 가지 Task

### 30초 답변
Detection은 박스와 클래스만, Segmentation은 박스에 더해 객체마다 픽셀 마스크를, Pose는 박스에 더해 사람마다 관절 좌표를 출력합니다. YOLO11에서 세 모델은 Backbone과 Neck 구조가 같고 마지막 Head만 다릅니다. Seg와 Pose는 Detection Head 위에 마스크나 키포인트 예측을 하나 더 얹은 구조라서 둘 다 박스를 함께 내놓습니다.

### 자세히
| Task | 가중치 | 출력 | 파라미터 수 (n) |
|---|---|---|---|
| Detection | `yolo11n.pt` | 박스 | 2,624,080 |
| Segmentation | `yolo11n-seg.pt` | 박스 + 픽셀 마스크 | 2,876,848 |
| Pose | `yolo11n-pose.pt` | 박스 + 키포인트 17개 | 2,874,462 |

- Seg Head: 이미지당 기본 마스크 32장 + 객체마다 계수 32개
- Pose Head: 박스마다 관절 17개의 `(x, y, conf)`
- 구조가 비슷해도 가중치는 파일마다 따로 학습한 것이라 서로 바꿔 쓸 수 없다.

### 꼬리질문
#### Pose 모델에 개 사진을 넣으면 개를 찾나요?
못 찾습니다. Detection과 Seg는 COCO 80 클래스로 학습했지만 Pose는 COCO-Pose의 person 한 클래스로만 학습해서 `names`가 `{0: 'person'}` 하나뿐입니다. 직접 개 사진을 넣었을 때 Seg는 개, 자전거, 자동차 세 개를 찾았고 Pose는 0개였습니다.

#### Seg 모델이 있으면 Detection 모델은 필요 없지 않나요?
Seg 결과에도 박스가 있으니 기능상으로는 대신할 수 있습니다. 다만 마스크 Head만큼 파라미터와 후처리 비용이 늘어나므로, 박스만 필요하면 Detection 모델이 더 가볍습니다.

## Q. Segmentation이란 무엇이고, 박스 대신 마스크가 필요한 경우는 언제인가요?
id: seg-pixel-classification
type: 개념
level: 기본
tags: Segmentation, 픽셀 분류, 활용
source: 2. Segmentation › 픽셀마다 분류한다

### 30초 답변
Segmentation은 이미지의 픽셀 하나하나를 분류해서 객체의 윤곽까지 예측하는 작업입니다. 박스는 물체가 대충 이 네모 안에 있다는 것까지만 알려 주지만, 마스크는 어느 픽셀이 그 물체인지 알려 줍니다. 그래서 화상회의 배경 분리, 의료 영상의 종양 윤곽, 자율주행의 차선과 인도 구분처럼 경계 모양이 중요한 일에 씁니다.

### 자세히
- 분류: 이미지 1장 → 클래스 1개
- Detection: 이미지 1장 → 박스 N개 + 클래스
- Segmentation: 이미지 1장 → 픽셀마다 클래스 (H×W개의 분류 문제)

```
semantic seg 출력: (C, H, W) 점수 → 픽셀마다 argmax → (H, W) 클래스 맵
```

- 박스는 물체 바깥 배경까지 포함하므로, 기울어진 물체나 가늘고 긴 물체일수록 박스 안의 배경 비율이 커진다.

### 꼬리질문
#### 픽셀마다 분류한다면 일반 분류 CNN을 그대로 쓰면 되나요?
분류 CNN은 풀링과 stride로 해상도를 줄이고 마지막에 FC 층으로 공간 정보를 없애기 때문에 그대로는 픽셀별 출력을 못 냅니다. 그래서 FC 층을 합성곱으로 바꾸고 줄어든 특징 맵을 다시 원본 크기로 키우는 구조(FCN, U-Net 등)를 씁니다.

## Q. Semantic, Instance, Panoptic Segmentation의 차이를 설명해 보세요.
id: seg-semantic-instance-panoptic
type: 비교
level: 기본
tags: Semantic, Instance, Panoptic, Segmentation
source: 2. Segmentation › Semantic과 Instance (Panoptic은 보충)

### 30초 답변
Semantic Segmentation은 픽셀마다 클래스만 정해서, 사람이 3명이어도 '사람' 영역 하나로 칠합니다. Instance Segmentation은 사람1, 사람2, 사람3을 각각 다른 마스크로 나누는 방식으로, Detection의 개별 구분과 Segmentation의 픽셀 정밀도를 합친 것입니다. Panoptic Segmentation은 둘을 합쳐서 모든 픽셀에 클래스를 주고, 셀 수 있는 물체에는 개체 번호까지 붙입니다.

### 자세히
| | Semantic | Instance | Panoptic |
|---|---|---|---|
| 픽셀마다 | 클래스 | 객체에 속한 픽셀만 개체별 마스크 | 클래스 + 개체 ID |
| 같은 클래스 여러 개 | 한 덩어리 | 따로 구분 | 따로 구분 |
| 하늘, 도로 같은 배경 | 칠함 | 보통 다루지 않음 | 칠함 |
| 대표 모델 | FCN, U-Net, DeepLab | Mask R-CNN, YOLO-seg | Panoptic FPN 등 |

- YOLO11-seg는 Instance 방식이라 결과가 객체마다 박스 1개 + 마스크 1장의 짝으로 나온다.
- 블로그의 Semantic 그림은 이 인스턴스 마스크들을 일부러 하나로 합쳐 칠한 것이다.
- 보충: Panoptic에서는 셀 수 있는 물체를 things, 하늘이나 도로처럼 셀 수 없는 영역을 stuff라고 부른다.

### 꼬리질문
#### 인스턴스 마스크들끼리는 겹칠 수 있나요?
Instance 방식은 객체마다 독립된 마스크를 내므로 두 마스크가 같은 픽셀을 함께 칠할 수 있습니다. Semantic과 Panoptic은 픽셀마다 답이 하나라서 겹침이 없습니다.

#### 인스턴스 마스크로 Semantic 결과를 만들 수 있나요?
같은 클래스 마스크를 OR로 합치면 됩니다. 반대로 Semantic 결과에서 맞닿은 같은 클래스 객체를 개체별로 나누는 것은 정보가 없어서 어렵습니다.

## Q. YOLO-seg는 객체마다 마스크를 어떻게 만드나요?
id: seg-proto-coefficients
type: 개념
level: 심화
tags: YOLO-seg, prototype, 마스크 계수, YOLACT
source: 1. 같은 YOLO, 세 가지 Task › Head가 하는 일

### 30초 답변
YOLO-seg는 이미지 한 장에 대해 공용 기본 마스크(prototype) 32장을 만들고, 검출한 객체마다 계수 32개를 함께 예측합니다. 객체 마스크는 32장을 계수만큼 곱해서 더한 뒤 sigmoid를 씌우고 박스 영역으로 잘라 낸 결과입니다. 객체마다 마스크를 통째로 그리지 않고 숫자 32개만 예측하면 되니 가볍고, 이 방식은 YOLACT에서 나온 아이디어와 같습니다.

### 자세히
```
mask_i = sigmoid( Σ_k c_ik · P_k )   # P: (32, H/4, W/4), c_i: 객체 i의 계수 32개
mask_i = crop(mask_i, box_i) > 0.5
```

- `proto` 출력을 hook으로 꺼내 보니 bus.jpg에서 `(32, 160, 120)`, 입력 640×480의 1/4 해상도였다.
- 어떤 장은 사람만 밝고(#9, #12), 어떤 장은 버스 몸통(#15, #17), 어떤 장은 바닥이나 벽(#0, #10)을 잡았다.
- 사람 마스크라면 사람이 밝은 장에 큰 계수를 주는 식으로 조합한다.

### 꼬리질문
#### 마지막에 박스로 잘라 내는 이유는 무엇인가요?
prototype은 클래스나 객체 구분 없이 공용이라, 계수를 조합한 결과가 박스 밖의 다른 비슷한 물체까지 밝힐 수 있습니다. 박스로 잘라 내면 그런 픽셀이 다른 객체 마스크에 섞이는 것을 막을 수 있습니다.

#### prototype이 32장뿐인데 80개 클래스 마스크를 다 표현할 수 있나요?
prototype은 클래스별 템플릿이 아니라 위치, 경계, 전경과 배경 같은 성분을 담은 기저라서 계수 조합으로 다양한 모양을 만듭니다. 클래스 판단은 박스 쪽 분류 Head가 따로 합니다.

## Q. YOLO 추론 결과에서 박스와 마스크는 어떻게 꺼내나요?
id: seg-results-object
type: 개념
level: 기본
tags: ultralytics, Results, Boxes, Masks
source: 2. Segmentation › 출력은 Results 객체의 리스트 / Boxes와 Masks

### 30초 답변
추론 결과는 `Results` 객체의 리스트로 나오고, 이미지 1장이면 `results[0]`으로 꺼냅니다. 여기서 `boxes`는 모든 작업에 공통이고 `masks`는 seg 모델, `keypoints`는 pose 모델에서만 채워지고 나머지는 `None`입니다. `boxes.cls[i]`와 `masks.data[i]`는 같은 순서로 짝지어 있어서 클래스 번호를 보고 원하는 객체의 마스크만 고를 수 있습니다.

### 자세히
```python
result = model("dog.jpg")[0]
if result.masks is not None:          # 아무것도 못 찾으면 None
    cls = result.boxes.cls            # (N,)
    masks = result.masks.data         # (N, h, w) 0/1, 추론 해상도
    polys = result.masks.xy           # 객체마다 (점 개수, 2), 원본 픽셀 좌표
    m = masks[0].cpu().numpy()        # GPU 텐서는 .cpu()가 먼저
```

| | `masks.data` | `masks.xy` |
|---|---|---|
| 형태 | 0/1 픽셀 배열 | 외곽선 점 목록 |
| 기준 | 추론 해상도 (dog.jpg에서 `(3, 480, 640)`) | 원본 이미지 픽셀 |
| 예 | | 강아지 321점, 자전거 672점 |

### 꼬리질문
#### `conf`, `classes` 같은 추론 파라미터는 결과를 어떻게 바꾸나요?
`conf`보다 신뢰도가 낮은 예측은 버리고, `classes=[0]`이면 person만 남깁니다. 강아지 사진에 `conf=0.7`을 줬을 때 줄어들 줄 알았는데 자동차 신뢰도가 0.70이라 3개 그대로였고, 0.8로 올려야 2개가 됐습니다.

#### `masks is not None` 검사를 빼면 어떤 문제가 생기나요?
seg 모델이라도 아무것도 검출하지 못한 프레임에서는 `masks`가 `None`이라 속성을 꺼내는 순간 에러가 납니다. 웹캠 코드에서는 사람이 화면 밖으로 나가는 순간 프로그램이 멈춥니다.

## Q. masks.data를 원본 크기로 resize했더니 마스크가 살짝 어긋났다고 했는데, 원인이 뭐였나요?
id: seg-letterbox-resize
type: 경험
level: 심화
tags: letterbox, 패딩, retina_masks, 후처리
source: 2. Segmentation › resize로 키운 마스크가 살짝 어긋나는 이유

### 30초 답변
원인은 letterbox 패딩이었습니다. 720×1280 사진은 긴 변 640에 맞추면 360×640인데, YOLO가 높이와 너비를 32의 배수로 맞추려고 위아래에 12px씩 여백을 붙여 384×640으로 추론하고 마스크도 여백을 포함한 크기로 나옵니다. 이걸 그대로 원본 크기로 늘리면 여백까지 늘어나서 마스크가 세로로 눌리고, 머리 위와 발밑 경계가 안쪽으로 들어옵니다. `retina_masks=True`로 여백을 잘라 낸 원본 크기 마스크를 받아서 해결했습니다.

### 자세히
```
720×1280 → 360×640 → 패딩 12px×2 → 384×640 (masks.data 크기)
cv2.resize(mask, (1280, 720))   # 여백까지 늘어남, 크기 인자는 (너비, 높이)
```

- resize 방식과 `retina_masks` 방식 마스크의 IoU는 0.905. 같은 예측인데 후처리만으로 10% 가까이 차이가 났다.
- 놓친 영역은 머리 꼭대기와 아래쪽 가장자리에 몰려 있었다.
- 576×768 강아지 사진은 줄이면 정확히 480×640이라 여백이 없어서 문제가 안 보였다.

### 꼬리질문
#### 왜 32의 배수로 맞추나요?
Backbone이 stride를 거듭해 최대 1/32까지 줄이기 때문에, 입력이 32의 배수가 아니면 특징 맵 크기가 정수로 떨어지지 않아 Neck에서 업샘플한 맵과 크기가 맞지 않습니다.

#### retina_masks 없이 직접 고치려면 어떻게 하나요?
여백 크기를 계산해서 `masks.data`에서 패딩 줄을 먼저 잘라 낸 뒤 원본 크기로 resize하면 됩니다. 사진마다 여백 여부를 따지기 번거로워서 원본 위에 마스크를 쓸 때는 `retina_masks=True`를 켜 두는 쪽이 편합니다.

## Q. Segmentation의 정답(Ground Truth)은 어떤 형식이고, 어떤 한계가 있나요?
id: seg-gt-polygon
type: 개념
level: 기본
tags: Ground Truth, 폴리곤, 라벨링
source: 3. Segmentation 모델 평가 › 정답(Ground Truth)은 폴리곤이다

### 30초 답변
Segmentation 정답 파일에는 보통 객체 외곽선의 점 좌표, 즉 폴리곤이 들어 있고, 평가할 때 이것을 픽셀 마스크로 바꿔 예측과 비교합니다. 폴리곤은 사람이 윤곽을 따라 점을 찍어 만든 근사값이라 머리카락이나 손가락 같은 세밀한 경계는 정답 자체가 거칠 수 있습니다. 그래서 모델이 정답보다 더 정확하게 따도 점수가 깎일 수 있습니다.

### 자세히
```
YOLO seg 라벨 한 줄: <class> x1 y1 x2 y2 ... xn yn   (0~1 정규화 좌표)
평가: 폴리곤 → 래스터화 → 0/1 마스크 → 예측 마스크와 픽셀 비교
```

- 폴리곤은 점 몇십 개로 저장되니 픽셀 마스크보다 훨씬 가볍다.
- 보충: COCO는 객체를 폴리곤으로 저장하고, 여러 사람이 뭉친 군중(iscrowd) 영역은 RLE(run-length encoding) 마스크로 저장한다.

### 꼬리질문
#### 라벨이 거칠면 평가 지표에 어떤 영향이 있나요?
정밀한 경계를 요구하는 높은 IoU 임계값에서 특히 손해를 봅니다. 라벨 오차 몇 픽셀이 작은 객체에서는 IoU를 크게 떨어뜨리기 때문에, mAP50-95처럼 엄격한 기준에서는 라벨 품질이 성능 상한을 정합니다.

## Q. Mask IoU는 어떻게 계산하고, Box IoU와 무엇이 다른가요?
id: seg-mask-iou
type: 개념
level: 기본
tags: Mask IoU, 평가지표
source: 3. Segmentation 모델 평가 › Mask IoU

### 30초 답변
Mask IoU도 교집합 넓이를 합집합 넓이로 나누는 공식은 Box IoU와 같습니다. 차이는 넓이를 가로×세로가 아니라 픽셀 개수로 센다는 것입니다. 박스는 좌표 4개의 max, min으로 교집합을 구하지만, 마스크는 모양이 불규칙해서 정답과 예측을 같은 크기의 0/1 이미지로 겹쳐 놓고 AND와 OR로 픽셀을 셉니다.

### 자세히
```python
intersection = np.logical_and(gt_mask, pred_mask).sum()   # 둘 다 1
union = np.logical_or(gt_mask, pred_mask).sum()           # 하나라도 1
iou = intersection / union
```

- 손계산 예제: 정답 10칸, 예측 9칸, 교집합 7칸 → 합집합 10 + 9 − 7 = 12 → IoU = 7/12 = 0.58
- 두 마스크는 반드시 같은 해상도여야 한다. 그래서 앞의 letterbox 문제처럼 크기 맞추는 후처리가 점수에 직접 영향을 준다.

### 꼬리질문
#### Box IoU는 높은데 Mask IoU는 낮을 수 있나요?
그렇습니다. 박스는 바깥 네 변만 맞으면 되지만 마스크는 내부 모양까지 맞아야 합니다. 자전거처럼 구멍이 많고 가는 물체는 박스가 거의 일치해도 마스크 픽셀은 많이 어긋날 수 있습니다.

## Q. Dice 계수란 무엇인가요?
id: seg-dice
type: 개념
level: 기본
tags: Dice, F1, 평가지표, 의료영상
source: 3. Segmentation 모델 평가 › Dice

### 30초 답변
Dice는 두 영역의 겹침 정도를 0~1로 나타내는 지표로, 교집합을 2배 해서 정답 영역과 예측 영역의 합으로 나눕니다. 1이면 완벽히 일치, 0이면 전혀 겹치지 않음이고, 의료 영상 Segmentation에서 특히 많이 씁니다. 픽셀 단위로 보면 Precision과 Recall의 조화평균인 F1 score와 같은 식입니다.

### 자세히
```
Dice = 2 × |A ∩ B| / (|A| + |B|)
     = 2TP / (2TP + FP + FN)          # 픽셀 단위 F1
```

- 손계산 예제: 2 × 7 / (10 + 9) = 0.74 (같은 예제의 IoU는 0.58)
- 분모가 합집합이 아니라 두 넓이의 합이라 교집합이 두 번 들어간다.

### 꼬리질문
#### 의료 영상에서 Dice를 많이 쓰는 이유는 무엇인가요?
종양처럼 관심 영역이 이미지의 아주 작은 부분이라, 정답 배경을 맞힌 픽셀(TN)이 식에 들어가지 않는 Dice가 전경을 얼마나 잘 땄는지만 보여 줍니다. 손실 함수로 바꿔 쓰기도 쉬워서 학습과 평가를 같은 기준으로 맞추기 좋습니다.

## Q. IoU와 Dice는 어떤 관계인가요? Dice 0.8인 모델과 IoU 0.7인 모델은 어느 쪽이 더 좋나요?
id: seg-iou-dice-relation
type: 비교
level: 심화
tags: IoU, Dice, 지표 비교
source: 3. Segmentation 모델 평가 › IoU와 Dice는 같은 정보다

### 30초 답변
두 지표는 같은 겹침을 다른 눈금으로 잰 것이라 `Dice = 2·IoU / (1 + IoU)`로 서로 바로 바뀝니다. 0과 1을 빼면 Dice가 항상 IoU보다 큽니다. Dice 0.8을 IoU로 바꾸면 0.67이라 IoU 0.7인 모델보다 오히려 낮으므로, 모델끼리는 같은 지표로 바꿔서 비교해야 합니다.

### 자세히
```
I = 교집합, U = 합집합,  |A| + |B| = U + I
Dice = 2I / (U + I) = 2(I/U) / (1 + I/U) = 2·IoU / (1 + IoU)
IoU  = Dice / (2 − Dice)
```

| IoU | 0.5 | 0.67 | 0.7 |
|---|---|---|---|
| Dice | 0.67 | 0.80 | 0.82 |

- 실습에서 직접 구한 Dice와 `2·IoU/(1+IoU)`가 소수점 여섯째 자리까지 같았다.
- 단조 증가 관계라 마스크 한 쌍에서는 순위가 바뀌지 않는다.

### 꼬리질문
#### 그러면 둘 중 하나만 보면 되나요?
마스크 한 쌍에서는 그렇습니다. 다만 여러 이미지의 점수를 평균하면 변환이 비선형이라 평균 Dice를 변환한 값과 평균 IoU가 정확히 일치하지는 않으니, 보고할 때는 어떤 지표로 어떻게 평균했는지 밝혀야 합니다.

## Q. Segmentation에서 TP, FP, FN은 무엇을 세나요?
id: seg-pixel-tp-fp-fn
type: 개념
level: 기본
tags: TP, FP, FN, 픽셀 단위, Pixel Accuracy
source: 3. Segmentation 모델 평가 › TP, FP, FN도 픽셀 단위

### 30초 답변
마스크 한 쌍을 비교할 때는 픽셀 단위로 셉니다. 정답도 1이고 예측도 1이면 TP(맞힌 픽셀), 정답은 1인데 예측이 0이면 FN(놓친 픽셀), 정답은 0인데 예측이 1이면 FP(잘못 칠한 픽셀)입니다. 이 세 값으로 IoU와 Dice를 모두 표현할 수 있습니다.

### 자세히
| | 예측 1 | 예측 0 |
|---|---|---|
| 정답 1 | TP | FN |
| 정답 0 | FP | TN |

```
IoU  = TP / (TP + FP + FN)
Dice = 2TP / (2TP + FP + FN)
```

- 두 지표 모두 TN(배경을 배경으로 맞힌 픽셀)을 쓰지 않는다.
- 주의: mAP를 구할 때의 TP, FP는 픽셀이 아니라 예측 객체 하나 단위다. 객체 단위 판정에 픽셀 단위로 구한 Mask IoU를 쓴다.

### 꼬리질문
#### 맞힌 픽셀 비율인 Pixel Accuracy로 평가하면 안 되나요?
배경이 대부분인 이미지에서는 전부 배경으로 예측해도 Pixel Accuracy가 높게 나옵니다. 큰 TN이 점수를 부풀리기 때문에 TN을 빼는 IoU나 Dice를 주로 씁니다.

## Q. 정답 마스크를 조금씩 밀어 보는 실험에서 IoU와 Dice는 어떻게 변했나요?
id: seg-shift-experiment
type: 경험
level: 심화
tags: Mask IoU, Dice, 실험, 민감도
source: 3. Segmentation 모델 평가 › TP, FP, FN도 픽셀 단위

### 30초 답변
모델이 딴 강아지 마스크를 정답으로 두고, 테두리를 2픽셀쯤 깎은 뒤 오른쪽 아래로 밀어서 예측으로 썼습니다. 밀수록 놓친 픽셀과 잘못 칠한 픽셀이 동시에 커져서 교집합은 줄고 합집합은 늘어 두 점수가 같이 떨어졌습니다. 박스가 180×320px쯤 되는 강아지인데 30px만 밀려도 IoU가 0.5 아래로 떨어져서, mAP50 기준으로도 생각보다 빡빡하다는 걸 알았습니다.

### 자세히
| SHIFT | Mask IoU | Dice |
|---|---|---|
| 0 | 0.9377 | 0.9678 |
| 15 | 0.7032 | 0.8258 |
| 60 | 0.2217 | 0.3629 |

- SHIFT 0에서도 1이 아닌 건 erode로 테두리를 깎았기 때문이다. 경계 2픽셀만으로 IoU가 0.94가 됐다.
- 같은 픽셀 오차라도 물체가 작을수록 넓이 대비 비율이 커져서 더 크게 떨어질 것이다.

### 꼬리질문
#### 이 실험에서 Dice가 IoU보다 늘 높게 나온 이유는 무엇인가요?
Dice는 `2·IoU/(1+IoU)`라서 0과 1 사이에서는 항상 IoU보다 큽니다. 예측이 나빠져도 두 지표의 순서는 바뀌지 않고 눈금만 다릅니다.

## Q. Mask mAP는 어떻게 계산하나요?
id: seg-mask-map
type: 개념
level: 기본
tags: mAP, Mask mAP, mIoU, 평가지표
source: 3. Segmentation 모델 평가 › Mask mAP

### 30초 답변
Mask mAP는 Detection의 mAP와 계산 방식이 같고, 맞음과 틀림을 판정할 때 Box IoU 대신 Mask IoU를 쓴다는 것만 다릅니다. 예측 하나와 정답의 Mask IoU가 임계값을 넘으면 TP, 아니면 FP로 세고, 이렇게 모은 TP와 FP로 Precision-Recall 곡선을 그려 아래 넓이가 AP, 클래스 평균이 mAP입니다. mAP50은 IoU 0.5 기준, mAP50-95는 0.5부터 0.95까지 0.05 간격 임계값의 AP 평균입니다.

### 자세히
1. 판정: Mask IoU ≥ 임계값 → TP, 아니면 FP. 못 찾은 정답은 FN
2. 신뢰도 순으로 정렬해 Precision, Recall 누적
3. PR 곡선 아래 넓이 = AP, 클래스 평균 = mAP

```python
metrics = YOLO("yolo11n-seg.pt").val(data="coco8-seg.yaml")
metrics.box.map     # Box mAP50-95
metrics.seg.map     # Mask mAP50-95
metrics.seg.map50   # Mask mAP50
```

- YOLO seg 모델은 Box mAP와 Mask mAP를 둘 다 보고한다.

### 꼬리질문
#### Semantic Segmentation도 mAP로 평가하나요?
Semantic은 객체 단위 예측과 신뢰도가 없어서 보통 mIoU를 씁니다. 클래스마다 데이터셋 전체 픽셀로 IoU를 구하고 클래스 평균을 내는 방식입니다.

## Q. 같은 Seg 모델인데 Mask AP가 Box AP보다 낮게 나온 이유는 무엇인가요?
id: seg-mask-ap-vs-box-ap
type: 경험
level: 심화
tags: Mask mAP, Box mAP, IoU 임계값, 실험
source: 3. Segmentation 모델 평가 › Mask mAP

### 30초 답변
coco8-seg 검증셋에서 yolo11n-seg의 Mask mAP50은 0.852였지만 mAP50-95는 0.571이었고, IoU 임계값을 올릴수록 AP가 계단처럼 내려가다 0.95에서는 0이 됐습니다. 박스는 네 변만 맞으면 되지만 마스크는 경계 전체가 맞아야 해서, 모든 임계값에서 Mask AP가 Box AP보다 낮았습니다. 테두리 2픽셀만 깎아도 IoU가 0.94였던 실험을 생각하면 0.95를 넘기 어려운 게 당연합니다.

### 자세히
- 데이터: coco8-seg 검증 4장, 객체 17개 (연습용이라 숫자는 참고만)

| | mAP50 | mAP50-95 | AP@0.95 |
|---|---|---|---|
| Mask | 0.852 | 0.571 | 0 |

- mAP50-95는 엄격한 임계값까지 평균하므로 경계 정밀도 차이가 크게 드러난다.
- 정답 폴리곤 자체가 거친 근사라 아주 높은 IoU는 라벨 오차 때문에도 나오기 어렵다.

### 꼬리질문
#### 검증 데이터가 4장뿐이면 이 숫자를 얼마나 믿을 수 있나요?
객체 17개라 하나만 맞고 틀려도 AP가 크게 출렁입니다. 경향을 확인하는 용도로만 쓰고, 모델 비교는 COCO val처럼 큰 검증셋으로 해야 합니다.

## Q. FCN(Fully Convolutional Network)을 설명해 보세요.
id: seg-fcn
type: 개념
level: 기본
tags: FCN, Semantic Segmentation, 구조
source: 보충 (블로그 밖 내용)

### 30초 답변
FCN은 2015년에 나온 Semantic Segmentation 모델로, 분류 CNN의 FC 층을 1×1 합성곱으로 바꿔 네트워크 전체를 합성곱으로만 만든 구조입니다. 그래서 입력 크기에 제한이 없고 출력도 클래스 점수 맵 형태로 나옵니다. 줄어든 점수 맵을 원본 크기로 업샘플링하고, 얕은 층의 특징 맵을 더해 주는 skip 연결로 경계를 보완합니다.

### 자세히
```
분류 CNN: conv → pool ... → FC → 클래스 1개
FCN:      conv → pool ... → 1×1 conv → (C, H/32, W/32) → 업샘플 → (C, H, W)
```

| 변형 | 업샘플 배율 | 섞는 특징 |
|---|---|---|
| FCN-32s | 32배 한 번 | 마지막 층만 |
| FCN-16s | | + pool4 |
| FCN-8s | | + pool4, pool3 (가장 정밀) |

- 업샘플링은 bilinear로 초기화한 transposed conv를 쓰고, skip은 점수 맵을 더하는(sum) 방식이다.

### 꼬리질문
#### 해상도 손실을 다른 방식으로 줄인 모델은 무엇이 있나요?
DeepLab 계열은 dilated(atrous) convolution을 써서 풀링을 덜 하고도 receptive field를 넓혀 해상도를 덜 잃습니다. dilation rate가 다른 여러 atrous conv를 병렬로 붙인 ASPP로 여러 크기의 문맥도 함께 봅니다.

## Q. U-Net의 구조와 skip connection의 역할을 설명해 보세요.
id: seg-unet
type: 개념
level: 기본
tags: U-Net, skip connection, encoder-decoder, 의료영상
source: 보충 (블로그 밖 내용)

### 30초 답변
U-Net은 2015년 Ronneberger 등이 생물의학 영상 분할용으로 낸 encoder-decoder 구조입니다. encoder가 해상도를 줄이며 무엇이 있는지에 대한 특징을 뽑고, decoder가 해상도를 다시 키우며 픽셀 위치를 복원합니다. 같은 해상도의 encoder 특징 맵을 decoder에 이어 붙이는 skip connection이 다운샘플링 중 잃은 위치와 경계 정보를 되살려 줘서, 적은 데이터로도 경계가 정밀한 마스크를 만듭니다.

### 자세히
```
encoder: [conv×2 → maxpool] × 4    해상도 ↓, 채널 ↑
decoder: [up-conv → concat(encoder 특징) → conv×2] × 4    해상도 ↑
출력:    1×1 conv → 클래스 수 채널
```

| | FCN | U-Net |
|---|---|---|
| skip 방식 | 점수 맵 더하기 | 특징 맵 채널 방향 concat |
| decoder | 얕음 (업샘플 위주) | encoder와 대칭인 깊은 decoder |

- 원 논문은 데이터가 적은 의료 영상에서 강한 데이터 증강(탄성 변형 등)과 함께 학습했다.

### 꼬리질문
#### skip connection이 없으면 어떻게 되나요?
깊은 층 특징은 의미 정보는 풍부하지만 해상도가 낮아서, 업샘플만으로는 경계가 뭉개진 마스크가 나옵니다. 얕은 층의 고해상도 특징을 붙여 줘야 경계를 정확히 그릴 수 있습니다.

#### 여기서 skip connection은 ResNet의 skip과 같은 건가요?
ResNet은 같은 블록의 입력을 출력에 더해서 깊은 망의 기울기 흐름을 돕는 것이고, U-Net은 encoder의 공간 정보를 멀리 떨어진 decoder로 전달하려고 이어 붙이는 것이라 목적이 다릅니다.

## Q. Segmentation에서 해상도를 다시 키우는 업샘플링 방법들을 비교해 보세요.
id: seg-upsampling
type: 비교
level: 심화
tags: 업샘플링, 보간, Transposed Convolution, Unpooling
source: 보충 (블로그 밖 내용)

### 30초 답변
가장 단순한 방법은 nearest나 bilinear 보간으로, 학습할 파라미터가 없고 빠르지만 정해진 규칙대로만 키웁니다. Transposed convolution은 업샘플링 커널을 학습해서 어떻게 키울지를 데이터로 정하지만, 커널 크기가 stride로 나눠떨어지지 않으면 출력 칸마다 겹치는 횟수가 달라 바둑판 무늬(checkerboard artifact)가 생길 수 있습니다. 그래서 보간으로 키운 뒤 일반 합성곱을 붙이는 방식도 많이 씁니다.

### 자세히
| 방법 | 학습 파라미터 | 특징 |
|---|---|---|
| Nearest / Bilinear 보간 | 없음 | 빠름, 단순 |
| Transposed conv | 있음 | 학습 가능, checkerboard 가능성 |
| 보간 + conv | 있음 (conv) | checkerboard 완화, 많이 쓰는 대안 |
| Max unpooling | 없음 | 인코더 max pooling 위치를 기억해 그 자리에 값 복원 (SegNet) |

```python
nn.ConvTranspose2d(c, c, kernel_size=4, stride=2, padding=1)    # 2배
nn.Upsample(scale_factor=2, mode="bilinear"); nn.Conv2d(c, c, 3, padding=1)
```

### 꼬리질문
#### Transposed convolution은 convolution의 역연산인가요?
값을 되돌리는 역연산은 아니고, 합성곱의 입력과 출력 크기 관계를 뒤집은 연산입니다. 합성곱을 행렬 곱으로 썼을 때 그 행렬의 전치를 곱하는 것과 같아서 transposed라는 이름이 붙었습니다.

## Q. Mask R-CNN은 어떤 구조이고, YOLO-seg와는 어떻게 다른가요?
id: seg-mask-rcnn
type: 개념
level: 심화
tags: Mask R-CNN, RoIAlign, Instance Segmentation, two-stage
source: 보충 (블로그 밖 내용)

### 30초 답변
Mask R-CNN은 2017년 He 등이 낸 Instance Segmentation 모델로, two-stage 검출기인 Faster R-CNN에 마스크를 예측하는 작은 FCN 가지를 분류, 박스 회귀와 병렬로 붙인 구조입니다. 후보 영역마다 특징을 정확히 잘라 오도록 RoIPool 대신 RoIAlign을 썼고, 클래스마다 이진 마스크를 따로 예측해서 분류와 마스크 예측을 분리했습니다. YOLO-seg는 one-stage로 이미지 전체에 공용 prototype을 한 번 만들고 객체마다 계수만 예측한다는 점이 다릅니다.

### 자세히
```
이미지 → Backbone(+FPN) → RPN 후보 박스 → RoIAlign → ┬ 분류 + 박스 회귀
                                                    └ 마스크 FCN → 클래스별 28×28 마스크
```

- 마스크 손실은 정답 클래스 채널에만 픽셀별 sigmoid + BCE를 건다. 클래스끼리 경쟁하지 않는다.
- 28×28 마스크를 박스 크기로 키워 원본에 붙인다.

| | Mask R-CNN | YOLO-seg |
|---|---|---|
| 단계 | two-stage | one-stage |
| 마스크 | 후보 박스마다 따로 예측 | 공용 prototype × 객체별 계수 |
| 장점 | 정밀도 | 속도, 단순한 구조 |

### 꼬리질문
#### RoIAlign은 왜 필요한가요?
RoIPool은 후보 박스 좌표와 구역 경계를 정수로 반올림해서 특징이 몇 픽셀씩 어긋납니다. 분류에는 큰 문제가 없지만 픽셀 단위 마스크에는 치명적이라, RoIAlign은 반올림 없이 bilinear 보간으로 정확한 위치의 값을 샘플링합니다.

## Q. Segmentation 학습에는 어떤 손실 함수를 쓰고, 클래스 불균형은 어떻게 다루나요?
id: seg-loss
type: 개념
level: 심화
tags: 손실 함수, Cross Entropy, Dice loss, Focal loss, 클래스 불균형
source: 보충 (블로그 밖 내용)

### 30초 답변
기본은 픽셀마다 분류 손실을 구해 평균하는 픽셀 단위 Cross Entropy이고, 전경과 배경 두 가지면 BCE를 씁니다. 그런데 배경 픽셀이 압도적으로 많으면 평균 손실이 배경에 끌려가서 작은 전경을 소홀히 학습합니다. 그래서 클래스 가중치를 준 CE, 쉬운 픽셀의 비중을 줄이는 Focal loss, 겹침을 직접 최적화하는 Dice loss를 쓰고, 실제로는 CE와 Dice loss를 더해 쓰는 경우가 많습니다.

### 자세히
```
CE        = −(1/N) Σ_pixel Σ_c y_c · log p_c
Dice loss = 1 − (2·Σ p·g + ε) / (Σ p + Σ g + ε)     # p: 예측 확률(0~1), g: 정답
Focal     = −(1 − p_t)^γ · log p_t
```

- Dice loss는 0/1 대신 확률을 그대로 넣은 soft Dice라 미분할 수 있다.
- 분모가 전경 크기로 정규화되어 배경 비율의 영향을 덜 받는다.

### 꼬리질문
#### Dice loss만 쓰면 안 되나요?
전경이 아주 작거나 없는 이미지에서는 분모가 작아 기울기가 크게 출렁여 학습이 불안정할 수 있습니다. 그래서 픽셀마다 고르게 기울기를 주는 CE와 함께 쓰는 조합이 흔합니다.

## Q. 사람 마스크로 배경을 바꾸는 가상 배경은 어떻게 만들었나요?
id: seg-virtual-background
type: 경험
level: 기본
tags: 가상 배경, 마스크 합성, np.where, 브로드캐스팅
source: 4. 미니 프로젝트: 배경 바꾸기

### 30초 답변
`boxes.cls`로 person(0번)인 객체를 고르고, 같은 인덱스의 마스크를 원본 크기로 맞춘 뒤 0.5로 이진화해 사람 마스크를 만들었습니다. 그다음 `np.where`로 마스크가 참인 픽셀은 원본, 거짓인 픽셀은 새 배경에서 가져왔습니다. 배경 대신 `GaussianBlur`로 흐린 원본을 쓰면 배경 흐림 효과가 됩니다.

### 자세히
```python
background = cv2.resize(bg, (W, H))
output = np.where(person_mask[:, :, None] > 0, image, background)
blurred = cv2.GaussianBlur(image, (51, 51), 0)   # 커널은 홀수, 클수록 흐림
```

- `person_mask`는 `(H, W)`, 이미지는 `(H, W, 3)`이라 `[:, :, None]`으로 `(H, W, 1)`을 만들면 브로드캐스팅으로 B, G, R에 같은 조건이 적용된다.
- 사람 두 명은 같은 마스크에 함께 칠했다. 넥타이는 tie(27번)로 따로 검출됐지만 사람 마스크 안에 이미 들어 있어 문제없었다.
- A와 B를 바꾸면 사람 모양으로 구멍을 뚫은 것처럼 된다.

### 꼬리질문
#### 경계에 원래 배경이 한 줄씩 남는 문제는 어떻게 줄이나요?
손가락 끝처럼 가는 부분은 마스크가 덜 덮어서 잘리고 경계에 원래 배경이 남았습니다. 이진 마스크 대신 마스크를 흐려서 0~1 알파 값으로 만들고 `out = α·image + (1−α)·background`로 섞으면 경계가 부드러워지고, 필요하면 erode/dilate로 경계 폭을 조절합니다.

#### 웹캠으로 실시간 처리할 때 주의할 점은 무엇이었나요?
모델은 반복문 밖에서 한 번만 불러와야 하고, 안에 넣으면 프레임마다 가중치를 다시 읽어 매우 느려집니다. 프레임마다 `verbose=False`로 로그를 끄고, 사람이 없는 프레임을 위해 `masks is not None` 검사를 넣었습니다.

## Q. Pose Estimation이란 무엇이고, Keypoint와 Skeleton은 무엇인가요?
id: pose-keypoint-skeleton
type: 개념
level: 기본
tags: Pose Estimation, Keypoint, Skeleton
source: 5. Pose Estimation › Keypoint와 Skeleton

### 30초 답변
Pose Estimation은 사람의 관절 위치를 점으로 찍어 자세를 추정하는 작업입니다. Keypoint는 관절 하나의 위치로, `(x, y)` 좌표에 신뢰도가 붙어 관절마다 값 3개를 출력합니다. Skeleton은 어깨 → 팔꿈치 → 손목처럼 관절을 어떤 순서로 이을지 미리 정한 연결표라서 모델이 예측하는 값이 아닙니다.

### 자세히
- 모델 출력: 사람마다 관절 17개 × `(x, y, conf)`
- `result.plot()`이 정해진 연결표대로 선을 그어 준다.
- 활용: 운동 자세 교정, 동작 인식, 낙상 감지, 제스처 입력 (손 들기 카운터)

### 꼬리질문
#### Skeleton이 예측값이 아니라면 왜 중요한가요?
시각화뿐 아니라 관절 사이 각도나 길이 같은 특징을 계산할 때 어느 관절끼리 이어지는지 알아야 하기 때문입니다. bottom-up 방식에서는 이 연결 관계를 이용해 관절을 사람 단위로 묶기도 합니다.

## Q. COCO 17 Keypoint는 어떻게 구성되어 있나요?
id: pose-coco17
type: 개념
level: 기본
tags: COCO, Keypoint, 좌우 규칙
source: 5. Pose Estimation › COCO 17 Keypoint

### 30초 답변
COCO 형식은 관절 17개를 0~16번으로 정의합니다. 0~4번은 얼굴(코, 눈, 귀), 5~10번은 상체(어깨, 팔꿈치, 손목), 11~16번은 하체(골반, 무릎, 발목)이고, 코를 빼면 홀수가 왼쪽, 짝수가 오른쪽입니다. Left와 Right는 화면 기준이 아니라 사진 속 사람 본인 기준이라, 카메라를 보고 선 사람의 왼어깨는 화면 오른쪽에 찍힙니다.

### 자세히
| 번호 | 관절 |
|---|---|
| 0 | 코 |
| 1, 2 / 3, 4 | 왼눈, 오른눈 / 왼귀, 오른귀 |
| 5, 6 | 왼어깨, 오른어깨 |
| 7, 8 | 왼팔꿈치, 오른팔꿈치 |
| 9, 10 | 왼손목, 오른손목 |
| 11, 12 | 왼골반, 오른골반 |
| 13, 14 / 15, 16 | 왼무릎, 오른무릎 / 왼발목, 오른발목 |

- 손 들기 판정에는 5·6(어깨)과 9·10(손목)만 쓴다.

### 꼬리질문
#### 좌우 기준을 헷갈리면 어떤 문제가 생기나요?
청기백기처럼 손을 구분하는 기능에서 왼손을 들었는데 오른손 카운트가 오르는 식으로 뒤바뀝니다. 화면에 보이는 위치가 아니라 관절 번호로 판단해야 합니다.

## Q. 여러 사람의 자세를 찾는 Top-down과 Bottom-up 방식을 비교해 보세요.
id: pose-topdown-bottomup
type: 비교
level: 기본
tags: Top-down, Bottom-up, One-stage, YOLO-pose
source: 5. Pose Estimation › 여러 사람의 자세를 찾는 방법

### 30초 답변
Top-down은 사람 박스를 먼저 찾고 박스마다 잘라서 관절을 찾는 방식이라 정확하지만, 사람 수만큼 반복해서 사람이 많을수록 느려집니다. Bottom-up은 이미지 전체에서 관절을 먼저 다 찾고 사람 단위로 묶어서 사람 수와 상관없이 속도가 일정하지만, 이 손목이 누구 것인지 묶는 과정이 까다롭습니다. YOLO-pose는 격자점마다 박스와 관절 17개를 한 번에 예측하는 one-stage 방식이라, 박스가 관절을 사람 단위로 묶어 주는 역할까지 합니다.

### 자세히
| | Top-down | Bottom-up | One-stage (YOLO-pose) |
|---|---|---|---|
| 순서 | 사람 검출 → 박스별 관절 | 관절 전부 → 사람별 묶기 | 박스 + 관절 동시 |
| 속도 | 사람 수에 비례 | 일정 | 일정, 추론 1번 |
| 어려운 점 | 검출기 의존 | 관절 묶기(grouping) | 작은 사람 정밀도 |
| 예 | HRNet 등 | OpenPose | YOLO-pose |

### 꼬리질문
#### 사람이 많이 겹친 장면에서는 각각 어떤 약점이 있나요?
Top-down은 검출기가 겹친 사람을 놓치면 그 사람의 자세도 못 찾고, 박스 안에 다른 사람 관절이 섞여 헷갈릴 수 있습니다. Bottom-up은 가까이 붙은 사람들의 관절을 잘못 묶을 위험이 커집니다.

#### OpenPose는 관절을 어떻게 사람별로 묶나요?
관절 위치 heatmap과 함께, 두 관절을 잇는 팔다리 방향을 벡터장으로 나타낸 Part Affinity Field를 예측합니다. 후보 관절 쌍 사이의 벡터장이 연결 방향과 잘 맞을수록 같은 사람으로 묶습니다.

## Q. Keypoint를 예측하는 heatmap 방식과 regression 방식을 비교해 보세요.
id: pose-heatmap-regression
type: 비교
level: 심화
tags: heatmap, regression, Keypoint, YOLO-pose
source: 보충 (블로그 밖 내용)

### 30초 답변
Heatmap 방식은 관절마다 한 장의 확률 지도를 출력하고, 정답 위치에 가우시안을 그린 heatmap을 학습한 뒤 가장 높은 점을 관절 위치로 씁니다. 공간 구조를 그대로 유지해서 정확도가 높은 편이지만 출력이 크고, 저해상도 heatmap에서 좌표를 뽑으면 양자화 오차가 생깁니다. Regression 방식은 좌표 숫자를 직접 출력해서 단순하고 빠르며 끝까지 한 번에 학습하기 쉽지만, 전통적으로는 heatmap보다 정확도가 낮았습니다.

### 자세히
| | Heatmap | Regression |
|---|---|---|
| 출력 | 관절 K개 × H'×W' 지도 | 관절 K개 × (x, y) |
| 좌표 추출 | argmax (+ 보정) | 출력 그대로 |
| 장점 | 정확, 공간 정보 유지 | 가볍고 빠름, 후처리 단순 |
| 단점 | 메모리, 양자화 오차 | 정밀도, 학습이 어려운 편 |

- YOLO-pose는 격자점마다 관절 좌표를 직접 내는 regression 쪽이다.

### 꼬리질문
#### Heatmap의 양자화 오차는 어떻게 줄이나요?
argmax 위치를 두 번째로 높은 이웃 쪽으로 조금 옮기거나, heatmap을 확률 분포로 보고 좌표의 기댓값을 구하는 soft-argmax(integral regression)를 씁니다. 기댓값 방식은 미분이 되어 좌표 자체로 손실을 걸 수도 있습니다.

## Q. YOLO Pose 결과에서 키포인트는 어떤 형식으로 나오나요?
id: pose-keypoint-output
type: 개념
level: 기본
tags: Keypoints, ultralytics, 좌표계
source: 5. Pose Estimation › 키포인트 꺼내기

### 30초 답변
`result.keypoints`에 사람 수 N × 관절 17개의 좌표와 신뢰도가 들어 있습니다. `.xy`는 `(N, 17, 2)` 픽셀 좌표, `.xyn`은 너비와 높이로 나눈 0~1 좌표, `.data`는 신뢰도까지 붙은 `(N, 17, 3)`입니다. 이미지 좌표는 왼쪽 위가 `(0, 0)`이고 아래로 갈수록 y가 커져서, y가 작을수록 화면 위쪽이라는 점을 주의해야 합니다.

### 자세히
```python
kp = model("person.jpg")[0].keypoints
kp.xy      # (N, 17, 2)  픽셀 좌표
kp.xyn     # (N, 17, 2)  x/W, y/H → (W, H)를 곱하면 xy
kp.conf    # (N, 17)     관절별 신뢰도
kp.data    # (N, 17, 3)  [x, y, conf]
kp.xy[0]   # 첫 번째 사람의 관절 17개, (17, 2)
```

- Pose 결과에도 `boxes`가 같이 나와서 사람 박스와 신뢰도를 쓸 수 있다.
- 사람이 없는 프레임에서 꺼내면 에러가 나므로 `len(result.keypoints.xy) > 0`일 때만 처리한다.

### 꼬리질문
#### 정규화 좌표 `xyn`은 언제 쓰면 좋나요?
해상도가 다른 영상끼리 비교하거나 해상도에 무관한 규칙을 만들 때 씁니다. 예를 들어 손 들기 여유값을 픽셀 대신 이미지 높이 비율로 정하면 카메라 해상도가 바뀌어도 기준이 유지됩니다.

## Q. 신뢰도가 낮은 관절은 좌표가 (0, 0)으로 나온다고 배웠는데, 직접 확인해 보니 어땠나요?
id: pose-low-conf-keypoint
type: 경험
level: 심화
tags: 신뢰도, 키포인트 필터링, 실험
source: 5. Pose Estimation › 신뢰도가 낮은 관절과 (0, 0)

### 30초 답변
강의자료에는 신뢰도 0.5 미만 관절이 (0, 0)으로 나오니 `x > 0 and y > 0`으로 검사하라고 되어 있었는데, 버스 사진에서 직접 세어 보니 신뢰도 0.5 미만 관절 28개 중 정확히 (0, 0)인 것은 0개였습니다. 제가 쓴 ultralytics 8.4.215는 신뢰도가 낮아도 모델이 찍은 그럴듯한 좌표를 그대로 돌려줬습니다. 그래서 좌표 검사 대신 `keypoints.conf`를 직접 보고 기준을 넘는 관절만 쓰는 게 안전하다고 결론 내렸습니다.

### 자세히
- 가운데 두 사람: 거의 모든 관절이 0.9 이상
- 화면 끝에 몸이 잘린 사람 1(평균 0.38), 사람 3(평균 0.20). 사람 1은 얼굴 쪽이 0.05~0.23
- x가 0인 관절이 몇 개 있었지만 사진 왼쪽 가장자리에 붙어서 0이 된 경우였다.

```python
conf = result.keypoints.conf[0]
valid = conf > 0.5          # 좌표가 아니라 신뢰도로 거른다
```

- 손목이 화면 밖인데 신뢰도 0.1짜리 좌표가 어깨 위에 찍히면 손을 든 것으로 잘못 셀 수 있다.

### 꼬리질문
#### 모델의 신뢰도와 정답 라벨의 visibility는 같은 건가요?
다릅니다. conf는 모델이 이 관절 위치를 얼마나 확신하는지에 대한 예측값이고, visibility `v`는 라벨러가 정답 파일에 적은 라벨 없음, 가려짐, 보임 상태입니다. 평가에는 v를, 실제 사용 시 필터링에는 conf를 씁니다.

## Q. Pose 평가는 왜 IoU 대신 거리로 하고, 왜 사람 크기로 정규화하나요?
id: pose-distance-normalization
type: 이유
level: 기본
tags: Pose 평가, 정규화, visibility, Ground Truth
source: 6. Pose 모델 평가 › 점은 겹칠 수가 없다 / Pose 정답 형식

### 30초 답변
Keypoint는 영역이 아니라 점이라 겹침을 잴 수 없어서 IoU를 계산할 수 없고, 대신 정답과의 거리로 평가합니다. 그런데 픽셀 거리를 그대로 쓰면 키가 500px인 사람의 10px 오차와 50px인 사람의 10px 오차를 똑같이 취급하게 됩니다. 그래서 사람 크기로 나눠 정규화한 뒤 비교하고, 이 아이디어가 PCK와 OKS의 공통 출발점입니다.

### 자세히
| | Segmentation | Pose |
|---|---|---|
| 비교 대상 | 두 영역 | 두 점 |
| 기준 | 겹침 | 거리 |
| 지표 | IoU, Dice | PCK, OKS |

- COCO 정답: 사람마다 관절 17개 × `(x, y, v)` = 51개 숫자, `reshape(-1, 3)`으로 `(17, 3)`
- YOLO 형식(`coco8-pose`)은 앞에 클래스와 박스 4개가 붙어 한 줄에 5 + 51 = 56개
- `v`: 0 라벨 없음, 1 가려졌지만 라벨 있음, 2 보임. `v = 0`인 관절은 평가에서 뺀다.

### 꼬리질문
#### 가려진 관절(v = 1)도 평가에 들어가나요?
COCO의 OKS는 `v > 0`인 관절을 모두 세므로 가려진 관절도 들어갑니다. 라벨러가 위치를 추정해 찍어 두었기 때문에, 모델도 보이지 않는 관절의 위치를 추론해 내야 점수를 받습니다.

## Q. PCK란 무엇이고, 어떤 한계가 있나요?
id: pose-pck
type: 개념
level: 기본
tags: PCK, PCKh, 평가지표
source: 6. Pose 모델 평가 › PCK

### 30초 답변
PCK(Percentage of Correct Keypoints)는 예측한 관절이 정답 위치에서 허용 거리 안에 들어온 비율입니다. 허용 거리는 α × 기준 크기라서, PCK@0.2면 사람 박스 높이의 20% 안이면 맞힌 것으로 칩니다. 다만 허용 범위 안이면 0.1px이든 20px이든 똑같이 맞음이라, 얼마나 정밀한지는 구분하지 못합니다.

### 자세히
```
맞음 조건: ‖예측 − 정답‖ ≤ α × H     (H = 사람 박스 높이)
PCK = 맞힌 관절 수 / 라벨 있는 관절 수 (v = 0 제외)
```

```python
distance = np.linalg.norm(gt[i, :2] - pred[i])
if distance <= bbox_h * ratio:
    correct += 1
```

- α가 작을수록 엄격해서 PCK가 같거나 낮아진다.
- 기준 크기로 머리 크기를 쓰는 변형이 PCKh다.
- 관절마다 같은 허용 거리를 써서, 라벨 자체가 흔들리기 쉬운 관절과 아닌 관절을 구분하지 않는다.

### 꼬리질문
#### 기준 크기를 박스 높이 대신 머리 크기로 쓰는 이유는 무엇인가요?
박스 높이는 앉거나 웅크린 자세, 몸이 잘린 사진에서 크게 달라지지만 머리 크기는 자세에 덜 흔들립니다. 그래서 MPII 같은 데이터셋에서는 머리 크기 기준의 PCKh@0.5를 많이 씁니다.

## Q. PCK를 직접 계산해 보니 작은 사람에서 어떤 일이 있었나요?
id: pose-pck-small-person
type: 경험
level: 심화
tags: PCK, 작은 객체, 실험
source: 6. Pose 모델 평가 › PCK

### 30초 답변
coco8-pose의 한 사람짜리 사진 세 장으로 α를 바꿔 가며 PCK를 계산했는데, 박스 높이 53px인 사람만 PCK@0.05에서 0%가 나왔습니다. 허용 반경이 465px짜리 사람은 23px인데 53px짜리 사람은 2.7px라서, 예측이 2.8, 4.9, 5.0px만 벗어나도 전부 탈락했습니다. 사람 크기로 정규화해도 작은 사람은 정답 라벨 자체가 몇 픽셀씩 흔들려서 엄격한 α에서는 운에 가까워진다는 걸 알았습니다.

### 자세히
| 이미지 | 박스 높이 | PCK@0.05 | PCK@0.1 | PCK@0.2 |
|---|---|---|---|---|
| 000000000036 | 465px | 84.6% | 100.0% | 100.0% |
| 000000000086 | 376px | 93.8% | 100.0% | 100.0% |
| 000000000151 | 53px | 0.0% | 100.0% | 100.0% |

- 마지막 사진은 라벨 있는 관절이 3개뿐이라 하나 틀릴 때마다 33%씩 떨어진다.
- 관절 수가 적은 샘플의 PCK는 분산이 커서, 여러 사람을 모아 관절 단위로 합산해야 안정적이다.

### 꼬리질문
#### 그러면 작은 사람에게 α를 더 크게 줘야 하나요?
사람마다 기준을 바꾸면 지표끼리 비교가 어려워져서 보통 그렇게 하지 않습니다. 대신 COCO처럼 키포인트를 라벨링하기 어려운 아주 작은 사람에게는 키포인트 정답을 달지 않아 사실상 평가에서 빠지게 하거나, 연속 점수인 OKS로 같은 오차라도 점수가 완만하게 깎이게 합니다.

## Q. OKS(Object Keypoint Similarity)를 설명해 보세요.
id: pose-oks
type: 개념
level: 심화
tags: OKS, COCO, 평가지표
source: 6. Pose 모델 평가 › OKS

### 30초 답변
OKS는 관절 위치의 정확도를 0~1의 연속 점수로 나타낸 지표입니다. 관절 하나의 점수는 `exp(−d²/(2s²k²))`로, 거리 d를 사람 크기 s와 관절별 상수 k로 정규화하고, 라벨 있는 관절의 점수를 평균하면 그 사람의 OKS가 됩니다. PCK와 크게 다른 점은 관절별 상수 k가 있다는 것으로, 라벨이 잘 흔들리지 않는 눈은 엄격하게, 옷에 가려 위치가 흔들리는 골반은 느슨하게 평가합니다.

### 자세히
```
OKS = Σ_i exp(−d_i² / (2·s²·k_i²)) · δ(v_i > 0)  /  Σ_i δ(v_i > 0)
d_i: 정답과 예측 거리,  s²: 사람 넓이,  k_i = 2σ_i (관절별 상수)
```

- COCO σ는 여러 사람이 같은 이미지에 라벨을 찍었을 때 관절별로 얼마나 흩어지는지에서 정했다. 눈 0.025, 골반 0.107 정도다.
- 150×400px 사람에서 10px 틀렸을 때 눈은 0.533점, 골반은 0.966점
- 점수가 절반이 되는 거리: 눈 10px, 골반 45px
- 거리가 0이면 1, 멀어질수록 가우시안 모양으로 0에 가까워진다.

### 꼬리질문
#### 사람 크기 s를 넓이로 쓰는 이유는 무엇인가요?
s²가 넓이라서 s는 길이 단위가 되고, 거리 d와 단위가 맞습니다. 그래서 사람이 두 배 크게 찍히면 허용 거리도 두 배가 되어 크기와 무관한 점수가 나옵니다.

#### OKS는 Detection의 무엇에 해당하나요?
IoU에 해당합니다. 예측한 사람과 정답 사람의 OKS가 임계값을 넘으면 TP로 치고, OKS 0.5~0.95 임계값별 AP를 평균해서 pose mAP를 구합니다.

## Q. 같은 Pose 모델에서 박스 기준 mAP와 포즈 기준 mAP가 어떻게 달랐나요?
id: pose-map-vs-box-map
type: 경험
level: 심화
tags: pose mAP, Box mAP, OKS, 실험
source: 6. Pose 모델 평가 › OKS

### 30초 답변
yolo11n-pose를 coco8-pose 검증셋으로 평가했을 때 박스 기준은 mAP50 0.917, mAP50-95 0.723이었지만 OKS 기준 포즈는 0.766, 0.349로 훨씬 낮았습니다. 박스는 사람을 감싸는 네모 하나만 맞으면 되지만, 포즈는 관절 17개가 각각 정답 가까이 있어야 OKS가 올라가기 때문입니다. 엄격한 기준까지 평균하는 mAP50-95에서 차이가 두 배 넘게 벌어졌습니다.

### 자세히
| 기준 | mAP50 | mAP50-95 |
|---|---|---|
| Box (IoU) | 0.917 | 0.723 |
| Pose (OKS) | 0.766 | 0.349 |

```python
metrics = YOLO("yolo11n-pose.pt").val(data="coco8-pose.yaml")
metrics.box.map, metrics.pose.map, metrics.pose.map50
```

- 판정 기준만 Box IoU → Mask IoU → OKS로 바뀌고, Precision, Recall, mAP를 쌓는 과정은 같다.

### 꼬리질문
#### PCK와 pose mAP는 무엇이 다른가요?
PCK는 정답 사람 한 명을 놓고 관절별로 허용 거리 안인지 세는 비율이라, 사람을 찾았다는 전제에서 관절 정확도만 봅니다. pose mAP는 OKS로 검출된 사람마다 TP, FP를 판정하고 신뢰도 순서까지 반영한 PR 곡선으로 계산해서, 사람을 못 찾거나 없는 사람을 찾은 경우까지 점수에 들어갑니다.

## Q. 손을 들었는지는 어떻게 판정했고, 왜 여유값을 뒀나요?
id: pose-hand-up-margin
type: 이유
level: 기본
tags: 손 들기 카운터, 여유값, 키포인트 노이즈
source: 7. 미니 프로젝트: 손 들기 카운터와 청기백기 › 손을 들었는지 판단하기

### 30초 답변
어깨(5·6번)와 손목(9·10번)의 y 좌표를 비교해서, y가 작을수록 위쪽이니 손목 y가 어깨 y보다 작으면 손을 든 것으로 봤습니다. 다만 손목이 어깨 높이 근처에 있으면 키포인트 좌표가 프레임마다 몇 픽셀씩 흔들려 판정이 True와 False를 오갑니다. 그래서 어깨보다 30px 넘게 올라가야만 든 것으로 치는 여유값을 둬서 이 흔들림을 무시했습니다.

### 자세히
```python
def is_hand_up(wrist_y, shoulder_y, margin=30):
    return wrist_y < shoulder_y - margin
```

- 자동 채점 케이스 "어깨보다 10px 위"는 `190 < 200 − 30`이 거짓이라 `False`여야 한다.
- 실습 자료의 트러블슈팅 팁: 손을 내려도 가끔 카운트가 오르면 여유값을 30에서 50으로 늘린다.
- 손을 들어도 안 오르면 어깨와 손목이 모두 화면에 보이도록 카메라에서 멀리 앉는다.

### 꼬리질문
#### 여유값을 픽셀로 고정하면 어떤 문제가 있나요?
카메라에서 멀리 있는 사람은 작게 찍혀서 30px이 몸에 비해 큰 거리가 되고, 가까운 사람은 작은 거리가 됩니다. 어깨너비나 몸통 길이에 비례하게 여유값을 정하면 거리에 덜 민감해집니다.

## Q. 손을 한 번 들었을 때 한 번만 세려면 어떻게 해야 하나요? 시뮬레이션 결과는 어땠나요?
id: pose-state-machine
type: 경험
level: 심화
tags: 상태 머신, 손 들기 카운터, 시뮬레이션
source: 7. 미니 프로젝트: 손 들기 카운터와 청기백기 › 한 번 들면 한 번만 세기

### 30초 답변
웹캠은 초당 수십 프레임이라 손을 들고 있는 동안 매 프레임 `True`가 나와서, 프레임마다 더하면 숫자가 폭주합니다. 그래서 `stage` 변수로 상태를 기억하고 `"down"`에서 `"up"`으로 바뀌는 순간에만 1을 더하는 상태 머신을 썼습니다. 실제로 두 번 든 150프레임 시뮬레이션에서 상태 머신이 없으면 47, 상태 머신만 쓰고 여유값을 0으로 두면 9, 둘 다 쓰면 정확히 2가 나왔습니다.

### 자세히
```python
if hand_up and stage == "down":
    count += 1
    stage = "up"
elif not hand_up:
    stage = "down"
```

| 설정 | 카운트 (정답 2) |
|---|---|
| 상태 머신 없음 | 47 |
| 상태 머신 + 여유값 0 | 9 |
| 상태 머신 + 여유값 30px | 2 |

- 손목 y에 표준편차 5px 흔들림을 주고, 어깨 높이에서 한동안 머무는 구간을 넣었다.
- 상태 머신은 들고 있는 동안 계속 세는 문제를, 여유값은 경계에서 흔들리며 여러 번 세는 문제를 막는다.

### 꼬리질문
#### 여유값 말고 흔들림을 더 확실히 막는 방법이 있나요?
올릴 때와 내릴 때 기준을 다르게 두는 히스테리시스(예: 어깨보다 30px 위로 가면 up, 어깨 아래로 내려와야 down)를 쓰거나, 같은 판정이 몇 프레임 연속될 때만 상태를 바꾸는 방법이 있습니다. 키포인트 좌표에 이동 평균을 씌워 흔들림 자체를 줄일 수도 있습니다.

## Q. 왼손과 오른손을 따로 세는 청기백기 게임은 어떻게 만들었나요?
id: pose-flag-game
type: 경험
level: 기본
tags: 청기백기, 상태 머신, 좌우 구분
source: 7. 미니 프로젝트: 손 들기 카운터와 청기백기 › 청기백기

### 30초 답변
왼손(5·9번)과 오른손(6·10번)을 각각 독립된 상태 머신으로 만들어 `left_count`, `left_stage`와 `right_count`, `right_stage`로 로직을 두 벌 돌렸습니다. 상태를 따로 기억하니 두 손을 함께 든 프레임에서도 서로의 카운트에 영향을 주지 않았고, 실습 프레임 기록에서 왼손 2번, 오른손 3번이 정확히 나왔습니다. 주의할 점은 COCO의 Left가 찍힌 사람 본인의 왼쪽이라, 웹캠 화면에서는 내 왼손이 화면 오른쪽에 보인다는 것입니다.

### 자세히
```python
for side, (s, w) in {"left": (5, 9), "right": (6, 10)}.items():
    up = is_hand_up(kp[w][1], kp[s][1])
    # side별 count, stage로 같은 상태 머신 실행
```

- 화면 표시: 왼손 카운트 초록 `(0, 255, 0)`, 오른손 노랑 `(0, 255, 255)`. OpenCV는 BGR 순서다.
- 더 해 볼 거리: "청기 올려!" 같은 명령을 랜덤으로 띄우고 맞게 들었는지 판정하기

### 꼬리질문
#### 셀카처럼 화면을 좌우로 뒤집어서 보여 주고 싶다면 어떻게 해야 하나요?
뒤집은 프레임을 그대로 모델에 넣으면 사람 모습도 거울상이 되어 Left와 Right가 바뀌어 나올 가능성이 큽니다. 추론은 원본 프레임으로 하고 화면에 그릴 때만 뒤집거나, 뒤집은 프레임으로 추론했다면 좌우 관절 번호를 바꿔서 해석해야 합니다.
