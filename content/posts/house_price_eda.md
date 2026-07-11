+++
title = "[ML] Kaggle House Price 데이터로 배우는 머신러닝 프로젝트 전체 프로세스"
date = 2026-07-07
draft = false
tags = ["Python", "Machine Learning", "Kaggle", "Pandas", "EDA", "Scikit-learn", "RandomForest"]
categories = ["ml"]
+++

Kaggle의 House Prices 데이터셋으로 머신러닝 프로젝트의 전체 프로세스를 처음부터 끝까지 한 번 실습해보았다.

이번에는 모델 하나만 덜렁 학습시키는 게 아니라, **EDA → 결측치 처리 → 범주형 변수 인코딩 → 상관관계 분석 → Pipeline 기반 모델 학습**까지 실제 프로젝트 흐름을 그대로 따라가 보는 데 목적을 두었다.

이번 글에서는 그 과정을 단계별로 정리하고, 실습 중 겪었던 실수까지 함께 기록해보았다.

---

# 📌 프로젝트 전체 흐름

이번 실습은 아래와 같은 순서로 진행했다.

```text
데이터 로드

↓

EDA (구조/분포 확인)

↓

결측치 처리

↓

범주형 변수 인코딩

↓

상관관계 분석

↓

Pipeline 구축 및 모델 학습

↓

예측 및 제출 파일 생성
```

---

# 1️⃣ 데이터 살펴보기 (EDA)

가장 먼저 `train.csv`를 불러와 데이터의 기본 구조를 확인했다.

```python
train_df = pd.read_csv('.../train.csv')

print("--- Train Data Info ---")
print(train_df.info())

print("--- Train Data Description ---")
display(train_df.describe())
```

전체 1460개 행, 81개 컬럼으로 이루어진 데이터였고, `PoolQC`, `MiscFeature`, `Alley`, `Fence`처럼 데이터가 거의 비어있는 컬럼들이 눈에 띄었다. 이 컬럼들은 삭제할지, 아니면 `'None'`이라는 새로운 카테고리로 채울지 미리 고민이 필요한 부분이었다.

## 타겟 변수(SalePrice) 분포 확인

모델이 예측해야 할 타겟인 `SalePrice`의 분포부터 확인해보았다.

```python
plt.figure(figsize=(10, 5))
sns.histplot(train_df['SalePrice'], kde=True)
plt.title('Distribution of SalePrice')
plt.show()
```

### 📷 실행 결과

![SalePrice 분포 확인](/images/ml/SalePrice분포확인.png)

오른쪽으로 꼬리가 긴(right-skewed) 분포였다. 이런 분포는 회귀 모델을 학습시킬 때 모델이 고가 주택 같은 큰 값에 과도하게 반응하거나 학습이 불안정해질 수 있다는 특징이 있다는 걸 알게 되었다.

---

# 2️⃣ 결측치(Missing Value) 처리

## 결측치 확인

결측치 개수와 비율을 함께 확인해서 어떤 컬럼을 어떻게 처리할지 판단했다.

```python
missing_data = train_df.isnull().sum().sort_values(ascending=False)
missing_percentage = (train_df.isnull().sum() / len(train_df) * 100).sort_values(ascending=False)
missing_table = pd.concat([missing_data, missing_percentage], axis=1, keys=['Total', 'Percent'])
```

`PoolQC`(99.5%), `MiscFeature`(96.3%), `Alley`(93.8%), `Fence`(80.8%) 순으로 결측치 비율이 매우 높았다.

## 처리 전략

결측치 비율에 따라 처리 방식을 나눠서 적용했다.

- **80% 이상 비어있는 컬럼**: 컬럼 자체를 삭제
- **범주형 컬럼**: `'None'`이라는 새로운 카테고리로 채움
- **수치형 컬럼**: 중앙값(또는 의미에 맞는 값)으로 채움
- **나머지 소수 결측치**: 최빈값으로 채움

```python
cols_to_drop = ['PoolQC', 'MiscFeature', 'Alley', 'Fence']
train_df = train_df.drop(columns=cols_to_drop)

cat_cols = ['MasVnrType', 'FireplaceQu', 'GarageType', 'GarageFinish', 'GarageQual', 'GarageCond',
            'BsmtExposure', 'BsmtFinType2', 'BsmtQual', 'BsmtCond', 'BsmtFinType1']
for col in cat_cols:
    train_df[col] = train_df[col].fillna('None')

train_df['LotFrontage'] = train_df['LotFrontage'].fillna(train_df['LotFrontage'].median())
train_df['GarageYrBlt'] = train_df['GarageYrBlt'].fillna(0)
train_df['MasVnrArea'] = train_df['MasVnrArea'].fillna(0)

train_df = train_df.fillna(train_df.mode().iloc[0])
```

> **핵심 정리**
>
> 결측치가 많다고 무조건 삭제하는 게 아니라, 컬럼의 의미(범주형인지 수치형인지, 결측이 "없음"을 뜻하는지)를 먼저 파악한 뒤 처리 방식을 결정해야 한다.

---

# 3️⃣ 범주형 변수 인코딩 (Label Encoding)

컴퓨터는 문자를 직접 이해하지 못하기 때문에, 범주형 데이터를 숫자로 바꿔줘야 한다. 여기서는 가장 간단한 Label Encoding을 적용했다.

```python
from sklearn.preprocessing import LabelEncoder

cat_cols = train_df.select_dtypes(include=['object']).columns

for col in cat_cols:
    le = LabelEncoder()
    train_df[col] = le.fit_transform(train_df[col].astype(str))
```

