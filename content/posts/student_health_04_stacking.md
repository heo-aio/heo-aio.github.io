+++
title = "[프로젝트] 학생 건강상태 분류 (4/4) — 스태킹 & 최종 스퍼트"
date = 2026-07-14T14:00:00+09:00
draft = false
tags = ["Python", "Machine Learning", "Kaggle", "Stacking", "Optuna", "후처리", "팀프로젝트"]
categories = ["ml"]
+++

시리즈의 마지막 편이다. [3편](../student_health_03_ensemble/)에서 가중 앙상블로 balanced accuracy 약 **0.949**까지 왔고, 이번 편에서는 목표인 **0.952** 이상을 향해 성능을 끝까지 끌어올리는 실험들을 정리한다.

이 단계의 원칙은 3편과 같다.

> 새 모델을 하나 더 얹기보다, 개선 방법을 **하나씩 적용하고 효과를 검증**한다. 이득이 노이즈 수준이면 미적용한다.

---

## 📌 준비한 개선 방법 메뉴

| 실험 | 확인할 내용 |
|---|---|
| 원본 데이터 결합 | 대회 데이터의 원본 데이터셋을 학습에 추가하면 나아지는가 |
| 스태킹 | OOF 예측을 메타모델 입력으로 써서 단순 평균보다 나은가 |
| Optuna 튜닝 | 이전 결과를 참고해 파라미터를 효율적으로 탐색 |
| Seed 평균 | 여러 random seed 결과를 평균내 변동을 줄임 |
| Fold 수 증가 | 10-Fold에서 성능이 달라지는가 |
| 클래스 확률 보정 | 후처리로 클래스별 예측 비율을 조정 |

이 여섯 가지를 모두 코드로 구현해두고, 실행 설정만 바꿔가며 단계적으로 켤 수 있게 만들었다. 아래 결과는 그중 **스태킹·가중 평균·확률 보정**을 30% 샘플·5-Fold·단일 seed 설정에서 실제로 비교한 것이다. (원본 결합·Optuna·다중 seed·10-Fold는 최종 풀파워 실행용 옵션으로 준비만 해두었다.)

---

## 1️⃣ 캐싱 — 반복 실험의 근본 문제 해결

여러 앙상블·스태킹 조합을 실험하려면 모델을 계속 다시 학습해야 하는데, 부스팅 4종을 여러 번 돌리면 시간이 엄청나게 걸린다. 그래서 **캐싱**을 넣었다.

```python
key = hashlib.md5(json.dumps(dict(n=name, f=features, s=seed, k=N_SPLITS,
      fr=RUN_FRACTION, ...), sort_keys=True).encode()).hexdigest()[:12]
fpath = CACHE + f'{name}_s{seed}_{key}.npz'
if os.path.exists(fpath):          # 캐시 히트: 재학습 없이 즉시 로드
    z = np.load(fpath); return z['oof'], z['test'], score_fn(y_run, z['oof'].argmax(1))
```

(모델, seed, fold수, 변수, 파라미터)가 같으면 저장된 확률을 즉시 불러온다. 덕분에 가중치·스태킹·보정 실험을 **재학습 없이 몇 초 만에** 반복할 수 있었다. 실험을 많이 돌려야 하는 단계에서 이게 정말 큰 차이를 만들었다.

---

## 2️⃣ 원본 데이터 결합 — 누수 없이 (설계)

Playground 대회 데이터는 보통 어떤 원본 데이터셋을 바탕으로 합성된다. 그 원본을 학습에 추가하면 도움이 될 수 있는데, 여기서 **데이터 누수 방지**가 핵심이다.

```python
for tr, va in skf.split(X, y_run):
    Xtr, ytr = X.iloc[tr], y_run[tr]
    if orig_fe is not None:               # 원본은 학습 부분에만 추가
        Xtr = pd.concat([Xtr, orig_fe[features]])
        ytr = np.concatenate([ytr, y_orig])
    # 검증(va)에는 대회 데이터만 → OOF 점수도 대회 데이터로만 계산
```

