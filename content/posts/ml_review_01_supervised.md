+++
title = "[ML 복습] 지도학습 — 회귀와 분류 정리"
date = 2026-07-22T10:00:00+09:00
draft = false
tags = ["Python", "Machine Learning", "Scikit-learn", "지도학습", "복습"]
categories = ["ml"]
math = true
+++

머신러닝 기초를 다시 손에 붙이려고 복습 노트를 정리한다. 첫 편은 가장 기본이 되는 **지도학습(Supervised Learning)** 이다.

지도학습은 **정답(Label)이 있는 데이터**를 모델에 보여주며 학습시키는 방식이다. 예측하려는 값이 연속적인 숫자면 회귀, 정해진 카테고리면 분류로 나뉜다.

이번 편에서 그리는 그래프는 전부 sklearn으로 직접 돌린 결과다.

---

## 📈 회귀 — 연속적인 숫자를 예측

회귀는 집값, 기온, 매출처럼 **연속적인 실수값**을 예측하는 문제다. 입력 X와 출력 Y 사이의 패턴(함수)을 찾는 게 목표다.

가장 단순한 건 직선을 긋는 선형 회귀다. 변수가 하나면 단순 선형 회귀, 여러 개면 다중 선형 회귀다. 데이터가 곡선 형태면 특성을 제곱·세제곱으로 늘려주는 다항 회귀를 쓴다.

여기서 바로 만나는 함정이 **과적합**이다. 차수를 높일수록 학습 데이터엔 딱 맞지만, 정작 새 데이터 예측은 나빠진다.

![다항 회귀 차수에 따른 적합](/images/ml/review_회귀_다항적합.png)

1차(직선)는 데이터 흐름을 못 따라가는 과소적합, 15차는 점 하나하나를 다 외워버린 과대적합이다. 4차 정도가 원래 패턴(사인 곡선)을 가장 잘 따라간다. **복잡한 모델이 항상 좋은 게 아니다** 라는 걸 눈으로 확인할 수 있다.

### 과적합을 막는 두 가지 카드

하나는 **교차 검증**이다. 데이터를 한 번만 나누지 않고 여러 조각으로 나눠 돌아가며 검증해서, 우연히 잘 맞은 게 아닌지 확인한다.

다른 하나는 **정규화(Regularization)** 다. 모델이 특정 가중치를 너무 크게 키우지 못하게 벌점을 준다.

```python
from sklearn.linear_model import Ridge, Lasso, ElasticNet

# L1 규제 (Lasso): 불필요한 특성의 계수를 0으로 만들어 버림 (특성 선택 효과)
lasso = Lasso(alpha=10)

# L2 규제 (Ridge): 계수를 0에 가깝게 줄이되 완전히 0으로는 안 만듦
ridge = Ridge(alpha=10)

# 둘을 섞은 것이 ElasticNet
elastic = ElasticNet(alpha=10, l1_ratio=0.5)
```

`alpha`가 클수록 규제가 세진다. Lasso는 계수를 아예 0으로 만들어 변수를 걸러내고, Ridge는 0 근처로만 줄인다는 차이를 기억해두면 된다.

### 회귀 평가지표

회귀는 실제값과 예측값의 차이를 얼마나 줄였는지로 성능을 본다.

- **MSE**: 오차를 제곱해 평균낸 값. 제곱하니 큰 오차(이상치)에 민감하다.
- **RMSE**: MSE에 루트를 씌운 것. 단위가 원래 값과 같아져 해석이 쉽다.
- **MAE**: 오차의 절댓값 평균. 이상치에 덜 민감하다.
- **R²**: 1에 가까울수록 모델이 데이터를 잘 설명한다는 뜻.

```python
from sklearn.metrics import mean_squared_error, mean_absolute_error, r2_score

y_pred = model.predict(X_test)
mse = mean_squared_error(y_test, y_pred)
mae = mean_absolute_error(y_test, y_pred)
r2  = r2_score(y_test, y_pred)
```

---

## 🗂️ 분류 — 카테고리를 나누기

