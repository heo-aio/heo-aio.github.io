+++
title = "[Data Analysis] Matplotlib, 왜 plt 말고 fig, ax를 써야 할까"
date = 2026-07-10
draft = false
tags = ["Python", "Matplotlib", "Seaborn", "Visualization"]
categories = ["data-analysis"]
math = false
+++

NumPy로 배열을 다루고 Pandas로 표를 가공했다면, 마지막은 결과를 눈으로 확인하는 **시각화**다.

그동안 나는 아무 생각 없이 `plt.plot()`, `plt.title()`을 써왔다.

그런데 잘 짠 예제나 실무 코드는 하나같이 `fig, ax = plt.subplots()`로 시작하는 것이 계속 눈에 밟혔다.

이번에 그 이유를 제대로 파고들었고, 이번 데이터 분석 공부에서 개인적으로 가장 크게 얻은 부분이었다.

> 📓 **실습 노트북 전체**: [GitHub에서 보기](https://github.com/heo-aio/heo-aio.github.io/blob/main/static/notebooks/matplotlib-fig-ax.ipynb) · [다운로드 (.ipynb)](/notebooks/matplotlib-fig-ax.ipynb)
>
> 두 방식을 비교한 코드와 iris 데이터 2×3 subplot 예시가 노트북에 그대로 담겨 있다.

---

# 📌 두 가지 방식

같은 그래프를 그리는 방법에는 두 가지가 있다. 결과 그림은 같지만 성격은 완전히 다르다.

## 방식 1 — plt 직접 호출 (암묵적 방식)

```python
import matplotlib.pyplot as plt

plt.figure(figsize=(6, 4))
plt.plot([1, 2, 3], [4, 5, 6])
plt.title("title")
plt.xlabel("x-axis")
plt.show()
```

## 방식 2 — fig, ax 명시적 방식 (객체지향)

```python
fig, ax = plt.subplots(figsize=(6, 4))

ax.plot([1, 2, 3], [4, 5, 6])
ax.set_title("title")
ax.set_xlabel("x-axis")
plt.show()
```

방식 1은 "그냥 그린다"이고, 방식 2는 "**그림(fig)과 축(ax)이라는 객체를 손에 쥐고** 그 위에 그린다"이다.

이 차이가 왜 중요한지가 핵심이다.

---

# 1️⃣ plt 직접 호출이 위험한 이유

핵심은 `plt`가 **"현재 활성화된 Figure/Axes"를 전역 상태로 관리**한다는 점이다.

즉 `plt.plot()`은 "지금 활성화된 어딘가"에 그리는 것인데, 그 대상이 코드 흐름에 따라 **오류도 경고도 없이** 조용히 바뀐다.

내가 정리한 문제점은 이렇다.

- **전역 상태 의존** — 함수 호출 하나가 활성 axes를 조용히 바꿔버릴 수 있고, 이때 아무 오류도 발생하지 않는다.
- **반복문·함수 안에서 오작동** — `for` 루프나 헬퍼 함수 안에서 `plt.subplot()`을 호출하면 의도한 axes가 아니라 "그 순간 활성화된" axes에 그려진다. 버그가 코드 에러가 아니라 그림 결과로만 나타나 원인 찾기가 매우 어렵다.
- **재사용 불가** — 플로팅 함수를 만들어도 특정 axes를 인자로 받을 수 없어, 항상 전역 상태가 가리키는 곳에만 그려진다.
- **subplot 개별 제어 불가** — 여러 subplot 중 하나에만 옵션(로그 스케일, y축 범위 등)을 주려면 매번 `plt.subplot()`으로 다시 활성화해야 한다.
- **Figure 수준 옵션 접근 불가** — `fig.suptitle()`, `fig.tight_layout()`처럼 그림 전체를 제어하는 메서드는 `fig` 객체 없이는 호출하기 어렵다.

특히 "버그가 코드 오류가 아니라 그림 결과로만 나타난다"는 부분이 뜨끔했다.

반면 `fig, ax`로 객체를 직접 쥐고 있으면 `axes[0]`, `axes[1]`처럼 원하는 subplot을 콕 집어 제어할 수 있다.

---

# 2️⃣ seaborn을 쓸 거면 사실상 필수

결정적인 이유는 seaborn이었다.

seaborn의 모든 플로팅 함수는 `ax=` 인자로 **그릴 대상 axes를 받는다.**

```python
sns.scatterplot(data=iris, x="sepal_length", y="petal_length",
                hue="species", ax=axes[0, 0])   # 이 축에 그려라
```

`ax=`를 지정하지 않으면 seaborn이 새 Figure를 만들거나 엉뚱한 axes에 그려서, 기존 subplot 레이아웃이 망가진다.

아래처럼 2×3 subplot에 서로 다른 그래프를 각각 꽂아 넣으려면, 객체지향 방식이 아니면 애초에 통제가 안 된다.

```python
fig, axes = plt.subplots(2, 3, figsize=(14, 8))

fig.suptitle("iris dataset — fig, ax pattern demo")

sns.scatterplot(data=iris, x="sepal_length", y="petal_length",
                hue="species", ax=axes[0, 0])
sns.violinplot(data=iris, x="species", y="sepal_width", ax=axes[0, 1])
# ... axes[0, 2], axes[1, 0] ... 각 축을 직접 지정

fig.tight_layout()
```

## 📷 실행 결과

![위 예시 코드 fig, ax로 그린 2x3 subplot 예시](/images/data/matplotlib_fig_ax_subplots.png)

앞으로 seaborn을 자주 쓸 것 같은데, 그렇다면 `fig, ax` 방식은 선택이 아니라 기본으로 깔고 가야겠다고 생각했다.

---

# 📊 두 방식 비교

| 구분 | plt 직접 호출 | fig, ax (객체지향) |
|------|---------------|--------------------|
| 그리는 대상 | 현재 활성화된 축(전역) | 지정한 축 객체 |
| subplot 개별 제어 | 어렵다 | `axes[i]`로 명확 |
| 함수로 재사용 | 불가 (전역 의존) | `ax` 인자로 가능 |
| seaborn 연동 | 레이아웃 깨질 수 있음 | `ax=`로 안정적 |
| 재현성 | 실행 순서에 영향받음 | 일관적 |

---

# 💡 공부하면서 느낀 점

그동안 `plt.`으로 대충 그려도 그림이 나오니까 문제를 못 느꼈던 것뿐이었다.

"결과가 실행 순서에 따라 달라진다"는 말이 특히 와닿았다. Jupyter에서 셀 순서를 바꿔 실행하면 같은 코드도 다른 그림이 나올 수 있다는 뜻이니까.

시각화도 결국 재현성이 중요하고, 그러려면 전역 상태에 기대지 말고 **객체를 직접 쥐고 그려야 한다**는 것을 배웠다.

앞으로는 그래프 한 장을 그리더라도 `fig, ax = plt.subplots()`로 시작하는 습관을 들이려고 한다.

---

## 참고 자료

- [Matplotlib 공식 — Figure and Axes 개요](https://matplotlib.org/stable/users/explain/figure/figure_intro.html)
- [Matplotlib 공식 — subplot 튜토리얼](https://matplotlib.org/stable/gallery/subplots_axes_and_figures/subplots_demo.html)
- [Real Python — Matplotlib Guide](https://realpython.com/python-matplotlib-guide/)
