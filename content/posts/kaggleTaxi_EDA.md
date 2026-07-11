+++
title = "[ML] Kaggle Taxi Fare 데이터로 배우는 데이터 전처리와 EDA - 결측치, 이상치, 상관관계, Feature Engineering"
date = 2026-07-05
draft = false
tags = ["Python", "Machine Learning", "Pandas", "EDA", "Data Preprocessing", "Feature Engineering", "Kaggle"]
categories = ["ml"]
+++

머신러닝 프로젝트에서 모델을 만드는 것만큼 중요한 과정이 **데이터 전처리(Data Preprocessing)** 이다.

실무에서는 오히려 모델을 학습시키기 전에 데이터를 정리하는 데 더 많은 시간을 사용하는 경우도 많다.

이번 글에서는 **Kaggle의 New York City Taxi Fare Prediction 데이터셋**을 활용하여 머신러닝 프로젝트에서 수행하는 데이터 전처리와 탐색적 데이터 분석(EDA) 과정을 실습한 내용을 정리해보았다.

이번 글에서는 다음 내용을 다룰 것이다.

- 데이터를 불러오는 과정
- 결측치 처리
- 이상치 제거
- 상관관계 분석(Heatmap)
- 상관관계가 약한 이유와 Feature Engineering(거리 변수 생성)

> 📝 이번 실습은 강의 플랫폼 내에서 진행하여 별도의 노트북이 남지 않았다. 그래서 이 글을 정리하면서 **Kaggle과 동일한 컬럼 구조의 재현용 샘플 데이터**로 같은 파이프라인을 다시 실행하고, 그 결과와 그래프를 함께 담았다. 코드 로직은 실습과 동일하며, 아래 결과 수치는 재현용 샘플 기준이다.

---

# 📌 머신러닝 프로젝트의 전체 흐름

머신러닝 프로젝트는 일반적으로 아래와 같은 순서로 진행된다.

```text
데이터 수집

↓

EDA(탐색적 데이터 분석)

↓

데이터 전처리

↓

Feature Engineering

↓

Train/Test 데이터 분리

↓

모델 선택

↓

모델 학습

↓

성능 평가

↓

모델 배포
```

### 📷 그림으로 한 눈에 보기

머신러닝 프로젝트의 전체 진행 프로세스를 그림으로 정리해보았다.

![머신러닝 프로젝트 진행 프로세스](/images/ml/머신러닝프로젝트진행프로세스.png)

이번 실습은 위 과정 중 **EDA와 데이터 전처리, 그리고 Feature Engineering의 첫걸음**에 해당하는 부분이다.

모델을 학습시키기 전에 데이터의 품질을 높이고, 데이터의 특성을 파악하는 것이 주요 목적이다.

---

# 📌 실습 데이터 소개

이번 실습에서는 Kaggle에서 제공하는 **New York City Taxi Fare Prediction** 데이터셋을 사용하였다.

이 데이터셋은 뉴욕 택시의 운행 기록을 기반으로

- 승객 수(passenger_count)
- 출발 위치(pickup_longitude, pickup_latitude)
- 도착 위치(dropoff_longitude, dropoff_latitude)
- 요금(fare_amount)

등의 정보를 이용하여 **택시 요금을 예측하는 회귀(Regression) 문제**이다.

이번 글에서는 모델을 학습하기 전 수행하는 데이터 전처리와 EDA 과정을 중심으로 살펴본다.

---

# 1️⃣ 데이터 불러오기

먼저 CSV 파일을 Pandas DataFrame 형태로 불러온다.

```python
import pandas as pd

def load_csv(path):
    data_frame = pd.read_csv(path)
    return data_frame

df = load_csv(DATA_PATH)
```

`pd.read_csv()`는 CSV 파일을 DataFrame으로 변환하는 Pandas의 대표적인 함수이다.

머신러닝 프로젝트에서는 대부분 CSV 형태의 데이터를 다루므로 가장 많이 사용하는 함수 중 하나이다.

---

# 2️⃣ 결측치 처리

머신러닝 모델은 결측치(Missing Value)가 존재하면 정상적으로 학습하지 못하거나 성능이 저하될 수 있다.

따라서 모델을 학습하기 전에 반드시 결측치를 처리해야 한다.

먼저 `isnull().sum()`으로 컬럼별 결측치 개수를 확인한다.

```python
print(df.isnull().sum())
```

### 출력 예시

```text
dropoff_longitude    60
passenger_count      59
...
```

