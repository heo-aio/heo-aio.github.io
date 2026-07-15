+++
title = "[개념정리] SHAP과 Optuna — 감으로 하던 걸 근거로 바꾸기"
date = 2026-07-15T10:00:00+09:00
draft = false
tags = ["Python", "Machine Learning", "SHAP", "Optuna", "모델해석", "하이퍼파라미터튜닝"]
categories = ["ml"]
math = true
+++

지난 앙상블 정리 글 마지막에 다음 공부거리로 SHAP이랑 Optuna를 적어뒀었다. 미뤄두면 결국 안 볼 것 같아서 이번 주에 바로 붙잡았다.

계기는 사실 단순했다. LightGBM 돌려서 성능표 하나 뽑아놓고 뿌듯해하다가, "근데 왜 이 사람을 unhealthy로 예측한 거야?"라는 질문에 말문이 막혔다. AUC 몇 나왔다는 건 말할 수 있는데, 그 안에서 무슨 일이 일어났는지는 설명을 못 하고 있었다. 그리고 하이퍼파라미터는 여전히 `learning_rate=0.05` 이런 식으로 어디서 본 값을 그냥 갖다 쓰고 있었다. 이 두 개를 이번에 정리하면서 좀 뜯어보기로 했다.

## 모델이 "왜" 이렇게 예측했는지 — SHAP

SHAP은 게임이론에서 나온 Shapley Value라는 개념을 모델 예측에 갖다 쓴 거다. 처음 들었을 때는 이름부터 부담스러웠는데, 풀어보면 아이디어 자체는 단순하다.

> 여러 명이 같이 일해서 얻은 성과를, 각자 기여한 만큼 공평하게 나눠 갖는 방법.

이걸 모델에 적용하면, "이 예측값이 나오기까지 각 feature가 얼마나 기여했는가"를 숫자로 쪼개준다. 그냥 feature importance랑 뭐가 다른가 싶었는데, 차이가 있었다. 기존 feature importance는 "이 변수가 전체적으로 얼마나 중요한가"만 알려주는 반면, SHAP은 **샘플 하나하나에 대해** "이번 예측에서는 이 변수가 이만큼 예측값을 밀어올렸다/끌어내렸다"까지 알려준다. 그래서 개별 케이스를 설명할 때 훨씬 쓸모가 있다.

트리 기반 모델(RandomForest, XGBoost, LightGBM, CatBoost)은 `TreeExplainer`를 쓰면 되고, 계산도 다른 모델용 Explainer보다 훨씬 빠르다.

```python
import shap

explainer = shap.TreeExplainer(lgbm)
sv = explainer(X_test)          # shap.Explanation 객체로 반환됨

# 전체적으로 어떤 변수가 중요한지
shap.summary_plot(sv, X_test)

# 샘플 하나(0번)에 대해서 왜 이렇게 예측했는지
shap.plots.waterfall(sv[0])
```

여기서 좀 헤맨 부분이 있었는데, `shap_values`가 라이브러리 버전에 따라 배열 하나로 나오기도 하고 클래스별 리스트로 나오기도 한다. 예전 튜토리얼 코드 그대로 따라 하면 `sv[:, :, 1]`이 필요한지 그냥 `sv`를 쓰면 되는지 버전마다 달라서 한참 검색했다. 지금 버전 기준으로는 위처럼 `shap.Explainer`로 한 번 감싸서 쓰는 API가 정석인 것 같다.

`summary_plot`을 처음 보고 좋았던 점은, 변수 하나하나가 값이 높을 때(빨강)와 낮을 때(파랑) 예측에 어느 방향으로 영향을 주는지까지 한 그림에 다 나온다는 거였다. 예를 들어 수면시간이 짧을수록(파랑) unhealthy 쪽으로 밀어붙이는 게 점 색깔로 바로 보인다. 표로 봤을 때는 안 느껴지던 감각이 그림 하나로 확 와닿았다.

주의할 점 하나는 속도다. 데이터가 크면 `TreeExplainer`도 느려질 수 있어서, 전체 test set 말고 `shap.sample(X_test, 500)`처럼 일부만 뽑아서 보는 걸 권장한다고 한다. 나는 아직 데이터가 크지 않아서 전체로 돌렸는데, 나중에 프로젝트에 적용할 때는 이 부분을 신경 써야 할 것 같다.

## 하이퍼파라미터, 이제 감으로 안 하기 — Optuna

