+++
title = "[프로젝트] 학생 건강상태 분류 (2/4) — 12개 모델 비교 & 모델 선택"
date = 2026-07-13T14:00:00+09:00
draft = false
tags = ["Python", "Machine Learning", "Kaggle", "Scikit-learn", "Pipeline", "모델선택", "팀프로젝트"]
categories = ["Project"]
+++

[지난 편](../student_health_01_eda/)에서는 EDA와 파생변수 설계를 다뤘다. 이번 편에서는 그렇게 만든 변수들을 가지고 **12개 분류 모델을 비교하고 최종 모델을 선택하는 과정**을 정리한다.

이 단계에서 스스로 세운 목표는 "아무거나 성능 1등을 고르지 않는다, **기준**을 세우고 근거로 선택한다"였다.

---

## 📌 이 단계의 흐름

```text
변수 가지치기 (검증에서 죽은 변수 제거)

↓

인코딩 전략 (순서형=서열 / 명목형=OneHot)

↓

Pipeline 아키텍처 구성

↓

12개 모델 비교 (성능 + 시간)

↓

기준 세워 최종 모델 선택

↓

튜닝 → 전체 학습 → 제출
```

---

## 1️⃣ 변수 가지치기 — 많다고 좋은 게 아니다

1편에서 파생변수를 잔뜩 만들었지만, MI와 리프트 검증 결과를 근거로 **실제로 도움이 된 변수만** 남겼다.

- **유지**: `lifestyle_risk_score`, 서열 인코딩(`*_ord`), `sleep_debt`, `sleep_ideal`, `bmi_cat`, `cal_per_step` 등
- **제거**: 결측 지표(`*_isna`), `n_missing`, `low_water`, `hr_high`, `sleep_excess`, `bmi_abnormal` 등 (MI가 0에 가깝고 성능 기여가 없던 변수)

결과적으로 **수치·서열 18개 + 명목 2개 = 20개** 변수로 정리됐다.

---

## 2️⃣ 인코딩 전략 — 순서가 있느냐 없느냐

범주형 변수는 **값에 순서가 있는지**에 따라 인코딩을 다르게 적용했다.

| 종류 | 변수 | 인코딩 | 이유 |
|---|---|---|---|
| 순서형 | stress_level, sleep_quality, physical_activity_level, smoking_alcohol | 서열 인코딩(`*_ord`) | `low < medium < high`처럼 순서가 의미를 가지므로 숫자로 변환해 순서 정보를 반영 |
| 명목형 | diet_type, gender | One-Hot Encoding | 순서가 없어서 숫자를 부여하면 없는 순서 관계를 잘못 학습할 수 있음 |

> 순서형 변수에 OneHot과 서열 인코딩을 동시에 적용하면 같은 정보를 중복 제공하게 되므로, 최종 모델에서는 서열 인코딩 변수만 사용했다.

---

## 3️⃣ Pipeline 아키텍처

전처리와 모델을 하나의 `Pipeline`으로 묶어, 12개 모델 앞에 **동일한 전처리기**를 붙였다.

```python
numeric_pipe = Pipeline([('imputer', SimpleImputer(strategy='median')),
                         ('scaler', StandardScaler())])
categorical_pipe = Pipeline([('imputer', SimpleImputer(strategy='most_frequent')),
                             ('onehot', OneHotEncoder(handle_unknown='ignore', sparse_output=False))])
preprocessor = ColumnTransformer([('num', numeric_pipe, num_final),
                                  ('cat', categorical_pipe, cat_final)])
```

이렇게 하면 모든 모델이 정확히 같은 전처리 조건에서 비교되므로 비교가 공정해진다.

---

## 4️⃣ 12개 모델 비교 — 계열별로 의도를 갖고 배치

"10개 이상 모델 비교"가 요건이었는데, 아무거나 채우지 않고 **계열별로 의도를 갖고** 배치했다.

- **선형**: LogisticRegression, RidgeClassifier, LinearSVC
- **거리/확률 기반**: KNN, GaussianNB
- **트리/배깅**: DecisionTree, RandomForest, ExtraTrees
- **부스팅**: HistGradientBoosting, LightGBM, XGBoost, CatBoost

불균형 대응으로 지원 모델은 `class_weight='balanced'`(CatBoost는 `auto_class_weights`, XGBoost는 `sample_weight` 자동 주입)를 적용했다.

## 비교 방법

전체 69만 건으로 12개를 돌리면 너무 오래 걸리므로, **10만 건 층화 샘플**로 3-Fold 교차검증하며 상대 비교했다. 성능(balanced accuracy)뿐 아니라 **학습 시간**도 함께 측정했다.

## 결과

| 순위 | 모델 | balanced_acc | std | fit_time |
|---:|---|---:|---:|---:|
| 1 | **HistGradientBoosting** | **0.9057** | 0.0015 | 2.9s |
| 2 | CatBoost | 0.9051 | 0.0021 | 15.0s |
| 3 | XGBoost | 0.8911 | 0.0013 | 11.7s |
| 4 | DecisionTree | 0.8885 | 0.0039 | 1.7s |
| 5 | LightGBM | 0.8831 | 0.0022 | 13.4s |
| 6 | LogisticRegression | 0.8672 | 0.0004 | 2.7s |
| 7 | GaussianNB | 0.8405 | 0.0008 | 0.3s |
| 8 | RandomForest | 0.8386 | 0.0013 | 29.3s |
| 9 | ExtraTrees | 0.8364 | 0.0018 | 12.6s |
| 10 | LinearSVC | 0.8284 | 0.0038 | 0.9s |
| 11 | RidgeClassifier | 0.8166 | 0.0018 | 0.3s |
| 12 | KNN | 0.7869 | 0.0026 | 0.2s |