- Fold의 **학습 부분**: 대회 데이터 + 원본 데이터
- Fold의 **검증 부분**: 대회 데이터만
- OOF 점수: 대회 검증 데이터로만 계산

이렇게 해야 검증 데이터에 원본 정보가 섞이지 않아, "대회 데이터만으로 평가했을 때"의 진짜 성능을 볼 수 있다. (이번 실행에서는 원본 슬러그를 지정하지 않아 이 경로는 건너뛰었고, 구조만 갖춰두었다.)

---

## 3️⃣ Optuna 튜닝 — RandomSearch보다 똑똑하게 (설계)

2편에서는 RandomizedSearch로 무작위 탐색을 했다. 최종 단계용으로는 **Optuna**를 준비했다.

> RandomizedSearch는 정해진 범위에서 후보를 **무작위로** 뽑는다. Optuna는 **이전 실험 결과를 참고해** 성능이 좋을 가능성이 높은 조합을 다음 후보로 선택한다(TPE 샘플러).

```python
st = optuna.create_study(direction='maximize',
                         sampler=optuna.samplers.TPESampler(seed=RS))
st.optimize(objective, n_trials=OPTUNA_TRIALS)
```

찾은 파라미터는 파일로 저장해서 이후 OOF 엔진에 그대로 주입할 수 있게 했다. (GPU가 필요해 이번 검증 실행에서는 끄고 진행했다.)

---

## 4️⃣ 앙상블 방식 비교 — 단순 vs 가중 vs 스태킹

핵심 실험은 세 가지 앙상블 방식을 같은 OOF로 비교한 것이다.

- **단순 평균**: 4개 모델 확률을 같은 비율로 평균
- **가중 평균**: 성능 참고해 서로 다른 가중치 (Dirichlet 랜덤서치)
- **스태킹**: 4개 모델의 OOF 확률을 **입력으로 삼아 로지스틱 회귀 메타모델**을 학습

스태킹은 누수 방지가 중요해서 `cross_val_predict`로 메타모델을 별도 교차검증했다.

```python
meta_X = np.hstack([runs[k][0] for k in keys])     # 각 모델 OOF 확률을 특징으로
meta = LogisticRegression(max_iter=2000, class_weight='balanced')
oof_meta_pred = cross_val_predict(meta, meta_X, y_run,
                    cv=StratifiedKFold(5, shuffle=True, random_state=RS),
                    method='predict_proba', n_jobs=-1)
```

결과는 이랬다.

```text
① 단순 평균  OOF = 0.94262
③ 스태킹     OOF = 0.94817
② 가중 평균  OOF = 0.94867   ← 채택
```

3편에서 본 대로 단순 평균이 가장 낮았고, 스태킹과 가중 평균이 비슷하게 높았다. 최종적으로 가중 평균을 채택했다.

---

## 5️⃣ 클래스 확률 보정 — 마지막 후처리 ⭐

마지막으로 적용한 건 **클래스별 확률 보정(post-processing)** 이었다. 이게 왜 필요했는지가 흥미롭다.

OOF 예측을 뜯어보니, `at-risk`는 안정적으로 맞혔지만 `fit`과 `unhealthy`는 **경계에 있는 샘플**이 서로 헷갈렸다. 예측 확률 차이가 크지 않은 샘플이 `argmax`에서 다른 클래스로 넘어가면서 소수 클래스 recall이 깎이는 현상이었다.

balanced accuracy는 세 클래스 recall을 똑같은 비중으로 평균내기 때문에, 이 경계를 살짝 조정하는 것만으로 점수가 오를 수 있다.

```python
def tune_multipliers(proba, y, iters=5000):
    best = np.ones(proba.shape[1]); best_s = score_fn(y, proba.argmax(1))
    for _ in range(iters):
        m = np.exp(rng.normal(0, 0.15, proba.shape[1]))   # 1.0 근처 랜덤 배수
        s = score_fn(y, (proba * m).argmax(1))            # 클래스 확률에 배수 적용
        if s > best_s: best_s, best = s, m
    return best, best_s
```

