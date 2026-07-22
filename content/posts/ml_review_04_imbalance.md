+++
title = "[ML 복습] 데이터 불균형 처리 — 언더샘플링과 오버샘플링(SMOTE)"
date = 2026-07-22T16:00:00+09:00
draft = false
tags = ["Python", "Machine Learning", "imbalanced-learn", "SMOTE", "데이터전처리", "복습"]
categories = ["ml"]
+++

복습 마지막 편은 실무에서 자주 부딪히는 **데이터 불균형(Data Imbalance)** 문제다. 앞서 정리한 학생 건강상태 분류 프로젝트에서도 `at-risk`가 86%를 차지하는 불균형 데이터라 애를 먹었던 부분이라, 개념을 제대로 다시 정리해둔다.

---

## 📌 데이터 불균형이 왜 문제인가

타깃 클래스의 비율이 크게 치우친 상태를 데이터 불균형이라고 한다. 희귀 질환 진단(정상 99% vs 환자 1%)이 대표적이다.

문제는 모델이 **수가 많은 클래스만 맞추도록 학습**된다는 것이다. 전부 "정상"이라고 찍어도 정확도 99%가 나오니, 정작 중요한 소수 클래스(환자)를 놓친다. 그래서 데이터 자체의 비율을 맞춰주는 작업이 필요하다.

크게 두 방향이다. 다수 클래스를 줄이는 **언더샘플링**, 소수 클래스를 늘리는 **오버샘플링**.

![샘플링 비교](/images/ml/review_샘플링.png)

위 그림이 핵심을 다 담고 있다. 왼쪽 원본은 파란 다수(539개) 대 빨간 소수(61개)로 심하게 치우쳐 있다. 가운데 언더샘플링은 다수를 61개로 **줄여** 맞췄고, 오른쪽 SMOTE는 소수를 539개로 **합성해 늘려** 맞췄다. 같은 균형이지만 접근이 정반대다.

---

## 🔽 언더샘플링 — 다수를 줄이기

다수 클래스 데이터를 삭제해 소수 클래스와 수를 맞춘다. 데이터가 줄어 학습이 빨라지지만, 다수 클래스의 유용한 정보가 사라질 수 있다. **전체 데이터가 충분히 많을 때** 주로 쓴다.

- **Random**: 다수 클래스에서 무작위로 골라 삭제. 단순하지만 중요한 데이터도 날아갈 수 있다.
- **Tomek Links**: 서로 다른 클래스면서 가장 가까이 붙어 있는 한 쌍을 찾아, 그중 다수 클래스 쪽을 삭제한다. 경계가 애매한 다수 데이터를 걷어내 **분류 경계를 또렷하게** 만든다.
- **CNN (Condensed Nearest Neighbor)**: 경계 근처 핵심 샘플만 남겨 전체 구조를 보존한다.

```python
from imblearn.under_sampling import (RandomUnderSampler,
                                     TomekLinks, CondensedNearestNeighbour)

X_rus, y_rus     = RandomUnderSampler(random_state=42).fit_resample(X, y)
X_tomek, y_tomek = TomekLinks().fit_resample(X, y)
X_cnn, y_cnn     = CondensedNearestNeighbour(random_state=42).fit_resample(X, y)
```

---

## 🔼 오버샘플링 — 소수를 늘리기

소수 클래스를 복제하거나 합성해 다수와 수를 맞춘다. 원본 정보를 하나도 안 버린다는 게 장점이지만, 과적합 위험이 커진다. **데이터가 절대적으로 부족할 때** 주로 쓴다.

- **Random**: 소수 데이터를 그대로 복제. 쉽지만 같은 데이터를 반복하니 과적합 위험이 크다.
- **SMOTE**: 소수 데이터에 KNN을 적용해, 가까운 이웃 사이 선분 위에 **새로운 합성 데이터**를 만든다. 단순 복제가 아니라서 과적합을 상당히 완화한다.
- **Borderline SMOTE**: 소수 데이터 중에서도 **다수와 맞닿은 경계 영역**에 집중해 SMOTE를 적용한다. 결정 경계 부근 식별력을 높이지만, 경계 노이즈까지 키울 위험이 있다.

```python
from imblearn.over_sampling import RandomOverSampler, SMOTE, BorderlineSMOTE

X_ros, y_ros       = RandomOverSampler(random_state=42).fit_resample(X, y)
X_smote, y_smote   = SMOTE(k_neighbors=4, random_state=42).fit_resample(X, y)
X_bsmote, y_bsmote = BorderlineSMOTE(k_neighbors=4, random_state=42).fit_resample(X, y)
```

> **복습 포인트**
>
> 위 그림 오른쪽에서 빨간 점이 기존 소수 데이터들 **사이사이에 채워진** 걸 볼 수 있다. 이게 SMOTE가 "복제"가 아니라 "이웃 사이 보간으로 합성"한다는 증거다. 단, 합성이다 보니 실제 분포와 살짝 다를 수 있다는 것도 같이 기억해둔다.

---

## 📝 핵심 요약

| 구분 | 대표 기법 | 특징 | 권장 상황 |
|---|---|---|---|
| **언더샘플링** | `RandomUnderSampler` | 다수 무작위 제거 | 데이터가 충분히 많을 때 |
| | `TomekLinks` | 경계 다수 제거 → 경계 명확화 | 경계를 깔끔히 정리하고 싶을 때 |
| | `CondensedNearestNeighbour` | 구조 보존하며 경계 샘플 선택 | 정보 손실 최소화 |
| **오버샘플링** | `RandomOverSampler` | 소수 무작위 복제 | 단순하나 과적합 위험 |
| | `SMOTE` | KNN 보간으로 합성 | 정보 손실 없이 과적합 방지 |
| | `BorderlineSMOTE` | 경계 소수에 집중 합성 | 경계 식별력 향상 |

---

## 💡 정리하며 — 그리고 주의할 점

한 가지 실전에서 꼭 지켜야 할 게 있다. **샘플링은 반드시 train 데이터에만, 그것도 train/test를 나눈 뒤에 적용해야 한다.** 전체 데이터에 SMOTE를 먼저 걸고 나누면, 합성된 데이터가 test에 새어 들어가(데이터 누수) 검증 점수가 실제보다 부풀려진다. 교차검증을 쓴다면 `imblearn`의 `Pipeline`으로 각 fold 안에서 샘플링이 일어나게 묶는 게 안전하다.

이번 4편짜리 복습으로 지도학습부터 불균형 처리까지 기본기를 한 바퀴 돌았다. 개념을 눈으로 보고 코드로 손에 붙이는 걸 목표로 했는데, 다음은 이 코드들을 백지에서 다시 짜보는 연습으로 이어갈 생각이다.