이번 실습에서는

- 불필요한 컬럼 제거
- 결측치가 포함된 행 제거

를 하나의 함수로 묶어서 수행하였다.

```python
def del_missing(df):
    # ① CSV 저장 과정에서 생긴 인덱스 컬럼 제거
    del_un_df = df.drop(['Unnamed: 0'], axis='columns')
    # ② 식별자 컬럼 제거
    del_un_id_df = del_un_df.drop(['id'], axis='columns')
    # ③ 결측치가 포함된 행 제거
    removed_df = del_un_id_df.dropna()
    return removed_df

df = del_missing(df)
```

## 불필요한 컬럼 제거

`Unnamed: 0` 컬럼은 CSV 저장 과정에서 생성된 인덱스 정보이므로 머신러닝 학습에는 의미가 없다.

`id` 컬럼 역시 각 데이터를 구분하기 위한 식별자일 뿐, 요금을 예측하는 데 도움이 되는 Feature가 아니기 때문에 제거하였다.

## 결측치 제거

이후 `dropna()`를 이용하여 결측치가 포함된 행을 제거하였다.

결측치를 그대로 사용하면 모델 학습 오류, 성능 저하, 통계값 왜곡 등이 발생할 수 있다.

결측치를 처리하는 방법에는 삭제(Drop), 평균값 대체, 중앙값 대체, 최빈값 대체 등이 있으며, 이번 실습에서는 가장 간단한 **삭제(Drop)** 방식을 사용하였다.

---

# 3️⃣ 이상치(Outlier) 제거

데이터에는 실제 값이라고 보기 어려운 이상치(Outlier)가 존재할 수 있다.

이러한 데이터는 모델이 잘못된 패턴을 학습하도록 만들 수 있기 때문에 적절한 처리가 필요하다.

이번 실습에서는 세 가지 이상치를 제거하였다.

```python
# ① 음수 요금 제거 (요금은 음수가 될 수 없음)
df = df[df['fare_amount'] >= 0]

# ② 음수 승객 수 제거 (승객 수는 음수가 될 수 없음)
df = df[df['passenger_count'] >= 0]

# ③ 출발 위치와 도착 위치가 동일한 데이터 제거 (이동 거리 0)
same_mask = (df['pickup_longitude'] == df['dropoff_longitude']) & \
            (df['pickup_latitude'] == df['dropoff_latitude'])
df = df[~same_mask]
```

## ① 음수 요금 제거

택시 요금은 음수가 될 수 없기 때문에 입력 오류로 판단할 수 있다.

## ② 음수 승객 수 제거

승객 수 역시 음수가 될 수 없으므로 비정상적인 데이터이다.

## ③ 출발 == 도착 위치 제거

출발 위치와 도착 위치가 완전히 동일한 데이터는 이동 거리가 0인 경우로, 일반적인 택시 운행 데이터와는 다른 특성을 가지므로 모델 학습에 방해가 될 수 있다.

> 다만 실제 프로젝트에서는 이상치를 무조건 삭제하기보다 입력 오류인지, 실제 발생 가능한 데이터인지를 먼저 판단한 후 처리 방법을 결정하는 것이 중요하다. 예를 들어 VIP 고객의 매우 높은 구매 금액은 이상치처럼 보이지만 중요한 데이터일 수도 있다.

---

# 4️⃣ 상관관계(Correlation) 분석

데이터 전처리가 끝났다면 변수들 간의 관계를 확인한다.

```python
corr_df = df.corr(numeric_only=True)

plt.figure(figsize=(15, 10))
sns.heatmap(corr_df, annot=True, cmap="PuBu")
plt.show()
```

상관계수는

- **1**에 가까울수록 강한 양의 상관관계
- **-1**에 가까울수록 강한 음의 상관관계
- **0**에 가까울수록 관계가 거의 없음

을 의미한다.

### 📷 실행 결과 (raw 특성)

![택시 요금 상관관계 히트맵 (raw)](/images/ml/택시요금상관관계히트맵.png)

---

# 📌 여기서 발견한 것 — 왜 상관관계가 이렇게 약할까?

히트맵을 확인해보니 예상과 달리 출발/도착 좌표(위도·경도)는 `fare_amount`와 상관계수가 거의 **0**에 가까웠다.

```text
fare_amount          1.000000
dropoff_longitude    0.032158
pickup_longitude     0.026080
passenger_count      0.025868
dropoff_latitude     0.020252
pickup_latitude     -0.003038
```

