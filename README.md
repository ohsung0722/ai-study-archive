# 딥러닝 면접 노트

딥러닝·컴퓨터 비전 공부 내용을 면접 질문과 개념 노트로 정리한 웹페이지입니다.

## 구성

| 탭 | 내용 |
|---|---|
| 개념 노트 | 13개 주제. 개념 지도(단계마다 핵심과 막힌 점), 한 장 요약, 이해 확인 문제(개념·만약에·생각해보기·연결·OX·계산) |
| 면접 질문 | 13개 주제 445문항. 질문마다 30초 답변, 자세히, 꼬리질문. 질문 유형(개념·비교·이유·경험)과 난이도(기본·심화)로 거를 수 있음 |

주제: 딥러닝 기초, 학습 기법, CNN, RNN·LSTM, 자연어처리, OpenCV 영상처리, 데이터 증강·ResNet, Object Detection, Segmentation·Pose, Transformer 비전, LLM, Multimodal, Tools & MCP

## 파일

```
index.html            화면 (마크다운을 읽어서 보여 줌)
questions/
  manifest.json       불러올 파일 목록과 순서
  _template.md        면접 질문 작성 형식
  *.md                주제별 면접 질문
notes/
  *.md                주제별 개념 노트 (형식은 notes/cnn.md 맨 위 주석)
```

## 내용 추가

1. 면접 질문: `questions/_template.md`를 복사해 새 파일을 만들고 `manifest.json`의 `files`에 추가
2. 개념 노트: `notes/`에 새 파일을 만들고 `manifest.json`의 `notes`에 추가

## 로컬에서 보기

브라우저가 파일을 직접 열면 마크다운을 불러오지 못하므로 간단한 서버로 엽니다.

```
python -m http.server 8000
```

그다음 `http://localhost:8000`에 접속합니다.
