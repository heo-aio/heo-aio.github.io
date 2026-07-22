+++
title = "[ML 복습] 비지도학습 — 클러스터링과 차원 축소"
date = 2026-07-22T12:00:00+09:00
draft = false
tags = ["Python", "Machine Learning", "Scikit-learn", "비지도학습", "클러스터링", "복습"]
categories = ["ml"]
math = true
+++

복습 두 번째 편은 **비지도학습(Unsupervised Learning)** 이다. 지도학습과 결정적으로 다른 점은 **정답(Label)이 없다**는 것이다.

정답이 없으니 "맞췄다/틀렸다"로 채점할 수 없다. 대신 데이터 자체의 구조를 파악한다. 크게 비슷한 것끼리 묶는 **클러스터링**과, 특성 수를 줄이는 **차원 축소**로 나뉜다.

---

## 🔵 클러스터링 — 정답 없이 끼리끼리 모으기

클러스터링은 그룹 정보가 없는 상태에서 **유사한 데이터끼리 군집으로 묶는** 작업이다. 목표는 간단하다. 군집끼리는 서로 멀리 떨어뜨리고, 같은 군집 안 데이터는 가깝게 뭉치게 하는 것.

### K-Means

가장 대표적인 알고리즘이다. 사용자가 군집 수 K를 정하면, 중심점을 잡고 옮기는 과정을 반복하며 데이터를 K개로 나눈다.

![K-Means 군집과 엘보우](/images/ml/review_kmeans.png)

왼쪽이 K=4로 나눈 결과다. X 표시가 각 군집의 중심점이다. 그런데 K를 몇으로 줘야 할까? 이때 쓰는 게 오른쪽 **엘보우 방법**이다. K를 늘려가며 군집 내 오차(inertia)를 그려보면, 꺾이는 지점(팔꿈치)이 적정 K다. 여기선 4에서 확 꺾인다.

```python
from sklearn.cluster import KMeans

model = KMeans(n_clusters=4, random_state=100)
model.fit(X)

labels = model.labels_              # 각 데이터가 속한 군집 번호
centroids = model.cluster_centers_  # 군집 중심점 좌표
```

기억해둘 점이 몇 개 있다. K-Means는 **거리 기반**이라 스케일링이 필수고, 평균으로 중심을 잡기 때문에 **이상치에 민감**하다. 초기 중심점 위치에 따라 결과가 흔들려서 여러 번 돌려 안정적인 결과를 고른다.

### GMM (Gaussian Mixture Model)

K-Means가 "너는 A 그룹"이라고 딱 잘라 정하는 하드 클러스터링이라면, GMM은 "너는 A일 확률 80%, B일 확률 20%"처럼 확률로 유연하게 표현하는 **소프트 클러스터링**이다. 데이터가 여러 정규분포가 섞여 만들어졌다고 가정한다.

```python
from sklearn.mixture import GaussianMixture

model = GaussianMixture(n_components=3, random_state=100)
model.fit(X)
labels = model.predict(X)
```

확률 분포를 계산하느라 무거워서 대용량엔 부담이 크고, 이상치에 취약한 건 K-Means와 비슷하다.

### 클러스터링은 어떻게 평가할까

정답이 없으니 오차율을 못 낸다. 대신 "군집끼리 얼마나 멀고, 군집 안은 얼마나 뭉쳤나"로 간접 평가한다.

- **던 인덱스(Dunn Index)**: 군집 간 최소 거리 ÷ 군집 내 최대 거리. **클수록** 좋다.

$$\text{Dunn Index} = \frac{\text{군집 간 거리의 최솟값}}{\text{군집 내 요소 간 거리의 최댓값}}$$

- **실루엣 계수(Silhouette)**: -1 ~ 1 값이고, **1에 가까울수록** 잘 나뉜 것이다.

---

## 🔻 차원 축소 — 핵심만 남기고 덜어내기

여기서 '차원'은 곧 **컬럼(특성)의 개수**다. 특성이 수천, 수만 개면 학습이 느려지고, 데이터가 부족하면 노이즈까지 외워 과적합이 생긴다. 이걸 **고차원의 저주**라 부른다. 차원 축소는 정보를 최대한 지키면서 특성 수를 줄이는 작업이다.

### PCA vs t-SNE

두 기법을 같은 데이터(64차원 손글씨 숫자)에 적용해 2차원으로 줄여봤다.

![PCA와 t-SNE 비교](/images/ml/review_차원축소.png)

**PCA**는 데이터가 가장 넓게 퍼진(분산이 큰) 방향으로 축을 잡아 차원을 줄이는 **선형** 기법이다. 빠르지만, 왼쪽 그림처럼 숫자들이 서로 좀 겹쳐 보인다.

**t-SNE**는 가까운 점들의 거리를 저차원에서도 유지하려는 **비선형** 기법이다. 오른쪽처럼 같은 숫자끼리 또렷하게 뭉쳐서 시각화에 특히 좋다. 대신 계산이 무겁다.

```python
from sklearn.decomposition import PCA
from sklearn.manifold import TSNE

pca = PCA(n_components=2)
X_pca = pca.fit_transform(X)

# t-SNE는 fit과 transform을 분리 못 하고 항상 fit_transform으로 한 번에
tsne = TSNE(n_components=2, random_state=100)
X_tsne = tsne.fit_transform(X)
```

> **복습 포인트**
>
> PCA는 여러 컬럼을 수학적으로 섞어 새 축을 만들기 때문에, 축소 후엔 "이 축이 원래 무슨 의미였는지" 해석이 어려워진다. t-SNE는 `fit_transform`만 쓸 수 있다는 것도 자주 헷갈리는 지점이라 따로 적어둔다.

---

## 💡 정리하며

비지도학습은 정답이 없어서 **데이터 구조를 보는 안목**이 더 중요하다는 걸 다시 느꼈다.

직관적으로 묶고 싶으면 K-Means, 확률로 유연하게 묶고 싶으면 GMM. 특성이 너무 많아 느리면 PCA로 요약하고, 고차원 데이터가 실제로 어떻게 뭉쳐 있는지 눈으로 보고 싶으면 t-SNE로 그려보는 것. 이 정도 기준을 잡아두면 상황마다 뭘 꺼낼지 덜 헤맬 것 같다.

다음 편은 의사결정나무와 앙상블 기법이다.