지금까지 하이퍼파라미터는 대충 예제 코드에서 본 값 넣고, 안되면 몇 개 바꿔보고 하는 식이었다. GridSearch를 써본 적은 있는데, 조합 몇 개만 늘어나도 학습 시간이 감당이 안 됐다. `max_depth` 5개, `learning_rate` 5개, `subsample` 5개만 해도 벌써 125번을 다 돌려봐야 한다.

Optuna는 이 문제를 다르게 푼다. 미리 모든 조합을 정해놓고 다 돌리는 게 아니라, **이전 시도 결과를 보고 다음에 어디를 시도할지 스스로 정한다**(TPE라는 방식을 기본으로 쓴다고 한다). 그러니까 성능이 안 좋았던 영역은 점점 덜 보고, 좋았던 영역 근처를 더 집중적으로 탐색하는 식이다. 같은 시도 횟수라도 GridSearch보다 훨씬 효율적으로 좋은 값을 찾는다는 게 체감이 됐다.

```python
import optuna
from sklearn.model_selection import cross_val_score

def objective(trial):
    params = {
        "n_estimators": trial.suggest_int("n_estimators", 100, 800),
        "learning_rate": trial.suggest_float("learning_rate", 0.01, 0.3, log=True),
        "max_depth": trial.suggest_int("max_depth", 3, 10),
        "num_leaves": trial.suggest_int("num_leaves", 15, 255),
        "subsample": trial.suggest_float("subsample", 0.5, 1.0),
        "colsample_bytree": trial.suggest_float("colsample_bytree", 0.5, 1.0),
    }
    model = LGBMClassifier(**params, random_state=42)
    score = cross_val_score(model, X_train, y_train, cv=5, scoring="roc_auc").mean()
    return score

study = optuna.create_study(direction="maximize")
study.optimize(objective, n_trials=50)

print(study.best_params)
print(study.best_value)
```

`objective` 함수 안에서 매번 `trial.suggest_xxx`로 값을 하나씩 뽑고, 그 조합으로 교차검증 점수를 돌려주기만 하면 나머지는 Optuna가 알아서 한다. `learning_rate`에 `log=True`를 넣은 이유는, 이 값은 0.01과 0.03 차이가 0.2와 0.22 차이보다 실질적으로 더 크게 작동해서 로그 스케일로 탐색하는 게 맞다고 한다 — 처음엔 그냥 넣었다가 왜 넣는지 찾아보고 이해했다.

n_trials를 50, 100처럼 늘릴수록 좋은 조합을 찾을 확률은 올라가지만 그만큼 시간이 걸린다. 이걸 줄이는 방법으로 **pruning**(가망 없는 시도를 중간에 끊어버리는 기능)이 있다는데, 이건 아직 직접 써보진 못했다. LightGBM용 콜백은 최근 버전부터 `optuna` 본체가 아니라 `optuna_integration`이라는 별도 패키지로 분리됐다는 것도 설치하면서 알았다. 다음에 실제로 붙여볼 때 정리해야겠다.

끝나고 나서는 `optuna.visualization.plot_optimization_history(study)`로 시도할수록 점수가 어떻게 올라갔는지, `plot_param_importances(study)`로 어떤 하이퍼파라미터가 결과에 제일 큰 영향을 줬는지도 바로 확인할 수 있었다. `max_depth`보다 `learning_rate` 쪽이 훨씬 영향이 크게 나와서 조금 의외였다.

## 정리하면서 든 생각

SHAP은 "이 모델을 믿어도 되는가"에 답하는 도구고, Optuna는 "이 모델이 낼 수 있는 최선을 뽑아냈는가"에 답하는 도구라는 생각이 든다. 둘 다 이번에 처음 제대로 써봤는데, 성능 숫자 하나 뽑는 것보다 훨씬 품이 많이 든다는 걸 느꼈다. 대신 그만큼 "왜 이렇게 나왔는지" 스스로 설명할 수 있는 상태가 됐다.

다음은 이 둘을 지난번 앙상블 글에서 정리한 RandomForest·부스팅 3종·Stacking에 실제로 붙여서, 학생 건강상태 분류 프로젝트 데이터로 직접 돌려볼 차례다. Optuna로 각 모델을 제대로 튜닝한 다음 SHAP으로 뜯어보면, 지난 EDA 글에서 상호정보량 기준으로 걸러냈던 파생변수들이 실제 모델 안에서는 어떻게 쓰이는지도 비교해볼 수 있을 것 같다.
