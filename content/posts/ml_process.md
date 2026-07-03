+++
title = "[ML] Kaggle Taxi Fare 데이터로 배우는 머신러닝 모델 학습 및 복습 (2) - Train/Test Split, StandardScaler, Pipeline"
date = 2026-07-04
draft = false
tags = ["Python", "Machine Learning", "Scikit-learn", "Pipeline", "StandardScaler", "Kaggle"]
categories = ["ml"]
math = true
+++

이전 글에서는 Kaggle의 **New York City Taxi Fare Prediction** 데이터셋을 이용하여

- 결측치 처리
- 이상치 제거
- 상관관계 분석(EDA)

까지 진행하였다.

이번 글에서는 전처리가 완료된 데이터를 이용하여 실제 머신러닝 모델을 학습하기 위한 과정을 정리해보려고 한다.

이번 글에서는 아래와 같은 머신러닝 프로젝트의 Process를 중심으로 정리할 것이다. (복습!!)

- Train/Test 데이터 분리
- 데이터 표준화(StandardScaler)
- 모델 학습
- Pipeline


---

# 📌 모델 학습 전 전체 흐름

데이터 전처리가 끝났다면 이제 모델을 학습할 차례이다.

일반적인 머신러닝 프로젝트는 아래와 같은 순서로 진행된다.

```text
데이터 전처리

↓

Train / Test 데이터 분리

↓

데이터 표준화

↓

모델 선택

↓

모델 학습

↓

예측

↓

성능 평가
```

이번 글에서는 이 과정을 하나씩 살펴본다.

---

# 1️⃣ Train / Test 데이터 분리

머신러닝 모델은 학습용 데이터와 평가용 데이터를 반드시 구분해야 한다.

만약 학습한 데이터를 그대로 평가한다면

> 모델이 실제로 잘 예측하는 것인지,
> 단순히 데이터를 외운 것인지 알 수 없다.

이를 방지하기 위해 Scikit-learn에서는 `train_test_split()` 함수를 제공한다.

```python
from sklearn.model_selection import train_test_split

X_train, X_test, y_train, y_test = train_test_split(
    X,
    y,
    test_size=0.2,
    random_state=42
)
```

여기서

- `test_size`
  - 테스트 데이터의 비율

- `random_state`
  - 데이터를 항상 같은 방식으로 분리하기 위한 난수 고정 값

을 의미한다.

---

## 왜 Train/Test를 나눌까?

예를 들어 학생이 시험 문제를 미리 알고 공부했다면 시험을 잘 봤다고 해서 실력이 좋다고 말하기는 어렵다.

머신러닝도 마찬가지이다.

훈련 데이터로만 평가하면 모델이 새로운 데이터를 얼마나 잘 예측하는지 알 수 없다.

따라서

- Train Data
- Test Data

위와 같이 분리하여 일반화 성능을 평가한다.

---

# 2️⃣ 데이터 표준화(StandardScaler)

각 Feature의 단위가 다르면, 모델은 값이 큰 Feature에 더 큰 영향을 받을 수 있다.

예를 들어

| Feature | 값 |
|----------|----:|
| 나이 | 28 |
| 키(cm) | 175 |
| 연봉 | 8000 |

컴퓨터는 숫자의 크기만 보기 때문에

연봉이 가장 중요한 Feature라고 오해할 수 있다.

이를 방지하기 위해 사용하는 것이

**StandardScaler**이다.

---

## StandardScaler의 원리

StandardScaler는 평균을 0, 표준편차를 1로 맞추어 데이터를 변환한다.

공식은 아래와 같다.

$$
z=\frac{x-\mu}{\sigma}
$$

- $x$ : 원본 데이터
- $\mu$ : 평균
- $\sigma$ : 표준편차

---

## 사용 방법

```python
from sklearn.preprocessing import StandardScaler

scaler = StandardScaler()

X_train_scaled = scaler.fit_transform(X_train)

X_test_scaled = scaler.transform(X_test)
```

---

## fit()과 transform()

Scikit-learn에서 가장 자주 사용하는 메서드이다.

| 메서드 | 역할 |
|---------|------|
| fit() | 데이터의 평균과 표준편차 학습 |
| transform() | 학습한 기준으로 데이터 변환 |
| fit_transform() | 학습과 변환을 동시에 수행 |

---

## 왜 Test 데이터에는 fit을 하지 않을까?