`astype(str)`을 먼저 적용한 이유는, 결측치 처리 과정에서 남은 `None` 객체나 `float`형 데이터가 컬럼에 섞여 있을 수 있기 때문이다. `LabelEncoder`는 입력값 타입이 일관될 때 가장 안정적으로 동작하므로, 모든 값을 문자열로 통일해 인코딩 오류를 예방했다.

---

# 4️⃣ 상관관계 분석 (Correlation Analysis)

인코딩까지 끝난 뒤에는 어떤 변수가 `SalePrice`와 가장 관련이 깊은지 확인했다. 이 과정을 통해 중요하지 않은 변수를 걸러내거나, 어떤 특성을 가진 집이 비싼지 파악할 수 있다.

```python
corr_matrix = train_df.corr()
top_10_corr = corr_matrix['SalePrice'].sort_values(ascending=False).head(11)

plt.figure(figsize=(10, 8))
sns.heatmap(train_df[top_10_corr.index].corr(), annot=True, cmap='coolwarm', fmt=".2f")
plt.title('Top 10 Features Correlated with SalePrice')
plt.show()
```

### 📷 실행 결과

![상관관계 히트맵](/images/ml/상관관계히트맵.png)

`OverallQual`(0.79), `GrLivArea`(0.71), `GarageCars`(0.64) 순으로 `SalePrice`와 상관관계가 높게 나타났다. 집의 전체적인 품질, 거실 면적, 차고 규모가 가격에 큰 영향을 준다는 걸 수치로 확인할 수 있었다.

---

# 5️⃣ Pipeline으로 머신러닝 모델 학습 (RandomForest)

마지막으로 회귀 문제에 자주 쓰이는 RandomForestRegressor를 이용해 가격을 예측했다. 이번에는 전처리 과정을 직접 하나씩 적용하는 대신, `Pipeline`과 `ColumnTransformer`로 묶어서 처리했다.

```python
numeric_transformer = Pipeline(steps=[
    ('imputer', SimpleImputer(strategy='median')),
    ('scaler', StandardScaler())
])

categorical_transformer = Pipeline(steps=[
    ('imputer', SimpleImputer(strategy='constant', fill_value='None')),
    ('encoder', OneHotEncoder(handle_unknown='ignore'))
])

preprocessor = ColumnTransformer(
    transformers=[
        ('num', numeric_transformer, numeric_features),
        ('cat', categorical_transformer, categorical_features)
    ])

model_pipeline = Pipeline(steps=[
    ('preprocessor', preprocessor),
    ('regressor', RandomForestRegressor(n_estimators=100, random_state=42))
])

model_pipeline.fit(X, y)
predictions = model_pipeline.predict(test_df)
```

`Pipeline`으로 묶으면 train 데이터에 적용한 전처리 규칙(스케일링 기준, 인코딩 규칙 등)이 그대로 test 데이터에도 동일하게 적용된다는 장점이 있다. 특히 `OneHotEncoder(handle_unknown='ignore')`를 써두면, test 데이터에만 등장하는 새로운 카테고리가 있어도 에러 없이 처리할 수 있다.

---

# 🐛 실습 중 겪은 실수

제출 파일을 저장하는 마지막 단계에서 `sample_submission.csv`를 `sample_submission2.csv`로 잘못 입력해서 `FileNotFoundError`를 만났다.

```text
FileNotFoundError: [Errno 2] No such file or directory: '.../sample_submission2.csv'
```

데이터 로드부터 전처리, 모델 학습, 예측까지는 전부 문제없이 돌아갔는데, 정작 마지막 파일 저장 단계에서 오타 하나 때문에 막힌 경험이었다. 파이프라인이 아무리 잘 짜여 있어도 경로나 파일명 같은 사소한 부분은 항상 한 번 더 확인해야 한다는 걸 새삼 느꼈다.

---

# 📊 전체 프로세스 정리

| 단계 | 내용 | 사용 기술 |
|------|------|-----------|
| 1. EDA | 데이터 구조/타겟 분포 확인 | `info()`, `describe()`, `histplot` |
| 2. 결측치 처리 | 컬럼 삭제 + 범주형/수치형 대체 | `dropna`, `fillna` |
| 3. 인코딩 | 범주형 → 숫자 변환 | `LabelEncoder` |
| 4. 상관관계 분석 | 타겟과 상관관계 높은 변수 파악 | `corr()`, `heatmap` |
| 5. 모델 학습 | 전처리~학습을 하나로 묶어 진행 | `ColumnTransformer`, `Pipeline`, `RandomForestRegressor` |

---

# 📁 전체 코드

이번 실습에 사용한 전체 코드는 아래 노트북에서 확인할 수 있다.

👉 [house_price_EDA.ipynb 전체 보기](https://github.com/heo-aio/heo-aio.github.io/blob/main/static/notebooks/house_price_EDA.ipynb)

---

# 💡 공부하면서 느낀 점

이번 실습을 통해 처음으로 EDA부터 모델 학습까지 머신러닝 프로젝트의 전체 흐름을 한 번에 경험해볼 수 있었다.

특히 인상 깊었던 부분은 전처리를 개별 코드로 하나씩 적용할 때와, `Pipeline`으로 묶어서 처리할 때의 차이였다. 개별로 처리할 때는 train과 test에 같은 규칙을 빠짐없이 똑같이 적용했는지 매번 신경 써야 했지만, `Pipeline`을 쓰니 그 부분을 훨씬 안정적으로 관리할 수 있었다.

또한 마지막에 겪은 파일명 오타 실수를 통해, 모델링 자체만큼이나 사소해 보이는 디테일을 꼼꼼히 챙기는 것도 실전에서는 중요하다는 걸 다시 한 번 느꼈다. 다음에는 여기서 나온 Feature들을 바탕으로 Feature Engineering까지 더 깊게 다뤄볼 계획이다.