처음에는 "좌표가 요금이랑 관련 있는 거 아니야?"라고 생각했는데, 다시 생각해보니 당연한 결과였다.

요금은 **특정 위도·경도 값**에 비례하는 게 아니라, **출발지에서 도착지까지 얼마나 멀리 갔는지(이동 거리)** 에 비례하기 때문이다.

즉, 좌표 4개를 따로따로 보면 요금과 아무 관계가 없어 보이지만, 이 좌표들을 조합해서 **거리**라는 새로운 정보로 바꿔주면 이야기가 완전히 달라진다.

→ 그래서 다음 단계로 **Feature Engineering(거리 변수 생성)** 을 직접 해보았다.

---

# 5️⃣ Feature Engineering — 이동 거리 만들기

출발/도착 좌표로부터 실제 이동 거리(km)를 계산하는 `distance_km` 컬럼을 새로 만들었다.

위경도 두 지점 사이의 거리는 **Haversine 공식**으로 구할 수 있다.

```python
import numpy as np

def haversine(lon1, lat1, lon2, lat2):
    # 위경도 -> km 단위 거리
    lon1, lat1, lon2, lat2 = map(np.radians, [lon1, lat1, lon2, lat2])
    dlon = lon2 - lon1
    dlat = lat2 - lat1
    a = np.sin(dlat / 2) ** 2 + np.cos(lat1) * np.cos(lat2) * np.sin(dlon / 2) ** 2
    return 2 * 6371 * np.arcsin(np.sqrt(a))

df['distance_km'] = haversine(
    df['pickup_longitude'], df['pickup_latitude'],
    df['dropoff_longitude'], df['dropoff_latitude']
)
```

이렇게 만든 `distance_km`을 포함해 다시 상관관계를 확인해보았다.

```text
fare_amount          1.000000
distance_km          0.898807   ← 새로 만든 거리 변수
dropoff_longitude    0.032158
pickup_longitude     0.026080
passenger_count      0.025868
```

### 📷 실행 결과 (거리 Feature 추가 후)

![택시 요금 거리 Feature 히트맵](/images/ml/택시요금거리Feature히트맵.png)

새로 만든 `distance_km`은 `fare_amount`와 상관계수 **약 0.9**로 매우 강한 관계를 보였다.

> **핵심 정리**
>
> 같은 데이터라도 raw 좌표(상관계수 ~0)를 그대로 쓰는 것과, 이를 거리(상관계수 0.9)로 가공하는 것은 모델이 학습할 수 있는 신호의 질이 완전히 다르다. 이것이 EDA와 Feature Engineering을 함께 하는 이유다.

---

# 📊 이번 실습의 전체 흐름

이번 실습 과정을 정리하면 다음과 같다.

```text
CSV 데이터 불러오기

↓

불필요한 컬럼 제거

↓

결측치 제거

↓

이상치 제거

↓

상관관계 분석 (raw → 약함)

↓

Feature Engineering (거리 변수 → 강함)

↓

모델 학습 준비 완료
```

---

# 📁 전체 코드

이번 실습을 재현한 전체 노트북은 아래에서 확인할 수 있다.

👉 [kaggleTaxi_EDA.ipynb 전체 보기](https://github.com/heo-aio/heo-aio.github.io/blob/main/static/notebooks/kaggleTaxi_EDA.ipynb)

---

# 💡 공부하면서 느낀 점

처음에는 머신러닝이라고 하면 모델을 선택하고 학습시키는 과정이 가장 중요하다고 생각했다.

하지만 이번 실습을 통해 실제 프로젝트에서는 **데이터 전처리와 EDA가 모델 성능에 큰 영향을 미친다**는 점을 알게 되었다.

특히 상관관계 분석에서 좌표가 요금과 거의 무관하게 나온 것을 보고 "데이터가 별로인가?"라고 생각했는데, 좌표를 거리로 가공하자 상관계수가 0.9까지 올라가는 것을 보며 생각이 바뀌었다. 데이터가 문제가 아니라, **데이터를 어떤 관점의 Feature로 바라보느냐**가 핵심이라는 것을 직접 확인할 수 있었다.

결측치와 이상치를 처리하고 변수 간의 관계를 분석하는 과정은 단순히 데이터를 정리하는 작업이 아니라, 이후 모델이 더 좋은 성능을 낼 수 있도록 기반을 만드는 과정이라는 점을 이해할 수 있었다.
