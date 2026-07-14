+++
title = "[프로젝트] 학생 건강상태 분류 (3/4) — 부스팅 앙상블 (OOF, Fold 평균)"
date = 2026-07-14T10:00:00+09:00
draft = false
tags = ["Python", "Machine Learning", "Kaggle", "Ensemble", "LightGBM", "XGBoost", "CatBoost", "팀프로젝트"]
categories = ["ml"]
+++

[2편](../student_health_02_model/)에서 단일 모델(HistGradientBoosting)을 골랐다. 이번 편에서는 **여러 부스팅 모델을 앙상블**해서 성능과 안정성이 더 좋아지는지 실험한 내용을 정리한다.

이 단계의 태도는 처음부터 분명했다.

> 앙상블을 **쓰는 것**이 목표가 아니라, 실제 검증 결과로 앙상블이 **도움이 되는지 확인**하는 것이 목표다.

그리고 결론부터 말하면, 이번 실험에서 **단순 평균 앙상블은 오히려 최고 단일 모델보다 낮았다.** 이게 이 편의 핵심 이야기다.

---

## 📌 이 단계의 흐름

```text
OOF 교차검증 엔진 구성

↓

변수 세트 진단 (PRUNED vs FULL)

↓

4종 부스팅 OOF 학습

↓

앙상블 (단순 평균 → 가중 평균)

↓

단일 모델과 비교 → 채택 여부 결정
```

이번 실험은 속도를 위해 전체의 30% 층화 샘플(약 20.7만 건)에 5-Fold OOF로 진행했다.

---

## 1️⃣ OOF(Out-Of-Fold) 교차검증

앙상블 실험의 핵심 엔진은 **OOF 예측**이다.

- 각 Fold에서, **학습에 사용하지 않은** 검증 부분을 예측한다 → 이걸 모으면 train 전체에 대한 "정직한" 예측(OOF)이 된다
- 동시에 각 Fold 모델로 **test를 예측**하고, Fold별 확률을 평균낸다 → 특정 분할에 치우치지 않은 test 예측

```python
def run_oof(name, features):
    oof = np.zeros((len(X), n_cls))
    test_proba = np.zeros((len(Xtest), n_cls))
    skf = StratifiedKFold(N_SPLITS, shuffle=True, random_state=RS)
    for tr, va in skf.split(X, y_run):
        m = fit_one(name, make_model(name), X.iloc[tr], y_run[tr], X.iloc[va], y_run[va], sw)
        oof[va] = m.predict_proba(X.iloc[va])         # 검증 부분 예측 → OOF
        test_proba += m.predict_proba(Xtest) / N_SPLITS  # test는 fold 평균
    return oof, test_proba
```

여기에 **Early Stopping**을 적용해, 트리 개수를 크게 잡아두고 검증 성능이 더 이상 좋아지지 않으면 자동으로 멈추게 했다. 네 모델(LGBM/XGB/CatBoost/HGB)의 early stopping API가 조금씩 달라서 분기 처리했다.

---

## 2️⃣ 변수 세트 진단 — PRUNED vs FULL ⭐

먼저 짚고 넘어갈 문제가 있었다. 2편의 단일 모델은 다른 설정(10만 샘플·3-Fold·가지치기 변수)에서 balanced accuracy 약 0.906였는데, 이 편에서 FULL 변수·5-Fold로 다시 돌리니 단일 모델도 훨씬 높게 나왔다.

> EDA 단계의 빠른 테스트에서는 약 **0.947**, 2편 모델링에서는 약 **0.906**. 이 차이가 **변수 가지치기 때문**인지, 아니면 데이터 분할·검증 방식의 차이인지 확인이 필요했다.

그래서 두 변수 세트를 완전히 같은 조건에서 OOF로 비교했다.

```python
for tag, feats in [('PRUNED', FEATURES_PRUNED), ('FULL', FEATURES_FULL)]:
    _, _, sc = run_oof('LightGBM', feats); diag[tag] = sc
```

결과는 이랬다.

```text
LightGBM  PRUNED  OOF = 0.9208
LightGBM  FULL    OOF = 0.9221
→ 채택: FULL (거의 차이 없음)
```

두 값이 거의 같았다. 즉 점수 하락은 **변수 가지치기 탓이 아니라**, 앞선 빠른 테스트가 소표본에서 낙관적으로 나왔던 것이 원인이었다. 이렇게 "점수가 왜 달라졌지?"를 감으로 추측하지 않고 **같은 조건에서 실험으로 원인을 판별**한 게 이 단계의 수확이었다.

---

## 3️⃣ 4종 부스팅 앙상블 — 단순 평균의 함정

변수 세트를 FULL로 정한 뒤, 4개 부스팅 모델을 각각 OOF로 학습하고 예측 확률을 평균냈다.

