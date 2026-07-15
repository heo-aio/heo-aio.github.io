+++
title = "[개념정리] 앙상블 학습 총정리 — Bagging, Boosting, Stacking 제대로 이해하기"
date = 2026-07-15T10:00:00+09:00
draft = false
tags = ["Python", "Machine Learning", "앙상블", "Bagging", "Boosting", "Stacking", "XGBoost", "LightGBM", "CatBoost"]
categories = ["ml"]
math = true
+++

ML 공부를 시작하고 나서 가장 자주 마주치는 단어가 "앙상블"이었다. 캐글 대회 상위권 솔루션 대부분이 부스팅 계열 모델을 쓰고, 면접 기출에도 "RandomForest와 GradientBoosting의 차이"가 단골로 등장한다. 그런데 정작 "왜 여러 모델을 합치면 더 좋아지는가"를 한 문장으로 설명하라고 하면 막혔다.

이 글은 그 막힘을 풀기 위해 개념부터 실전 코드까지 한 번에 정리한 나만의 참고 자료다. 나중에 다시 볼 걸 염두에 두고, **비유 → 원리 → 코드** 순서로 써나간다.

---

## 1️⃣ 앙상블이란? — 여러 전문가의 판단을 모으는 것

앙상블(Ensemble)은 여러 개의 모델을 결합해서 하나의 모델보다 더 나은 예측을 만드는 방법이다.

> 의사 한 명의 진단보다, 여러 전문의의 소견을 종합한 진단이 더 믿을 만하다.

핵심은 이 비유다. 모델 하나는 특정 패턴에서 실수를 할 수 있지만, **서로 다른 방식으로 학습한 여러 모델**을 조합하면 한 모델의 실수를 다른 모델이 상쇄해줄 가능성이 높아진다. 다만 이게 성립하려면 전제조건이 하나 있다 — 모델들이 **서로 다른 종류의 실수**를 해야 한다는 것. 똑같은 실수를 하는 모델을 100개 모아봐야 의미가 없다. 이 "다양성(diversity)"이 앙상블 전체를 관통하는 키워드다.

---

## 2️⃣ 왜 합치면 좋아지나 — Bias-Variance로 보기

모델의 예측 오차는 대략 이렇게 나눌 수 있다.

$$\text{Error} = \text{Bias}^2 + \text{Variance} + \text{Noise}$$

- **Bias(편향)**: 모델이 문제를 너무 단순하게 봐서 생기는 오차. "과소적합"에 가깝다.
- **Variance(분산)**: 모델이 학습 데이터의 사소한 노이즈까지 다 외워버려서 생기는 오차. "과적합"에 가깝다.

앙상블 기법마다 이 둘 중 어느 쪽을 줄이는 데 집중하는지가 다르다. 이 관점 하나만 잡고 있어도 Bagging과 Boosting이 왜 다른 상황에서 쓰이는지 헷갈리지 않는다.

- **Bagging**: 서로 독립적인 여러 모델의 예측을 평균 내서 **Variance를 줄인다.** (개별 모델은 일부러 복잡하게, 즉 과적합 위험이 있는 모델을 씀 — 예: 깊은 트리)
- **Boosting**: 이전 모델이 놓친 부분을 다음 모델이 순차적으로 보정해서 **Bias를 줄인다.** (개별 모델은 일부러 단순하게, 즉 약한 모델을 씀 — 예: 얕은 트리)

---

## 3️⃣ 3대 축 한눈에 비교

| 구분 | Bagging | Boosting | Stacking |
|---|---|---|---|
| 학습 방식 | 병렬 (모델끼리 독립적) | 순차적 (이전 모델의 오차를 다음 모델이 보정) | 서로 다른 종류의 모델 예측을 메타모델이 재학습 |
| 주로 줄이는 오차 | Variance | Bias | 상황에 따라 다름 (모델 다양성에 의존) |
| 대표 알고리즘 | RandomForest | AdaBoost, GradientBoosting, XGBoost, LightGBM, CatBoost | StackingClassifier/Regressor |
| 과적합 위험 | 낮은 편 | 튜닝 없으면 높음 | 메타모델 설계·검증 방식에 따라 다름 |
| 학습 속도 | 병렬이라 빠름 | 순차적이라 상대적으로 느림 (LightGBM은 예외적으로 빠름) | base 모델을 다 학습해야 해서 가장 느림 |

Voting은 이 표에는 없지만 자주 같이 언급된다. Stacking과 헷갈리기 쉬운데, **Voting은 단순 평균/다수결**이고 **Stacking은 "어떤 모델을 얼마나 신뢰할지"까지 메타모델이 학습**한다는 게 차이다.

---

## 4️⃣ Bagging 깊이 파기 — RandomForest