OOF에서 찾은 배수는 이랬다.

```text
클래스 배수: at-risk 0.65 | fit 0.996 | unhealthy 1.305
보정 전 0.94867 → 보정 후 0.94934 (+0.00067)
```

다수 클래스(`at-risk`)의 확률을 눌러(×0.65) 소수 클래스가 이길 여지를 주고, `unhealthy`를 키운(×1.305) 것이다. 모델을 다시 학습하는 게 아니라 최종 예측 단계에서 결정 경계를 미세 조정한 것이다.

```python
USE_CAL = (cal_s - base_s) > 0.0003   # 이득이 노이즈 수준이면 미적용 (OOF 과적합 방지)
```

이득(+0.00067)이 임계값(0.0003)을 넘어 적용했다.

### 📷 앙상블 방식 비교와 최종 후처리

![앙상블 방식과 최종](/images/ml/건강분류_앙상블방식_최종.png)

최종 기대 성능은 OOF 기준 약 **0.9493**으로, 목표인 0.952에는 아주 조금 못 미쳤다. 다만 이 실행은 30% 샘플·단일 seed·튜닝 없이 나온 값이고, 준비해둔 풀파워 옵션(전체 데이터·다중 seed·10-Fold·Optuna·원본 결합)을 켜면 더 끌어올릴 여지가 남아 있었다.

---

## 📊 프로젝트 전체 성능 흐름

```text
단일 모델 (2편, 다른 설정)        ≈ 0.906
        ↓ FULL 변수 · 5-Fold OOF (3편)
단일 최고 (CatBoost)              ≈ 0.9485
        ↓ 가중 앙상블
가중 평균                          ≈ 0.9487
        ↓ 클래스 확률 보정
최종 (가중 + 보정)                ≈ 0.9493   |  목표 0.952
```

> 📝 위 수치는 노트북에 기록된 OOF·검증 기준 값이다. 단계마다 "검증으로 확인하고, 이득이 있을 때만 채택"하는 방식으로 조금씩 끌어올렸다.

---

## 📁 전체 코드

👉 [student_health_04_스태킹_최종.ipynb 보기](https://github.com/heo-aio/heo-aio.github.io/blob/main/static/notebooks/student_health_04_%EC%8A%A4%ED%83%9C%ED%82%B9_%EC%B5%9C%EC%A2%85.ipynb)

---

## 💡 프로젝트를 마치며

7일간의 프로젝트에서 EDA·전처리와 모델링을 담당하며 가장 크게 배운 것은, **머신러닝이 "좋은 모델을 고르는 일"이 아니라 "매 단계 근거를 갖고 판단하는 일"** 이라는 점이었다.

결측이 정보인지 검정하고, 파생변수를 만들면 검증하고, 모델은 기준을 세워 고르고, 앙상블·스태킹·보정도 "정말 효과가 있는지" 데이터로 확인한 뒤에 채택했다. 특히 이득이 노이즈 수준이면 과감히 버린다는 원칙(`> 0.0003` 같은 임계값)을 반복하면서, "점수를 올리는 것"과 "재현 가능한 점수를 만드는 것"은 다르다는 걸 이해했다.

한편 아쉬운 점도 있다. 목표(0.952)에 약간 못 미친 채로 마무리됐고, 준비해둔 풀파워 옵션(전체 데이터·Optuna·다중 seed·원본 결합)을 시간상 충분히 돌려보지 못했다. 캐싱·GPU를 더 일찍 붙였다면 실험을 더 많이 돌려볼 수 있었을 것이다. 이런 부분은 다음 프로젝트에서 더 다뤄보고 싶다.

이 데이터가 실제 서비스라면, 생활습관 데이터로 건강 위험군을 조기에 선별해 맞춤 코칭으로 연결하는 서비스로 확장할 수 있을 것이다. 다만 합성 데이터일 가능성이 있으므로, 실제 적용 전에는 임상 데이터 검증이 반드시 필요하다.

긴 글 여기까지 읽어주신 분들께 감사를 ... 🙏