```python
oof_ens  = np.mean([oofs[n]  for n in ok], axis=0)   # OOF 확률 평균 → 점수 확인
test_ens = np.mean([tests[n] for n in ok], axis=0)   # test 확률 평균 → 제출용
```

### 📷 4종 모델 vs 앙상블 (OOF)

![앙상블 OOF 비교](/images/ml/건강분류_앙상블_OOF비교.png)

결과가 흥미로웠다. `CatBoost`(0.9485)와 `HistGB`(0.9480)는 강했지만, `LightGBM`(0.9221)과 `XGBoost`(0.9259)는 상대적으로 약했다. 그런데 이 넷을 **단순 평균**하니 앙상블 점수가 **0.9426** — 최고 단일 모델(CatBoost 0.9485)보다 **오히려 낮게** 나왔다.

이유는 명확했다. 단순 평균은 4개 모델에 똑같은 가중치를 주는데, 약한 두 모델(LGBM·XGB)이 강한 두 모델의 예측을 끌어내린 것이다.

> **핵심 정리**
>
> 앙상블은 무조건 이득이 아니다. 성능 격차가 큰 모델을 단순 평균하면 약한 모델이 발목을 잡아 오히려 손해를 볼 수 있다.

참고로 OOF 기준 클래스별 recall은 `at-risk` 0.95, `fit` 0.94, `unhealthy` 0.94로, 소수 클래스도 균형 있게 잡고는 있었다.

---

## 4️⃣ 가중 앙상블 — 강한 모델에 무게를

단순 평균의 문제를 확인했으니, 모델별로 다른 가중치를 주는 걸 시도했다. Dirichlet 분포로 합이 1인 랜덤 가중치를 여러 번 뽑아 OOF 점수가 가장 높은 조합을 찾았다.

```python
for _ in range(3000):
    w = rng.dirichlet(np.ones(len(ok)))          # 합=1 랜덤 가중치
    blend = np.tensordot(w, oof_stack, axes=(0,0))
    s = score_fn(y_run, blend.argmax(1))
    if s > best_s: best_s, best_w = s, w
```

찾은 가중치는 이랬다.

```text
LightGBM 0.005 | XGBoost 0.018 | CatBoost 0.531 | HistGB 0.446
→ 가중 앙상블 OOF = 0.9486  (단순 평균 대비 +0.0060)
```

탐색 결과가 사실상 **약한 두 모델을 거의 버리고(합 2%) 강한 두 모델(CatBoost + HistGB)에 무게를 몰아준** 것이다. 위 차트에서 가중 평균(맨 오른쪽)이 다시 단일 최고 수준으로 회복한 게 이 덕분이다.

> ⚠️ 다만 가중치를 OOF 점수만 보고 고르면 OOF에 과적합될 여지가 있다. 그래서 단순 평균과 차이가 아주 작으면 단순 평균을 믿는 게 안전하다고 판단했다.

---

## 📊 이 단계 정리

```text
OOF 엔진 (fold 평균 + early stopping)

↓

PRUNED vs FULL 진단 → 가지치기가 원인 아님을 확인

↓

4종 부스팅 OOF 학습 (강 2 · 약 2)

↓

단순 평균 = 0.9426 (단일 최고보다 낮음! ❌)

↓

가중 평균 = 0.9486 (강한 모델에 무게 → 회복 ✅)
```

---

## 📁 전체 코드

👉 [student_health_03_부스팅앙상블.ipynb 보기](https://github.com/heo-aio/heo-aio.github.io/blob/main/static/notebooks/student_health_03_%EB%B6%80%EC%8A%A4%ED%8C%85%EC%95%99%EC%83%81%EB%B8%94.ipynb)

---

## 💡 공부하면서 느낀 점

이번 단계에서 가장 크게 남은 건 **"앙상블도 검증 대상"** 이라는 감각이었다.

막연히 "여러 모델을 합치면 좋아진다"고 생각했는데, 실제로 단순 평균이 최고 단일 모델보다 낮게 나오는 걸 직접 보고 나니 생각이 바뀌었다. 성능 격차가 큰 모델을 그냥 평균내면 약한 모델이 결과를 갉아먹는다는 걸, 숫자로 확인했다.

또 PRUNED vs FULL을 같은 조건에서 비교해 "점수 차이의 원인"을 데이터로 판별한 과정도 기억에 남는다. 감으로 "변수 때문인가?" 추측하는 대신 실험으로 답을 찾는 방식은 앞으로도 계속 쓰게 될 것 같다.

다음 편에서는 마지막으로 **스태킹, 클래스 확률 보정 등 성능을 끝까지 끌어올리는 실험**을 정리한다.