Bagging은 **B**ootstrap **Agg**regat**ing**의 줄임말이다. 원본 데이터에서 중복을 허용한 무작위 샘플링(부트스트랩)으로 여러 서브셋을 만들고, 각 서브셋마다 독립적으로 모델을 학습시킨 뒤, 결과를 평균(회귀) 또는 다수결(분류)로 합친다.

**RandomForest = Bagging + 트리 분기 시 feature 무작위 선택**이다. 트리를 나눌 때마다 전체 feature가 아니라 일부만 무작위로 후보에 올려서, 트리들끼리 서로 다른 방식으로 자라도록 강제한다. 이렇게 하면 트리 간 상관관계가 낮아져서 평균을 냈을 때 Variance가 더 확실하게 줄어든다.

```python
from sklearn.ensemble import RandomForestClassifier
from sklearn.datasets import load_breast_cancer
from sklearn.model_selection import train_test_split, cross_val_score

X, y = load_breast_cancer(return_X_y=True)
X_train, X_test, y_train, y_test = train_test_split(X, y, test_size=0.2, random_state=42)

rf = RandomForestClassifier(
    n_estimators=300,      # 트리 개수 (많을수록 안정적이지만 느려짐)
    max_depth=None,        # 트리 깊이 제한 (None이면 끝까지 분기)
    max_features="sqrt",   # 분기마다 후보로 볼 feature 개수
    min_samples_leaf=2,    # 리프 노드 최소 샘플 수 (과적합 방지)
    oob_score=True,        # 부트스트랩에서 빠진 샘플로 검증 점수 자체 계산
    n_jobs=-1,
    random_state=42,
)
rf.fit(X_train, y_train)
print("OOB score:", rf.oob_score_)
print("Test score:", rf.score(X_test, y_test))
```

`oob_score`는 개인적으로 유용하다고 느낀 옵션이다. 부트스트랩 샘플링 특성상 각 트리는 원본 데이터의 약 63%만 사용하는데, 나머지 37%(Out-Of-Bag)로 별도 검증셋 없이 성능을 추정할 수 있다.

---

## 5️⃣ Boosting 깊이 파기 — XGBoost·LightGBM·CatBoost

Boosting은 약한 모델(weak learner)을 순차적으로 추가하면서, 매번 **이전까지의 모델이 못 맞춘 부분**에 집중한다.

- **AdaBoost**: 틀린 샘플의 가중치를 높여서, 다음 모델이 그 샘플을 더 신경 쓰게 만든다.
- **Gradient Boosting**: "틀린 정도(잔차)"를 새로운 타깃으로 삼아, 잔차를 예측하는 트리를 계속 추가한다. 손실함수를 경사하강법으로 줄여나가는 과정으로 이해할 수 있다.

요즘 실전에서 많이 쓰는 세 라이브러리는 같은 Gradient Boosting 계열이지만 구현 방식이 다르다.

| 구분 | XGBoost | LightGBM | CatBoost |
|---|---|---|---|
| 트리 성장 방식 | Level-wise (균형 있게 성장) | Leaf-wise (손실 감소가 큰 리프 우선 성장) | Level-wise + Ordered Boosting |
| 강점 | 정규화(L1/L2)로 과적합 억제, 안정적인 baseline | 대용량 데이터에서 빠른 속도 | 범주형 변수 자동 처리에 강함 |
| 주의할 점 | 대용량에서는 LightGBM보다 느릴 수 있음 | leaf-wise라 작은 데이터에선 과적합 위험 ↑ | 다른 둘보다 학습 시간이 더 걸릴 수 있음 |
| 결측치 처리 | 자동 처리 | 자동 처리 | 자동 처리 |

```python
from xgboost import XGBClassifier
from lightgbm import LGBMClassifier
from catboost import CatBoostClassifier

common_params = dict(
    n_estimators=500,
    learning_rate=0.05,   # 작을수록 안정적이지만 n_estimators를 늘려야 함
    max_depth=6,
    subsample=0.8,        # 매 트리마다 데이터의 80%만 샘플링 (과적합 방지)
    colsample_bytree=0.8, # 매 트리마다 feature의 80%만 사용 (과적합 방지)
)

xgb = XGBClassifier(**common_params, reg_alpha=0.1, reg_lambda=1.0,
                     eval_metric="logloss", random_state=42)
lgbm = LGBMClassifier(**common_params, reg_alpha=0.1, reg_lambda=1.0,
                       random_state=42)
cat = CatBoostClassifier(iterations=500, learning_rate=0.05, depth=6,
                          verbose=False, random_state=42)

xgb.fit(X_train, y_train, eval_set=[(X_test, y_test)], verbose=False)
lgbm.fit(X_train, y_train)
cat.fit(X_train, y_train)
```

> 💡 세 모델 모두 `early_stopping`(조기 종료)을 지원한다. 검증 점수가 일정 라운드 동안 개선되지 않으면 학습을 멈춰서, `n_estimators`를 크게 잡아놓고도 과적합 직전에 자동으로 멈추게 할 수 있다. 실전에서는 거의 필수로 쓰게 된다.

