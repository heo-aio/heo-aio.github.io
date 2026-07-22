+++
title = "[ML 복습] 의사결정나무와 앙상블 (Voting·Bagging·Boosting)"
date = 2026-07-22T14:00:00+09:00
draft = false
tags = ["Python", "Machine Learning", "Decision Tree", "Ensemble", "RandomForest", "복습"]
categories = ["ml"]
+++

세 번째 복습 편은 **의사결정나무(Decision Tree)** 와, 이 단일 모델의 한계를 넘는 **앙상블(Ensemble)** 기법이다. 요즘 캐글에서 자주 쓰는 부스팅 계열의 뿌리가 여기 있다.

---

## 🌳 의사결정나무

의사결정나무는 스무고개처럼 데이터에 계속 질문을 던지며 가지를 쳐 나가는 알고리즘이다. 장점이 뚜렷하다.

- **직관적이고 해석이 쉽다.** 예측 과정을 그림으로 설명할 수 있다.
- **비선형 관계도 표현한다.** 직선이 아니라 계단식 경계를 만든다.
- **회귀와 분류 모두** 같은 나무 구조로 푼다.
- **전처리에 덜 민감하다.** 스케일링 영향이 적다.

단, 나무가 제한 없이 깊어지면 학습 데이터의 노이즈까지 외워 **과적합**이 되기 쉽다. 그래서 `max_depth` 같은 걸로 깊이를 제한한다.

### 회귀 나무와 분류 나무

둘 다 "데이터를 어떤 구역으로 나눌까"를 찾지만 기준이 다르다.

![회귀 나무와 분류 나무](/images/ml/review_트리_회귀분류.png)

**회귀 나무**(왼쪽)는 구역별 평균값으로 예측해서 그림처럼 **계단 모양**이 된다. 나눌 때는 구역 안 오차 제곱합(RSS)이 가장 작아지는 지점을 고른다. 깊이를 5로 주면 2일 때보다 더 잘게 쪼개 곡선을 촘촘히 따라간다.

**분류 나무**(오른쪽)는 예측값이 평균이 아니라서 RSS를 못 쓴다. 대신 구역 안에 서로 다른 클래스가 얼마나 섞였는지(**불순도**)를 최소화한다. 불순도가 낮다 = 노드가 순수하다(한 클래스로 깔끔). 대표 지표가 **지니(Gini)** 와 **엔트로피(Entropy)** 다.

```python
from sklearn.tree import DecisionTreeRegressor, DecisionTreeClassifier

# 회귀 (과적합 방지 위해 max_depth 제한)
reg = DecisionTreeRegressor(max_depth=5, random_state=100)

# 분류 (criterion: 'gini' 기본값, 'entropy' 등)
clf = DecisionTreeClassifier(criterion='gini', max_depth=3, random_state=100)
```

---

## 🤝 앙상블 — 여러 모델을 모으기

단일 나무는 과적합 위험이 크다. 그래서 **여러 모델을 조합해** 성능과 안정성을 끌어올리는 게 앙상블이다. 방식이 크게 세 가지다.

![앙상블 세 가지 방식](/images/ml/review_앙상블_구조.png)

### Voting (보팅)

서로 **다른 종류**의 모델들(예: 로지스틱 + KNN + SVM)의 예측을 모아 투표로 결정한다.

- **하드 보팅**: 각 모델이 낸 예측 클래스를 다수결로.
- **소프트 보팅**: 각 모델의 클래스별 확률을 평균내 가장 높은 걸로. 보통 성능이 더 좋다.

```python
from sklearn.ensemble import VotingClassifier
from sklearn.linear_model import LogisticRegression
from sklearn.neighbors import KNeighborsClassifier

vo_clf = VotingClassifier(
    estimators=[('LR', LogisticRegression(max_iter=1000)),
                ('KNN', KNeighborsClassifier(n_neighbors=5))],
    voting='soft')
```

### Bagging (배깅)

**B**ootstrap **Agg**regat**ing**. 원본에서 **복원 추출**로 조금씩 다른 데이터셋 여러 개를 만들고, **같은 모델**을 각각 학습시켜 결과를 합친다. 각 모델이 다른 샘플을 보니 다양성이 생겨 과적합을 줄인다.

```python
from sklearn.ensemble import BaggingClassifier
from sklearn.tree import DecisionTreeClassifier

ba_clf = BaggingClassifier(estimator=DecisionTreeClassifier(),
                           n_estimators=100, random_state=100)
```

### Boosting (부스팅)

약한 모델을 **순차적으로** 이어 붙여 강한 모델을 만든다. 핵심은 앞 모델이 **틀린 데이터에 가중치를 줘서**, 다음 모델이 그걸 집중적으로 학습하게 하는 것이다. AdaBoost, Gradient Boosting, XGBoost, LightGBM, CatBoost가 다 이 계열이다.

> **복습 포인트**
>
> 배깅은 모델들을 **독립적·병렬**로 키워 합치고, 부스팅은 앞 결과를 받아 **순차적**으로 개선한다. 그래서 부스팅은 병렬 처리가 안 돼 학습이 더 오래 걸린다. 이 병렬 vs 순차 차이가 둘을 가르는 핵심이다.

---

## 🌲 대표 앙상블 모델들

**랜덤 포레스트 = 의사결정나무 + 배깅**이다. 복원 추출로 만든 여러 데이터셋에 나무를 각각 키워 숲을 만들고, 회귀는 평균 / 분류는 다수결로 합친다. 단일 나무의 과적합을 크게 완화한다.

부스팅 계열은 이렇게 발전해왔다.

- **AdaBoost**: 오분류된 데이터에 가중치를 주며 개선. 순차라 병렬 불가.
- **Gradient Boosting(GBM)**: 가중치 갱신에 경사하강법을 사용. 잔차(오차)를 줄여나간다. 성능은 뛰어나지만 연산량이 많다.
  - **XGBoost**: GBM에 속도와 규제를 강화.
  - **LightGBM**: 리프 중심 분할로 빠르고 가볍다.
  - **CatBoost**: 범주형 변수 처리에 강하다.

```python
from sklearn.ensemble import (RandomForestClassifier,
                              AdaBoostClassifier, GradientBoostingClassifier)

rf  = RandomForestClassifier(n_estimators=100, random_state=100)
ada = AdaBoostClassifier(n_estimators=100, random_state=100)
gb  = GradientBoostingClassifier(n_estimators=100, random_state=100)
```

붓꽃 데이터로 단일 트리와 앙상블들을 5-fold 교차검증으로 비교해봤다.

![트리 계열 모델 비교](/images/ml/review_트리모델_정확도.png)

붓꽃은 워낙 쉬운 데이터라 차이가 크진 않지만(단일 트리 0.960, 랜덤 포레스트 0.967, AdaBoost 0.953, GBM 0.960), 보통 데이터가 복잡하고 노이즈가 많을수록 앙상블의 이점이 커진다.

---

## 💡 정리하며

이번 복습으로 계보가 정리됐다. **단일 나무**는 직관적이지만 과적합에 약하고, 이를 보완하려고 **배깅(랜덤 포레스트)**과 **부스팅(GBM 계열)**이 나왔다. 배깅은 병렬로 다양성을, 부스팅은 순차로 오차 보정을 노린다는 점이 핵심이다.

앞서 정리했던 학생 건강상태 분류 프로젝트에서 LightGBM·CatBoost를 썼던 게 다 이 부스팅 계열이었다는 걸 다시 확인하니 연결이 됐다.

다음 편은 실무에서 자주 부딪히는 데이터 불균형 처리다.
