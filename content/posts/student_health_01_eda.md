+++
title = "[프로젝트] 학생 건강상태 분류 (1/4) — EDA & 파생변수 설계"
date = 2026-07-13T10:00:00+09:00
draft = false
tags = ["Python", "Machine Learning", "Kaggle", "EDA", "Feature Engineering", "통계검정", "팀프로젝트"]
categories = ["ml"]
math = true
+++

이어드림스쿨 팀 프로젝트로 Kaggle **Playground Series S6E7 — 학생 건강상태 분류** 대회를 7일간 진행했다. 나는 팀에서 **EDA·데이터 전처리와 모델링**을 담당했다.

이 시리즈는 그 과정을 4편으로 나눠 정리한 기록이다.

1. **(1/4) EDA & 파생변수 설계** ← 이 글
2. (2/4) 모델 비교 & 모델 선택
3. (3/4) 부스팅 앙상블
4. (4/4) 스태킹 & 최종 스퍼트

이번 글에서는 첫 단계인 EDA와 파생변수 설계를 다룬다.

> 🔗 대회 링크: [Kaggle Playground Series S6E7](https://www.kaggle.com/competitions/playground-series-s6e7/overview)

---

# 📌 이 노트북의 사고 틀

이번 EDA에서 스스로 세운 원칙은 하나였다.

> 좋은 모델은 좋은 알고리즘이 아니라 **좋은 이해**에서 나온다.

그래서 단순히 그래프를 그리고 끝내는 게 아니라, 아래 세 가지를 목표로 삼았다.

- 결측치가 **정보를 담고 있는지** 먼저 확인한다 (그냥 채우면 신호를 잃을 수 있으므로)
- 어떤 변수가 타깃을 **잘 구분하는지** 통계적으로 확인한다
- 파생변수는 만들었다고 끝이 아니라 **검증**을 통해 채택하거나 기각한다

데이터는 학생의 생활습관(수면, 심박수, BMI, 걸음수, 스트레스, 흡연·음주 등)으로 건강상태를 `fit / at-risk / unhealthy` 세 가지로 분류하는 문제였다. train 데이터는 약 **69만 건**(690,088행)이었다.

---

# 1️⃣ 타깃 분석 — 심한 불균형

가장 먼저 확인한 것은 타깃 분포였고, 여기서 이 프로젝트의 성격이 결정됐다.

### 📷 타깃 클래스 분포

![타깃 분포](/images/ml/건강분류_타깃분포.png)

`at-risk`가 **85.9%**, `unhealthy` 8.4%, `fit` 5.8%로 심한 불균형 데이터였다. 이게 왜 중요할까?

- 아무 생각 없이 전부 `at-risk`로 찍어도 **정확도(accuracy) 약 86%**가 나온다 → accuracy는 여기서 거의 무의미하다.
- 그래서 이 대회의 평가지표는 **balanced accuracy**(각 클래스 recall의 평균)와 **macro-F1**이다.
- 모델 학습 때도 소수 클래스(`fit`, `unhealthy`)를 놓치지 않도록 `class_weight='balanced'` 같은 장치가 필요하다.

> **핵심 정리**
>
> 불균형 데이터에서는 "정확도가 높다"가 함정이 될 수 있다. 지표부터 balanced accuracy로 잡고 시작해야 한다.

---

# 2️⃣ 결측치 분석 ⭐ — "결측도 신호일 수 있다"

이번 EDA에서 가장 공들인 부분이다. 결측치는 그냥 "채워야 할 빈칸"이 아니다. **결측 여부 자체가 정보**일 수 있다.

### 📷 변수별 결측률

![결측률](/images/ml/건강분류_결측률.png)

`stress_level`(12%), `sleep_duration`(11%)을 필두로 대부분 변수에 결측이 있었다. 여기서 결측을 무작정 채우기 전에, 결측의 성격을 먼저 나눠 생각했다.

- **MCAR**(완전 무작위 결측): 정말 랜덤 → 그냥 채우면 됨
- **MAR / MNAR**(조건부·비무작위 결측): 결측 여부가 타깃과 연관 → **결측 표시자(missing indicator)** 를 만들면 정보를 살릴 수 있음

그래서 각 변수마다 "값이 결측인 그룹"과 "값이 있는 그룹"의 **타깃 분포가 다른지**를 카이제곱 검정으로 확인했다.

```python
for c in feature_cols:
    if train[c].isnull().sum() == 0:
        continue
    is_miss = train[c].isnull()
    ct = pd.crosstab(is_miss, train[TARGET])
    chi2, p, _, _ = stats.chi2_contingency(ct)
    # p-value가 작으면(<0.05) "결측 여부와 타깃이 독립이 아니다" = 결측에 신호가 있다
```

`p < 0.05`로 나온 변수는 "결측이 정보를 담고 있다"는 뜻이므로, 뒤에서 `변수_isna` 형태의 결측 지표 변수를 만들 근거가 됐다.

---

# 3️⃣ 수치형 변수 — 타깃별 분포 & Kruskal-Wallis

수치형 변수는 "세 클래스 간 이 변수의 분포가 다른가?"를 확인했다. 먼저 타깃별 박스플롯으로 눈으로 보고, 통계 검정으로 뒷받침했다.

### 📷 타깃별 수치형 박스플롯

![타깃별 박스플롯](/images/ml/건강분류_수치형_타깃별박스플롯.png)

한눈에 봐도 `sleep_duration`이 타깃을 강하게 가른다. `fit`은 중앙값 약 8시간, `at-risk`는 7시간, `unhealthy`는 약 5.5시간으로 확연히 갈렸다. `bmi`는 `fit → unhealthy`로 갈수록 높아지고, `step_count`·`exercise_duration`은 `fit`에서 뚜렷이 높았다. 반면 `heart_rate`, `water_intake`는 세 클래스가 거의 겹쳐서 구분력이 약해 보였다.

이 눈짐작을 **Kruskal-Wallis 검정**으로 수치화했다. 여기서 ANOVA 대신 KW를 쓴 이유가 있다.

> ANOVA는 정규성·등분산 가정이 필요한데, KW 검정은 그 가정에 덜 민감한 **비모수 검정**이라 더 안전하다.

```python
for c in numeric_features:
    groups = [train.loc[train[TARGET]==g, c].dropna() for g in order]
    H, p = stats.kruskal(*groups)
    eta2 = (H - (len(order)-1)) / (n - len(order))   # 근사 효과크기
```

H 통계량이 크고 `p < 0.05`면 타깃을 잘 가르는 변수다. 여기에 효과크기(eta²)까지 같이 계산해서 "통계적으로 유의한가"뿐 아니라 "실질적으로 얼마나 큰가"도 함께 봤다.

---

# 4️⃣ 범주형 변수 — 타깃 비율 & Cramér's V

범주형 변수는 카테고리별 타깃 비율을 보고, 카이제곱 검정과 **Cramér's V**로 관계의 세기를 봤다.

### 📷 범주형 변수별 타깃 비율

![범주형 타깃 비율](/images/ml/건강분류_범주형_타깃별비율.png)

여기서도 명확한 대비가 보였다. `stress_level`이 `low`면 `fit` 비율이 20%까지 오르고 `unhealthy`가 거의 없지만, `high`면 `unhealthy` 비중이 크게 늘었다. `sleep_quality`, `physical_activity_level`도 비슷하게 갈랐다. 반대로 `diet_type`, `gender`는 카테고리별 비율이 거의 똑같아서 **구분력이 없어 보였다.**

이 관계를 Cramér's V로 수치화했는데, 여기서 중요한 포인트를 하나 배웠다.

> 표본이 크면(69만 건!) p-value는 거의 항상 유의하게 나온다. 그래서 p-value만 믿으면 안 되고, **효과크기(Cramér's V)** 를 함께 봐야 한다.

```python
def cramers_v(confusion):
    chi2 = stats.chi2_contingency(confusion)[0]
    n = confusion.sum().sum()
    r, k = confusion.shape
    phi2 = chi2 / n
    phi2corr = max(0, phi2 - (k-1)*(r-1)/(n-1))
    rcorr = r - (r-1)**2/(n-1)
    kcorr = k - (k-1)**2/(n-1)
    return np.sqrt(phi2corr / max(min(kcorr-1, rcorr-1), 1e-9))
```

Cramér's V는 0~1 값으로, 0.1↑ 약함 / 0.3↑ 중간 / 0.5↑ 강함으로 해석한다. `stress_level` 같은 변수가 높게 나왔다.

---

# 5️⃣ 파생변수 설계 ⭐⭐ — 각 변수에 '왜'가 있어야 한다

앞의 EDA와 도메인 지식을 근거로 파생변수를 만들었다. 여기서 스스로 지킨 원칙은 **"느낌으로 만들지 않는다, 각 변수에 근거가 있어야 한다"** 였다.

| 파생변수 | 근거 |
|---|---|
| 서열 인코딩 `*_ord` | stress/sleep_quality/activity/smoking은 **순서가 있는** 범주. OneHot보다 서열 숫자가 트리 모델에 유리하고 차원도 줄어듦 |
| `sleep_debt`, `sleep_ideal` | 건강 수면 기준 7~9시간. **부족분**과 **이상범위 여부**가 건강상태를 가르는 핵심 |
| `bmi_cat`, `bmi_abnormal` | WHO BMI 구간(저체중/정상/과체중/비만). '비정상 체중'이 위험 신호 |
| `hr_high` | 안정시 심박수 높음 = 위험 신호 |
| `steps_per_ex_min`, `cal_per_step` | 활동의 **효율·강도**. 단순 걸음수보다 조합이 생활패턴을 잘 표현 |
| **`lifestyle_risk_score`** ⭐ | 위험요인 개수 합산(수면부족·고스트레스·수면질나쁨·좌식·저활동·흡연음주·비정상BMI) |
| `n_missing`, `*_isna` | 4번에서 '결측이 정보'로 나온 변수의 결측 지표 |

특히 신경 쓴 부분은 **데이터 누수(leakage) 방지**였다.

```python
# train 기준으로 임계값(분위수)을 먼저 계산해 두고, test에도 동일하게 적용
HR_HI   = train['heart_rate'].quantile(0.75)
STEP_LO = train['step_count'].quantile(0.25)
WATER_LO= train['water_intake'].quantile(0.25)
```

> ⚠️ 파생변수는 **오직 피처만** 사용해서 만들고(타깃 사용 금지), train에서 정한 기준(중앙값·분위수)을 **test에도 똑같이** 적용해야 한다. 이걸 어기면 검증 점수가 실제보다 뻥튀기된다.

---

# 6️⃣ 파생변수 검증 — "만들었으면 정말 도움이 되는지 증명하자"

파생변수를 만든 것으로 끝내지 않고, 상호정보량(Mutual Information)으로 각 변수가 타깃과 얼마나 정보를 공유하는지 확인했다.

### 📷 상호정보량 (빨강 = 파생변수)

![상호정보량](/images/ml/건강분류_상호정보량.png)

`stress_level`, `sleep_duration` 같은 원본 변수가 상위였고, 내가 만든 파생변수 중 `sleep_debt`, `stress_level_ord`, `lifestyle_risk_score`, `sleep_ideal`도 상위권에 들었다. 반대로 결측 지표(`*_isna`)나 `n_missing`, `low_water`, `hr_high`는 MI가 거의 0이라, "만들었지만 도움 안 되는 변수"로 걸러졌다.

여기에 더해 리프트 테스트(원본만 vs 원본+파생을 HistGradientBoosting·LightGBM 두 모델로 교차검증 비교)로 실제 점수 상승도 확인했다.

> 두 모델 모두에서 오르면 그 상승이 특정 모델의 우연이 아니라 **진짜**라고 신뢰할 수 있다. 리프트가 미미하면 핵심 변수만 남기고 나머지는 과감히 버리기로 했다.

> 💡 한 가지 예고: MI에서 상위권이던 `lifestyle_risk_score`가, 다음 편에서 실제 최종 모델의 변수 중요도에서는 최하위로 나온다. MI(모델 독립적 지표)와 실제 모델이 쓰는 변수는 다를 수 있다는 걸 이때는 아직 몰랐다.

---

# 📊 이 단계 정리

```text
타깃 분석 (불균형 확인 → 지표를 balanced accuracy로)

↓

결측치 분석 (Chi-square로 '결측이 정보인지' 검정)

↓

수치형 EDA (박스플롯 + Kruskal-Wallis + 효과크기)

↓

범주형 EDA (타깃비율 + Chi-square + Cramér's V)

↓

파생변수 설계 (도메인 근거 + 누수 방지)

↓

파생변수 검증 (MI + 2모델 리프트 테스트)
```

---

# 📁 전체 코드

이 단계의 전체 노트북은 아래에서 확인할 수 있다. (통계 검정 표, 분포 그래프, MI 그래프 포함)

👉 [student_health_01_EDA_파생변수설계.ipynb 보기](https://github.com/heo-aio/heo-aio.github.io/blob/main/static/notebooks/student_health_01_EDA_%ED%8C%8C%EC%83%9D%EB%B3%80%EC%88%98%EC%84%A4%EA%B3%84.ipynb)

---

# 💡 공부하면서 느낀 점

이번 EDA에서 가장 크게 배운 건 **"통계 검정이 EDA의 언어가 된다"** 는 점이었다.

예전에는 EDA라고 하면 그래프를 그려놓고 "음, 관련 있어 보이네" 하고 넘어갔다. 그런데 이번에는 결측이 정보인지 카이제곱으로 검정하고, 변수의 구분력을 Kruskal-Wallis와 Cramér's V로 수치화하니, 파생변수를 만들 때 "왜 이걸 만들었는가"를 근거를 갖고 설명할 수 있었다.

특히 표본이 크면 p-value가 항상 유의하게 나온다는 것, 그래서 효과크기를 함께 봐야 한다는 것은 실제로 69만 건 데이터를 만지면서 몸으로 이해한 부분이다.

다음 편에서는 이렇게 만든 변수들을 가지고 **12개 모델을 비교하고 최종 모델을 선택하는 과정**을 정리한다.