많은 초보자가 실수하는 부분이다.

잘못된 예시는 아래와 같다.

```python
scaler.fit(X_train)

scaler.fit(X_test)
```

이렇게 하면 테스트 데이터의 정보를 모델이 미리 알게 된다.

이를 **데이터 누수(Data Leakage)** 라고 한다.

올바른 방법은

```python
scaler.fit(X_train)

X_train = scaler.transform(X_train)

X_test = scaler.transform(X_test)
```

훈련 데이터에서 학습한 기준을 테스트 데이터에도 그대로 적용하는 것이다.

---

# 3️⃣ 머신러닝 모델 학습

데이터가 준비되었다면 모델을 학습시킨다.

Scikit-learn에서는 대부분의 모델이 동일한 인터페이스를 제공한다.

예를 들어

Decision Tree를 사용한다면

```python
from sklearn.tree import DecisionTreeRegressor

model = DecisionTreeRegressor(random_state=42)

model.fit(X_train, y_train)
```

`fit()`은

훈련 데이터를 이용하여 모델을 학습시키는 과정이다.

---

# 4️⃣ 예측(Predict)

학습이 끝난 모델은

새로운 데이터를 예측할 수 있다.

```python
prediction = model.predict(X_test)
```

`predict()`는

처음 보는 데이터에 대한 예측 결과를 반환한다.

---

# 5️⃣ 모델 성능 평가

모델이 얼마나 잘 예측했는지 확인해야 한다.

회귀 문제에서는 대표적으로 아래와 같은 방법을 사용한다.

- MAE
- MSE
- RMSE
- R² Score


예를 들어

```python
from sklearn.metrics import mean_squared_error

mse = mean_squared_error(y_test, prediction)

print(mse)
```

를 이용하여 평균제곱오차(MSE)를 계산할 수 있다.

모델을 비교할 때는

여러 평가 지표를 함께 확인하는 것이 중요하다.

---

# 6️⃣ Pipeline

머신러닝 프로젝트에서는

전처리 과정이 점점 많아진다.

예를 들어

```text
Train/Test Split

↓

StandardScaler

↓

모델 학습

↓

예측
```

이러한 과정을 각각 수행하면 코드가 길어지고 실수할 가능성도 커진다.

이를 해결하기 위해 사용하는 기능이 **Pipeline**이다.

---

## Pipeline 사용 예제

```python
from sklearn.pipeline import Pipeline
from sklearn.preprocessing import StandardScaler
from sklearn.tree import DecisionTreeRegressor

pipeline = Pipeline([
    ("scaler", StandardScaler()),
    ("model", DecisionTreeRegressor())
])

pipeline.fit(X_train, y_train)

prediction = pipeline.predict(X_test)
```

Pipeline을 사용하면 전처리부터 모델 학습까지 하나의 객체에서 관리할 수 있다.

---

## Pipeline의 장점

- 코드가 간결해진다.
- 데이터 누수를 방지할 수 있다.
- 전처리를 잊지 않는다.
- 모델 교체가 쉽다.
- GridSearchCV와 함께 사용하기 좋다.

실무에서도 Pipeline은 매우 자주 사용하는 기능이다.

---

# 📊 지금까지의 전체 흐름

이번 실습을 처음부터 다시 정리하면 다음과 같다.

```text
CSV 데이터 불러오기

↓

결측치 처리

↓

이상치 제거

↓

EDA

↓

Train / Test Split

↓

StandardScaler

↓

모델 학습

↓

Predict

↓

성능 평가

↓

Pipeline 적용
```

머신러닝 프로젝트는 단순히 모델 하나를 만드는 것이 아니라,

데이터를 준비하고, 전처리하고, 학습시키고, 평가하는 전체 과정을 체계적으로 수행하는 것이 중요하다.

---

# 💡 공부하면서 느낀 점

처음에는 머신러닝 모델만 잘 선택하면 좋은 결과가 나올 것이라고 생각했다.

하지만 실제로 공부해보니 모델보다 더 중요한 것은 데이터를 어떻게 준비하고 전처리하느냐였다.

Train/Test 데이터를 분리하여 일반화 성능을 평가하고, StandardScaler를 통해 데이터의 단위를 맞추며,

Pipeline으로 전체 과정을 하나의 흐름으로 관리하는 것이 머신러닝 프로젝트에서 매우 중요한 과정이라는 점을 이해하게 되었다.

---