예측 대상이 숫자가 아니라 **참/거짓, 개/고양이, 스팸/정상** 같은 범주면 분류를 쓴다. 대표 모델 네 가지를 정리했다.

- **로지스틱 회귀**: 이름은 회귀지만 분류 모델이다. 확률을 출력하고 경계는 직선(선형)이다.
- **SVM**: 두 클래스 사이 간격(마진)을 최대로 벌리는 경계를 찾는다. 커널을 쓰면 곡선 경계도 가능하다.
- **KNN**: 새 데이터 주변 가까운 이웃 K개의 다수결로 정한다. 학습이랄 게 없고 거리 계산만 한다.
- **나이브 베이즈**: 확률(베이즈 정리) 기반이라 아주 빠르다.

같은 데이터에 이 네 모델을 씌워 결정 경계를 그려봤다.

![분류 모델별 결정 경계](/images/ml/review_분류_결정경계.png)

여기서 확실히 보이는 게 있다. 로지스틱 회귀와 나이브 베이즈는 경계가 **직선**에 가깝고, SVM(RBF)과 KNN은 데이터 모양을 따라 **곡선**으로 휘어진다. 데이터가 직선으로 안 갈리는 형태라면 곡선 경계를 만드는 모델이 유리하다는 뜻이다.

```python
from sklearn.linear_model import LogisticRegression
from sklearn.svm import SVC
from sklearn.neighbors import KNeighborsClassifier
from sklearn.naive_bayes import GaussianNB

lr  = LogisticRegression().fit(X_train, y_train)
svm = SVC(kernel='rbf', gamma=2).fit(X_train, y_train)
knn = KNeighborsClassifier(n_neighbors=15).fit(X_train, y_train)
nb  = GaussianNB().fit(X_train, y_train)
```

### 분류 평가지표와 혼동 행렬

분류 성능은 **혼동 행렬(Confusion Matrix)** 에서 출발한다. 예측과 실제를 4칸으로 나눈 표다.

![혼동 행렬](/images/ml/review_혼동행렬.png)

- **TP / TN**: 맞게 예측한 것
- **FP**: 음성인데 양성이라 잘못 예측 (거짓 양성)
- **FN**: 양성인데 음성이라 놓침 (거짓 음성)

여기서 지표들이 나온다.

- **정확도(Accuracy)**: 전체 중 맞춘 비율. 단, 불균형 데이터에선 함정이 된다.
- **정밀도(Precision)**: 양성이라 예측한 것 중 진짜 양성 비율.
- **재현율(Recall)**: 실제 양성 중 놓치지 않고 잡은 비율.
- **F1**: 정밀도와 재현율의 조화평균.

```python
from sklearn.metrics import accuracy_score, precision_score, recall_score, f1_score

accuracy  = accuracy_score(y_test, pred)
precision = precision_score(y_test, pred)
recall    = recall_score(y_test, pred)
f1        = f1_score(y_test, pred)
```

> **복습 포인트**
>
> 스팸 필터라면 정상 메일을 스팸으로 보내는 것(FP)이 치명적이라 정밀도가 중요하고, 암 진단이라면 환자를 놓치는 것(FN)이 치명적이라 재현율이 중요하다. 상황마다 봐야 할 지표가 다르다.

---

## 💡 정리하며

지도학습은 결국 "정답을 보고 패턴을 배운다"는 한 문장으로 요약된다. 이번 복습에서 스스로 다시 새긴 건 두 가지다.

하나는 **복잡한 모델이 항상 답이 아니라는 것**. 다항 회귀 그래프에서 봤듯 차수를 높이면 학습 데이터엔 잘 맞지만 과적합으로 이어진다.

다른 하나는 **평가지표를 상황에 맞게 골라야 한다는 것**. 정확도 하나만 보면 안 되고, 무엇을 놓치면 안 되는지에 따라 정밀도와 재현율 중 무엇을 볼지 정해야 한다.

다음 편은 정답이 없는 데이터를 다루는 비지도학습이다.
