+++
title = "[ML] Scikit-learn 핵심 개념 정리 - 데이터 전처리부터 모델 학습까지"
date = 2026-07-03
draft = false
tags = ["Python", "Scikit-learn", "Machine Learning", "Data Science"]
categories = ["ml"]
+++

머신러닝을 처음 공부하면 가장 많이 접하게 되는 라이브러리 중 하나가 **Scikit-learn(sklearn)** 이다.

딥러닝에서는 PyTorch나 TensorFlow를 많이 사용하지만,

전통적인 머신러닝에서는 Scikit-learn이 사실상 표준 라이브러리라고 할 수 있다.

이번 글에서는 강의를 들으며 정리한 내용을 바탕으로

- Scikit-learn이 무엇인지
- 머신러닝 프로젝트에서 어떤 역할을 하는지
- 자주 사용하는 핵심 기능은 무엇인지

를 정리해보았다.

---

# 📌 Scikit-learn이란?

**Scikit-learn(sklearn)** 은 Python 기반의 대표적인 머신러닝 라이브러리이다.

데이터 전처리부터 모델 학습, 예측, 성능 평가까지 머신러닝 프로젝트에 필요한 대부분의 기능을 제공한다.

대표적으로 아래와 같은 기능을 지원한다.

- 데이터 분할
- 데이터 전처리
- 분류(Classification)
- 회귀(Regression)
- 군집화(Clustering)
- 모델 평가
- 파이프라인(Pipeline)

덕분에 머신러닝 모델을 빠르게 구현하고 실험할 수 있어 교육과 실무 모두에서 널리 사용된다.

---

# 📌 머신러닝 프로젝트의 전체 흐름

Scikit-learn을 사용한 머신러닝 프로젝트는 일반적으로 아래와 같은 순서로 진행된다.

```text
데이터 수집

↓

데이터 전처리

↓

훈련 / 테스트 데이터 분할

↓

모델 선택

↓

모델 학습 (fit)

↓

예측 (predict)

↓

성능 평가

↓

모델 개선
```

이번 글에서 정리하는 기능들도 모두 이 과정 속에서 사용된다.

---

# 1️⃣ 데이터 분할: train_test_split

머신러닝 모델을 학습하기 전에 가장 먼저 해야 하는 작업 중 하나가 **데이터를 훈련용과 테스트용으로 나누는 것**이다. :contentReference[oaicite:0]{index=0}

---

## 왜 데이터를 나눌까?

학생이 시험 문제를 미리 보고 공부하면, 시험 점수가 높더라도 실력이 뛰어나다고 말하기 어렵다.

머신러닝도 마찬가지이다.

학습한 데이터만 평가하면 모델이 단순히 외운 것인지,

새로운 데이터도 잘 예측하는지 알 수 없다.

그래서

- 학습용 데이터(Training Set)
- 테스트용 데이터(Test Set)

위와 같이 분리하여 모델의 일반화 성능을 평가한다.

---

## train_test_split의 역할

Scikit-learn에서는

`train_test_split()`

함수를 이용하여 데이터를 쉽게 나눌 수 있다.

보통 아래와 같은 비율을 많이 사용한다.

- 80% : 학습
- 20% : 테스트


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
  - 테스트 데이터 비율

- `random_state`
  - 데이터를 항상 같은 방식으로 섞기 위한 난수 고정 값

`random_state`를 지정하면

실행할 때마다 동일한 결과를 얻을 수 있기 때문에 실험을 재현(Reproducibility)하기 쉽다.

---

## 데이터 분할 과정

```text
전체 데이터 (100%)

↓

train_test_split()

↓

훈련 데이터 (80%)

+

테스트 데이터 (20%)
```

이후에는 훈련 데이터로 모델을 학습시키고, 테스트 데이터로 성능을 평가한다.

---

# 2️⃣ 데이터 표준화: StandardScaler

머신러닝에서는 각 변수의 단위가 크게 다르면

모델이 특정 변수에 더 큰 영향을 받는 문제가 발생할 수 있다. :contentReference[oaicite:1]{index=1}

이를 해결하기 위해 사용하는 것이 **StandardScaler**이다.

---

## 왜 표준화가 필요할까?

예를 들어

사람의 건강 데이터를 이용하여 질병을 예측한다고 가정해보자.

| 변수 | 값 |
|------|----:|
| 키(cm) | 170 |
| 몸무게(kg) | 70 |
| 나이 | 28 |

컴퓨터는 숫자의 크기만 보기 때문에 170이라는 값이 28보다 훨씬 중요하다고 오해할 수도 있다.

이러한 영향을 줄이기 위해 각 변수의 평균을 0, 표준편차를 1로 맞추는 작업을 수행한다. :contentReference[oaicite:2]{index=2}

---

## StandardScaler의 동작 방식

StandardScaler는 아래 공식을 이용한다.

$$
z=\frac{x-\mu}{\sigma}
$$

- $x$ : 원래 데이터
- $\mu$ : 평균
- $\sigma$ : 표준편차