부스팅 계열이 상위권을 차지했고, 그중 **HistGradientBoosting**이 성능 1위이면서 학습 시간도 2.9초로 매우 빨랐다.

### 📷 시간 대비 성능 산점도

![모델별 시간 대비 성능](/images/ml/건강분류_모델별_시간대비성능.png)

왼쪽 위(빠르면서 정확)일수록 좋은데, HistGradientBoosting이 정확히 그 위치에 있다. CatBoost는 성능은 비슷하지만 학습이 5배 느리고, RandomForest는 29초나 걸리면서 성능은 오히려 낮다.

---

## 5️⃣ 최종 모델 선택 — 기준을 세우고 고른다

선택 기준을 우선순위대로 정했다.

1. **balanced_acc**(주지표) — 불균형 다중분류의 핵심
2. **안정성(std)** — fold별 편차가 작을수록 신뢰
3. **효율(fit_time)** — 비슷한 성능이면 빠른 쪽 (튜닝·재학습·서비스화에 유리)

이 기준을 종합하면 **HistGradientBoosting**이 성능 1위이면서 가장 빠른, 이견 없는 선택이었다.

## 튜닝

선택한 모델을 `RandomizedSearchCV`(n_iter=15, 3-fold)로 가볍게 튜닝했다.

```text
best CV balanced_acc = 0.9064
best params: {max_leaf_nodes: 31, max_iter: 500, learning_rate: 0.03, l2_regularization: 0}
```

기본값(0.9057) 대비 큰 폭은 아니었지만 소폭 개선됐다.

---

## 6️⃣ 최종 검증 — 혼동행렬로 클래스별 성능 확인

전체 train을 80/20으로 나눠 hold-out 검증한 혼동행렬이다.

### 📷 Confusion Matrix

![혼동행렬](/images/ml/건강분류_혼동행렬_HGB.png)

주목한 부분은 **소수 클래스의 recall**이었다. 불균형 데이터라 `at-risk`야 당연히 잘 맞히겠지만, `fit`과 `unhealthy`를 놓치지 않는 게 balanced accuracy의 핵심이다.

- `fit` recall ≈ 92% (7,333 / 7,961)
- `unhealthy` recall ≈ 94% (10,826 / 11,545)

소수 클래스도 90% 이상 잡아내고 있어서, `class_weight='balanced'` 전략이 제대로 작동했다는 걸 확인할 수 있었다.

---

## 7️⃣ 변수 중요도 — 예상이 빗나간 지점 🤔

마지막으로 선택 모델의 변수 중요도를 확인했는데, 여기서 예상과 다른 결과가 나왔다.

### 📷 변수 중요도 (Permutation Importance)

![변수 중요도](/images/ml/건강분류_변수중요도_HGB.png)

`stress_level_ord`, `sleep_duration`, `physical_activity_level_ord`, `bmi`가 상위권을 차지했다. 그런데 **1편에서 공들여 만든 복합 변수 `lifestyle_risk_score`는 오히려 최하위권**이었다.

이게 흥미로운 지점이었다. 여러 위험요인을 합쳐 만든 "그럴듯한" 변수보다, 단순한 서열 인코딩 하나(`stress_level_ord`)가 훨씬 강력했던 것이다.

> **핵심 정리**
>
> 강한 부스팅 모델은 개별 변수의 상호작용을 스스로 학습하기 때문에, 사람이 미리 합쳐 만든 복합 변수가 항상 유리한 건 아니다. `lifestyle_risk_score`는 로지스틱 같은 선형 모델을 도와주는 용도에 가까웠고, 트리 부스팅에선 이득이 작았다. **"만들었으면 검증한다"** 는 원칙이 있었기에 이 사실을 확인할 수 있었다.

---

# 📊 이 단계 정리

| 단계 | 내용 |
|------|------|
| 변수 가지치기 | 20개로 정리 (죽은 변수 제거) |
| 인코딩 | 순서형=서열 / 명목형=OneHot |
| 아키텍처 | ColumnTransformer + Pipeline |
| 모델 비교 | 12개, 3-Fold, balanced_acc + 시간 |
| 최종 선택 | HistGradientBoosting (성능 1위 + 최속) |
| 튜닝 | RandomizedSearch → CV 0.9064 |

---

## 📁 전체 코드

👉 [student_health_02_모델비교_모델선택.ipynb 보기](https://github.com/heo-aio/heo-aio.github.io/blob/main/static/notebooks/student_health_02_%EB%AA%A8%EB%8D%B8%EB%B9%84%EA%B5%90_%EB%AA%A8%EB%8D%B8%EC%84%A0%ED%83%9D.ipynb)

---

## 💡 공부하면서 느낀 점

이번 단계에서 가장 인상 깊었던 건 **"성능만 보고 모델을 고르지 않는다"** 는 관점이었다.

처음에는 그냥 balanced accuracy 1등을 고르면 되는 줄 알았는데, 안정성(std)과 학습 시간까지 함께 놓고 보니 "왜 이 모델인가"를 설명할 수 있게 됐다. 실제로 CatBoost와 HistGradientBoosting은 성능이 거의 같았지만, 학습 속도 차이(5배) 때문에 이후 튜닝과 실험 단계에서 HistGradientBoosting이 훨씬 유리했다.

그리고 변수 중요도에서 `lifestyle_risk_score`가 최하위로 나온 것은, 내 예상이 빗나간 순간이라 오히려 기억에 남는다. 공들여 만든 변수라도 데이터와 모델이 "아니다"라고 하면 받아들여야 한다는 걸 배웠다.

다음 편에서는 이 단일 모델을 넘어서 **여러 부스팅 모델을 앙상블하는 실험**을 정리한다.