---

## 6️⃣ Stacking 깊이 파기 — 메타모델이 다시 학습한다

Stacking은 **서로 다른 종류의 모델**(1차 모델, base learner)들의 예측 결과를 새로운 feature로 삼아, **메타모델(2차 모델)**이 최종 예측을 하도록 만드는 방법이다.

여기서 가장 중요한 포인트는 **데이터 누수(leakage) 방지**다. base 모델이 학습에 쓴 데이터를 그대로 예측해서 메타모델에 넘기면, 그 예측은 이미 과적합된 값이라 메타모델이 잘못된 신호를 학습하게 된다. 그래서 K-fold로 나눠 **학습에 쓰지 않은 fold의 예측(Out-Of-Fold)** 만 메타모델의 입력으로 쓴다.

```python
from sklearn.ensemble import StackingClassifier
from sklearn.linear_model import LogisticRegression
from sklearn.tree import DecisionTreeClassifier

estimators = [
    ("rf", RandomForestClassifier(n_estimators=200, random_state=42)),
    ("xgb", XGBClassifier(n_estimators=200, learning_rate=0.05, random_state=42)),
    ("lgbm", LGBMClassifier(n_estimators=200, learning_rate=0.05, random_state=42)),
]

stack = StackingClassifier(
    estimators=estimators,
    final_estimator=LogisticRegression(),
    cv=5,              # 5-fold로 OOF 예측을 만들어 누수를 방지
    n_jobs=-1,
)
stack.fit(X_train, y_train)
print("Stacking test score:", stack.score(X_test, y_test))
```

sklearn의 `StackingClassifier`는 `cv` 파라미터 하나로 이 OOF 로직을 알아서 처리해준다. 직접 구현하려면 `cross_val_predict`로 OOF 예측을 만들어 메타모델의 입력 데이터로 쓰면 된다.

---

## 7️⃣ 실전에서는 뭘 골라야 할까

| 상황 | 추천 |
|---|---|
| 빠르게 baseline부터 잡고 싶다 | RandomForest |
| 정형 데이터에서 최고 성능을 원한다 (캐글 등) | XGBoost / LightGBM / CatBoost |
| 범주형 변수가 많고 전처리에 시간을 쓰기 어렵다 | CatBoost |
| 데이터가 매우 크다 | LightGBM |
| 대회 마지막에 점수를 조금이라도 더 끌어올리고 싶다 | Stacking / Blending |
| 모델이 왜 이렇게 예측했는지 설명이 중요하다 | 단일 트리, 또는 RandomForest/부스팅 + SHAP |

---

## 8️⃣ 실수 조심하기!

- **다양성 없이 그냥 여러 모델을 합친다.** 비슷한 방식으로 학습한 모델을 여러 개 섞어봐야 성능 향상은 거의 없다. base 모델의 성격(트리 계열 + 선형 계열처럼)을 다르게 섞는 게 핵심이다.
- **Stacking에서 K-fold 없이 그냥 train 전체로 예측해서 메타모델에 넣는다.** 검증 점수가 실제보다 뻥튀기되는 대표적인 원인이다.
- **불균형 데이터에 튜닝 없이 부스팅을 바로 적용한다.** `class_weight`, `scale_pos_weight` 같은 옵션을 먼저 확인해야 한다.
- **`n_estimators`를 무작정 크게 주고 early stopping을 안 쓴다.** 검증셋 없이 학습하면 과적합을 확인할 방법이 없다.

---

## 📊 한 장 요약

```text
앙상블 = 여러 모델을 합쳐서 하나보다 잘 맞히기

Bagging (병렬, Variance↓)
  └ RandomForest = Bagging + feature 무작위 선택

Boosting (순차, Bias↓)
  └ XGBoost / LightGBM / CatBoost = Gradient Boosting 구현체별 차이

Stacking (메타모델이 재학습)
  └ K-fold OOF 예측으로 누수 방지가 핵심
```

---

## 💡 다음 공부 방향

이 정리를 하면서 다음에 파고들 주제도 같이 정해졌다.

- **SHAP**으로 부스팅 모델의 예측 근거를 뜯어보기 — "성능은 좋은데 왜 이렇게 예측했는지 설명이 안 된다"는 문제를 보완
- **Optuna** 같은 도구로 하이퍼파라미터 튜닝을 체계적으로 하기 (지금은 감으로 값을 넣고 있다)
- 실제로 진행했던 학생 건강상태 분류 프로젝트에 이 세 가지(RandomForest, 부스팅 3종, Stacking)를 다 적용해서 리프트를 직접 비교해보기

개념만 알고 넘어가면 금방 잊어버릴 것 같아서, 다음 글에서는 이 정리를 실제 프로젝트 코드에 적용해본 결과로 이어가려고 한다.