즉, 모든 데이터를 평균이 0, 표준편차가 1인 형태로 변환한다.

---

## 코드 예제

```python
from sklearn.preprocessing import StandardScaler

scaler = StandardScaler()

X_train_scaled = scaler.fit_transform(X_train)

X_test_scaled = scaler.transform(X_test)
```

---

## fit(), transform(), fit_transform()

Scikit-learn을 공부하면 반드시 알아야 하는 메서드이다.

| 메서드 | 역할 |
|---------|------|
| fit() | 데이터의 평균과 표준편차를 학습 |
| transform() | 학습한 기준으로 데이터를 변환 |
| fit_transform() | 학습과 변환을 동시에 수행 |

---

## 왜 Test 데이터에는 fit을 하지 않을까?

초보자가 가장 많이 실수하는 부분이다.

잘못된 예시는 아래와 같다.

```python
scaler.fit(X_train)

scaler.fit(X_test)
```

이렇게 하면 테스트 데이터의 평균과 표준편차를 모델이 미리 알게 된다.

즉, 평가 전에 정답을 일부 알려주는 것과 비슷한 상황이 되어 공정한 성능 평가가 어려워진다.


올바른 방법은 아래 처럼

```python
scaler.fit(X_train)

X_train = scaler.transform(X_train)

X_test = scaler.transform(X_test)
```

훈련 데이터에서 학습한 기준을 테스트 데이터에도 그대로 적용하는 것이다.

---

## 언제 사용할까?

StandardScaler는 특히 아래와 같은 모델에서 자주 사용된다.

- Logistic Regression
- SVM
- KNN
- PCA
- Neural Network

반면

Decision Tree나 Random Forest처럼

트리 기반 모델은 데이터의 크기에 크게 영향을 받지 않기 때문에 표준화가 필수는 아니다.

---

## 지금까지의 흐름

지금까지의 과정을 정리하면 다음과 같다.

```text
CSV 읽기

↓

train_test_split

↓

StandardScaler

↓

모델 선택

↓

(다음 단계)
```

다음으로는 Scikit-learn에서 가장 많이 사용하는 머신러닝 모델 중 하나인

**DecisionTreeClassifier**와 전처리부터 학습까지 자동으로 연결해주는 **Pipeline**에 대해 정리해보겠다.

---

# 3️⃣ 분류 모델: DecisionTreeClassifier

데이터를 전처리했다면, 이제 모델을 선택하여 학습시켜야 한다.

Scikit-learn에서는 다양한 머신러닝 알고리즘을 제공하는데,

그중 가장 이해하기 쉬운 모델 중 하나가 **DecisionTreeClassifier(의사결정나무)** 이다. :contentReference[oaicite:0]{index=0}

---

## Decision Tree란?

Decision Tree는 사람의 의사결정 과정을 나무(Tree) 구조로 표현한 머신러닝 알고리즘이다.

질문을 하나씩 던지면서 최종 결과를 예측한다.


예를 들어 동물을 분류한다고 하면

```text
몸무게가 50kg 이상인가?

          │

     ┌────┴────┐

    예         아니오

    │            │

줄무늬가 있는가?   고양이

    │

 ┌──┴──┐

예      아니오

│        │

호랑이    사자
```

이처럼

질문을 반복하면서

가장 적절한 결과를 찾아간다.

---

## 코드 예제

```python
from sklearn.tree import DecisionTreeClassifier

model = DecisionTreeClassifier(random_state=42)

model.fit(X_train, y_train)
```

여기서

`fit()`은

훈련 데이터를 이용하여

Decision Tree를 학습시키는 과정이다.

---

## 학습이 끝난 후에는?

모델이 학습되었다면

새로운 데이터를 예측할 수 있다.

```python
prediction = model.predict(X_test)
```

`predict()`는

처음 보는 데이터에 대해 모델이 예측한 결과를 반환한다.

---

# 📌 Scikit-learn의 공통 인터페이스

Scikit-learn의 가장 큰 장점 중 하나는

거의 모든 머신러닝 모델이 동일한 인터페이스를 사용한다는 점이다.

예를 들어 Decision Tree뿐 아니라

Random Forest,

SVM,

Logistic Regression,

KNN 등도

아래와 같은 형태로 사용할 수 있다.

```python
model.fit(X_train, y_train)

prediction = model.predict(X_test)

score = model.score(X_test, y_test)
```

즉, 모델만 바꾸면 나머지 코드 대부분을 그대로 사용할 수 있다.

이러한 일관성 덕분에 다양한 알고리즘을 빠르게 비교하고 실험할 수 있다.

---

# 4️⃣ 머신러닝 자동화: Pipeline

머신러닝 프로젝트에서는

모델 학습 전에 수행해야 하는 작업이 많다.

예를 들어

```text
데이터 분할

↓

StandardScaler

↓

모델 학습

↓

예측
```

위 처럼 여러 단계를 순서대로 수행해야 한다.

이를 하나로 묶어주는 기능이

**Pipeline**이다. :contentReference[oaicite:1]{index=1}

---

## 왜 Pipeline이 필요할까?

Pipeline을 사용하지 않으면 각 단계를 직접 실행해야 한다.

```python
scaler.fit(X_train)

X_train = scaler.transform(X_train)

model.fit(X_train, y_train)
```

프로젝트가 커질수록 전처리 과정이 많아지고 실수할 가능성도 커진다.

Pipeline은 이러한 작업을 하나의 흐름으로 연결해준다.

---

## Pipeline 예제

```python
from sklearn.pipeline import Pipeline
from sklearn.preprocessing import StandardScaler
from sklearn.svm import SVC

pipeline = Pipeline([
    ("scaler", StandardScaler()),
    ("model", SVC())
])

pipeline.fit(X_train, y_train)

prediction = pipeline.predict(X_test)
```

Pipeline을 사용하면

전처리부터 모델 학습,

예측까지 하나의 객체에서 관리할 수 있다.

---

## Pipeline의 장점

- 코드가 간결해진다.
- 전처리 과정을 잊지 않는다.
- 데이터 누수(Data Leakage)를 방지할 수 있다.
- 모델 교체가 쉽다.
- GridSearchCV와 함께 사용하기 좋다.

실무에서도 Pipeline은 매우 자주 사용되는 기능이다.

---

# 📊 모델 성능 평가

모델을 학습했다고 해서 끝나는 것은 아니다.

예측 결과가 얼마나 정확한지도 확인해야 한다.

Scikit-learn에서는 다양한 평가 지표를 제공한다.

| 평가 지표 | 설명 |
|-----------|------|
| Accuracy | 전체 예측 중 맞춘 비율 |
| Precision | Positive라고 예측한 것 중 실제 Positive 비율 |
| Recall | 실제 Positive 중 맞춘 비율 |
| F1 Score | Precision과 Recall의 조화평균 |

가장 많이 사용하는 것은

```python
from sklearn.metrics import accuracy_score

accuracy = accuracy_score(y_test, prediction)

print(accuracy)
```

이다.

단, 데이터가 불균형한 경우에는 Accuracy만으로 모델 성능을 판단하기 어렵기 때문에

Precision,

Recall,

F1 Score도 함께 확인하는 것이 좋다.

---

# 📚 Scikit-learn에서 자주 사용하는 모듈

Scikit-learn은 다양한 기능을 모듈 단위로 제공한다.

자주 사용하는 모듈을 정리하면 다음과 같다.

| 모듈 | 주요 기능 |
|------|----------|
| `model_selection` | train_test_split, GridSearchCV |
| `preprocessing` | StandardScaler, MinMaxScaler |
| `pipeline` | Pipeline |
| `metrics` | Accuracy, Precision, Recall |
| `tree` | DecisionTreeClassifier |
| `linear_model` | LinearRegression, LogisticRegression |
| `svm` | SVC |
| `ensemble` | RandomForest, GradientBoosting |

머신러닝 프로젝트를 진행하다 보면

이 모듈들은 거의 항상 사용하게 된다.

---

# 🚀 머신러닝 프로젝트 전체 흐름

지금까지 배운 내용을 하나의 흐름으로 정리하면 다음과 같다.

```text
CSV 데이터 불러오기

↓

EDA(데이터 탐색)

↓

결측치 처리

↓

train_test_split

↓

StandardScaler

↓

모델 선택

↓

fit()

↓

predict()

↓

성능 평가

↓

모델 개선 및 튜닝
```

Scikit-learn은

이 전체 과정을 매우 쉽게 구현할 수 있도록 다양한 도구를 제공한다.

---

# 📋 핵심 개념 요약

| 개념 | 역할 |
|------|------|
| train_test_split | 훈련 데이터와 테스트 데이터 분리 |
| StandardScaler | 데이터 표준화 |
| fit() | 모델 또는 전처리 기준 학습 |
| transform() | 데이터를 변환 |
| fit_transform() | 학습과 변환을 동시에 수행 |
| predict() | 새로운 데이터 예측 |
| score() | 모델 성능 평가 |
| Pipeline | 전처리부터 학습까지 하나로 연결 |

---

# 💡 공부하면서 느낀 점

Scikit-learn을 처음 접했을 때는 각 함수의 사용법만 익히면 된다고 생각했다.

하지만 학습을 진행하면서 중요한 것은 개별 API를 암기하는 것이 아니라,

**머신러닝 프로젝트 전체 흐름 속에서 각각의 기능이 어떤 역할을 하는지 이해하는 것**이라는 점을 느꼈다.

예를 들어

`train_test_split`은 공정한 성능 평가를 위해 사용되고,

`StandardScaler`는 데이터의 단위를 맞추며,

`Pipeline`은 이러한 과정을 하나의 흐름으로 자동화한다.

결국 Scikit-learn은 단순한 라이브러리가 아니라,

머신러닝 프로젝트를 체계적으로 수행할 수 있도록 도와주는 도구라는 점을 이해하게 되었다.

---