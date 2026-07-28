+++
title = "[HTML/CSS] 웹에 생동감 넣기 - transform, transition, animation으로 움직이는 갤러리 만들기"
date = 2026-07-28T11:00:00+09:00
draft = false
tags = ["HTML", "CSS", "transform", "transition", "animation"]
categories = ["dev"]
math = false
+++

지금까지는 레이아웃(박스 모델, display, grid)만 다뤘는데, 오늘은 처음으로 "움직이는" CSS를 배웠다. `transform`, `transition`, `animation` 세 개가 헷갈릴 수 있다고 해서 각각 뭐가 다른지부터 정리하고, 마지막엔 이 셋을 다 합쳐서 hover하면 반응하는 갤러리 페이지를 실습했다. 실습 이미지는 강의 자료 원본 대신 내가 직접 만든 아이콘으로 바꿔서 개인 블로그용으로 각색했다.

**목차**
1. [Transform - 요소의 모양을 바꾼다](#-1-transform---요소의-모양을-바꾼다)
2. [Transition - 변화에 시간을 준다](#-2-transition---변화에-시간을-준다)
3. [Animation - 반복/자동 재생되는 변화](#-3-animation---반복자동-재생되는-변화)
4. [응용: Transform + Animation](#-4-응용-transform--animation)
5. [실전 예제: 움직이는 갤러리 만들기](#-5-실전-예제-움직이는-갤러리-만들기)
6. [오늘 배운 것 정리](#-오늘-배운-것-정리)

---

## 🔄 1. Transform - 요소의 모양을 바꾼다

`transform`은 요소의 크기, 회전, 위치, 기울기를 바꾸는 속성이다. 중요한 건 `transform`은 **한 번의 "상태 변화"**일 뿐이고, 그 변화가 어떻게 보일지(서서히 vs 즉시)는 `transition`이나 `animation`이 따로 담당한다는 것. 오늘 처음 헷갈렸던 지점이 여기였다 — `transform`만 써놓고 왜 안 움직이지 했는데, 애초에 `transform`은 "움직임"이 아니라 "최종 모양"을 정의하는 속성이었다.

```css
.transform {
    width: 100px;
    height: 100px;
    background-color: red;
    margin: 100px 0 0 100px;

    /* 같은 속성에 값을 두 번 정의하면 뒤에 것으로 덮어써진다 */
    /* transform: rotate(45deg);
    transform: scale(2, 3); */

    /* 여러 효과를 동시에 주려면 한 줄에 이어붙이면 된다 */
    /* transform: rotate(45deg) scale(0.5, 0.5); */

    /* translate: 양수면 오른쪽/아래로, 음수면 왼쪽/위로 이동 */
    /* transform: translate(-50px, -70px); */

    /* skew: x축, y축 기준 기울이기 */
    /* transform: skew(0deg, 0deg); */
}
```

네 가지를 나란히 놓고 비교해보니 차이가 훨씬 명확해졌다.

![transform 4가지 비교 - 기본, rotate(45deg), scale(1.4,1), skew(20deg,0), translate(20px,-20px)](/images/html_css/transform-examples.png)

- `rotate(각도)`: 그 자리에서 회전. 음수도 가능
- `scale(x, y)`: width를 x배, height를 y배로 확대/축소. `scale(1.4, 1)`처럼 가로만 늘릴 수도 있다
- `skew(x각도, y각도)`: 축을 기준으로 비스듬히 기울임
- `translate(x, y)`: 원래 자리를 기준으로 좌표 이동. `margin`이랑 비슷해 보이지만 문서 흐름(다른 요소 배치)에는 영향을 안 준다는 게 다른 점 같다

### prefix(접두사)는 왜 붙일까

```css
/* Firefox */
-moz-transform: skew(0deg, 0deg);
/* Chrome, Safari */
-webkit-transform: skew(0deg, 0deg);
/* Opera */
-o-transform: skew(0deg, 0deg);
/* Internet Explorer */
-ms-transform: skew(0deg, 0deg);
```

`transform`처럼 비교적 최신 CSS 기능은 브라우저마다 지원 시점이 달랐어서, 구형 브라우저에서도 동작하게 하려면 각 브라우저 엔진 이름이 붙은 접두사 버전을 같이 써주는 관례가 있었다고 한다. 요즘 브라우저는 거의 다 지원해서 실무에서 자주 볼 일은 없을 것 같지만, "왜 코드에 이상한 하이픈 붙은 속성이 있지?" 싶을 때 이해할 수 있게 알아만 둔다.

> ✅ **핵심**: `transform`은 "최종 모양"을 정의하는 속성이고, 그 자체로는 애니메이션이 아니다. 여러 효과는 한 줄에 이어붙여서 동시 적용한다.

---

## ⏱️ 2. Transition - 변화에 시간을 준다

`transition`은 어떤 속성값이 바뀔 때(대표적으로 `:hover`), 그 변화가 순식간이 아니라 시간을 두고 부드럽게 일어나게 만드는 속성이다.

```css
.transition {
    width: 100px;
    height: 100px;
    background-color: red;

    /* 어떤 속성에 효과를 줄래? */
    transition-property: width, background-color;
    /* 효과를 얼마 동안 진행할까? (1s = 1000ms) */
    transition-duration: 1000ms;
    /* 어떤 속도로 움직일까? */
    transition-timing-function: ease;
    /* 몇 초 후에 시작할까? */
    transition-delay: 500ms;
}

.transition:hover {
    width: 500px;
    background-color: yellow;
}
```

![transition hover 전/후 비교 - width 100px 빨간 박스에서 400px 노란 박스로 변화](/images/html_css/transition-before-after.png)

`transition-timing-function`은 종류가 많아서 헷갈렸는데 이렇게 정리했다.

| 값 | 움직임 |
|---|---|
| `ease` (기본값) | 천천히 → 빠르게 → 천천히 |
| `ease-in` | 천천히 시작해서 점점 빨라짐 (가속) |
| `ease-out` | 빠르게 시작해서 점점 느려짐 (감속) |
| `ease-in-out` | `ease`랑 비슷하지만 시작/끝이 더 강조됨 |
| `linear` | 처음부터 끝까지 일정한 속도 |
| `cubic-bezier(p1,p2,p3,p4)` | 베지에 곡선으로 커스텀 속도 곡선을 직접 정의 |

한 줄로 축약하면 `transition: property duration timing-function delay;` 순서로 쓸 수 있다.

```css
.transition {
    transition: width 2s linear 1s;
}
.transition:hover { width: 300px; }
```

`:hover`처럼 미리 정의된 조건(가상 클래스)을 셀렉터 뒤에 붙일 때는 **띄어쓰기 없이** 붙여 써야 한다는 것도 오늘 알았다. `.transition :hover`(띄어쓰기 있음)라고 쓰면 완전히 다른 의미(`.transition`의 자손 요소에 hover가 걸렸을 때)가 되어버린다.

> ✅ **핵심**: `transition`은 "속성값이 바뀔 때 그 변화에 걸리는 시간"을 정의한다. `transition: property duration timing-function delay;` 순서로 축약 가능.

---

## 🔁 3. Animation - 반복/자동 재생되는 변화

`transition`이 "hover 같은 특정 조건이 있어야만" 동작한다면, `animation`은 조건 없이 스스로 재생된다. 대신 `@keyframes`로 "어떤 상태에서 어떤 상태로 변할지"를 미리 정의해둬야 한다.

```css
.animation {
    width: 300px;
    height: 300px;
    background-color: yellow;

    animation-name: changeWidth;
    animation-duration: 1000ms;
    animation-delay: 1000ms;
    animation-iteration-count: 1; /* infinite로 하면 무한 반복 */
    animation-direction: alternate;
    animation-fill-mode: backwards;
}

@keyframes changeWidth {
    from { width: 300px; }
    to   { width: 100px; }
}
```

![animation keyframe from/to 비교 - width 300px에서 100px로 변하는 애니메이션의 시작/끝 상태](/images/html_css/animation-before-after.png)

`animation-direction`이랑 `animation-fill-mode`가 제일 헷갈렸다.

**animation-direction**
- `normal`: from → to (한 번만)
- `alternate`: from → to → from (왕복. `iteration-count`를 2씩 소모하기 때문에 1로 주면 왕복이 다 안 끝나고 어색하게 멈출 수 있다)
- `alternate-reverse`: to → from → to
- `reverse`: to → from

**animation-fill-mode**: 애니메이션이 끝난 뒤(혹은 시작 전) 어떤 상태를 유지할지 정하는 옵션. 이게 없으면 애니메이션이 끝나자마자 요소가 원래 CSS 상태로 툭 끊기듯 돌아가서 부자연스러울 수 있다.
- `forwards`: 마지막 keyframe 상태에서 정지
- `backwards`: 시작 전에도 `from` 상태를 미리 적용

`@keyframes` 이름은 페이지 안에서 중복되면 안 된다는 것도 메모. `changeWidth`라는 이름의 keyframes가 또 있으면 나중에 정의한 게 앞의 걸 덮어써버릴 것 같다.

> ✅ **핵심**: `transition`은 조건(hover 등)이 있어야 동작하고, `animation`은 조건 없이 스스로 재생된다. `@keyframes`로 상태 변화를 정의하고 `animation-*` 속성으로 재생 방식을 제어한다.

---

## 🌀 4. 응용: Transform + Animation

`transform`(모양)과 `animation`(반복 재생)을 합치면 계속 회전하면서 크기도 변하는 요소를 만들 수 있다.

```css
.box1 {
    width: 300px;
    height: 300px;
    background-color: red;

    animation: rotation 1500ms linear 500ms infinite alternate;
}

@keyframes rotation {
    from { transform: rotate(45deg) scale(0.1); }
    to   { transform: rotate(-45deg) scale(0.5); }
}
```

![transform+animation 응용 - rotate(45deg) scale(0.1)에서 rotate(-45deg) scale(0.5)로 무한 반복 왕복하는 애니메이션의 두 극단 상태](/images/html_css/transform-animation-combo.png)

`animation` 한 줄로 축약할 때 순서를 헷갈리기 쉬운데, `name duration timing-function delay iteration-count direction` 순서다. 먼저 나오는 숫자가 `duration`, 그다음 숫자가 `delay`라는 걸 기억해두려고 한다. prefix를 붙일 때는 `@keyframes`에도 똑같이 prefix가 붙어야 하고(`@-webkit-keyframes`), 안에 있는 `transform`도 같은 prefix로 맞춰줘야 한다는 점도 메모.

> ✅ **핵심**: `transform`은 "무엇을"(모양), `animation`은 "어떻게"(재생 방식)를 담당한다. 둘을 `@keyframes` 안에서 조합하면 복합적인 움직임을 만들 수 있다.

---

## 🖼️ 5. 실전 예제: 움직이는 갤러리 만들기

오늘 배운 걸 다 합쳐서 헤더(로고+네비게이션) + 카드형 갤러리 레이아웃을 만들고, hover 시 색상/투명도/이미지 크기가 변하는 인터랙션을 넣는 실습을 했다. 원래 실습 이미지는 강의 자료 원본이었는데, 외부에 올리는 개인 블로그라 저작권 문제 없게 이미지는 전부 내가 새로 만든 기술 스택 아이콘(HTML/CSS/JS/Python/Git/Docker/AI/Algorithm)으로 바꾸고, 로고나 텍스트도 내 개인 컨셉으로 각색했다.

![움직이는 갤러리 기본 화면 - 상단 네비게이션과 8개의 기술 스택 카드가 그리드로 배치됨](/images/html_css/gallery-default.png)

### 헤더 레이아웃: float으로 좌우 배치

```css
#intro h1 {
    float: left;
    width: 50%;
    height: 80px;
}

#intro nav {
    float: right;
    width: 50%;
    height: 80px;
}
```

로고는 왼쪽, 네비게이션은 오른쪽으로 `float`를 이용해서 배치했다. 지난번 아티클 카드 실습에서 배운 blockify 개념이 여기서도 그대로 적용됐다 — `float`가 걸리면 자동으로 block처럼 취급되니까 `width`, `height`를 줄 수 있는 것.

### 네비게이션 메뉴에 transition 걸기

```css
#intro nav ul li:hover {
    background-color: #917f7f;
}

#intro nav ul li {
    transition-property: background-color;
    transition-duration: 500ms;
}
```

마우스를 올리면 배경색이 0.5초에 걸쳐 서서히 어두워진다. `:hover` 자체는 즉시 스타일을 바꾸는 조건이고, 그 변화를 부드럽게 만드는 건 `transition`의 역할이라는 걸 여기서 다시 확인했다.

### 카드에 opacity + 이미지 확대 효과

```css
#main article:hover {
    opacity: 0.8;
}
#main article {
    transition-property: opacity;
    transition-duration: 500ms;
}

#main article img:hover {
    /* width/height를 %로 늘리는 방법도 있지만
       0,0 좌표 기준으로 커지고 레이아웃 재계산(리플로우)이 필요해서
       transform: scale()보다 덜 부드럽다고 한다 */
    transform: scale(1.1);
}
#main article img {
    transition: all 500ms;
}
```

카드에 마우스를 올리면 카드 전체는 살짝 반투명해지고, 이미지는 `scale(1.1)`로 10% 커진다. 주석으로 남겨져 있던 설명이 인상 깊었는데, `width: 110%`처럼 크기 자체를 늘리는 방식은 브라우저가 레이아웃을 다시 계산해야 해서(리플로우) 상대적으로 버벅일 수 있고, `transform: scale()`은 레이아웃 재계산 없이 그리기(페인팅) 단계에서만 처리돼서 더 부드럽다고 한다. "크기를 키운다"는 결과는 같아도 내부적으로 어떤 방식이 더 효율적인지가 다르다는 게 신기했다.

![움직이는 갤러리 hover 화면 - 네비게이션 두번째 메뉴 배경색 변화, AI/ML 카드 이미지가 살짝 확대됨](/images/html_css/gallery-hover.png)

### footer에서 다시 만난 float와 clear

```css
#footer {
    clear: both; /* 이전에 사용한 float 무시 (안전하게) */
}
#footer div.copyright { width: 50%; float: left; }
#footer div.address   { width: 50%; float: right; }
```

`main` 영역에서 카드들을 `float`로 배치했기 때문에, 그 아래 오는 `footer`가 float 영향을 받지 않게 `clear: both`를 꼭 챙겨야 했다. 지난 아티클 카드 실습 때 메모해둔 "float 쓸 때마다 clear 필요한지 체크하기"가 여기서 바로 써먹혔다.

> ✅ **핵심**: `float` + `blockify`, `transition` + `:hover`, `transform: scale()`이 리플로우 없이 더 부드러운 이유까지 — 지금까지 따로 배운 개념들이 실전 예제 하나에 다 녹아있었다.

---

## 💡 오늘 배운 것 정리

- 🔄 `transform`은 "최종 모양"만 정의한다. 움직임(속도, 반복)은 `transition`/`animation`이 담당
- ⏱️ `transition`은 조건(`:hover` 등)이 있어야 동작, `animation`은 조건 없이 스스로 재생
- 🔁 `animation-direction: alternate`는 `iteration-count`를 2씩 소모한다 — 1로 주면 왕복이 어색하게 끊길 수 있음
- 🖼️ 크기를 키울 때 `width/height`보다 `transform: scale()`이 리플로우 없이 더 부드럽다
- 🎨 강의 자료 이미지를 그대로 개인 블로그에 올리면 안 되니, 실습 결과물을 공유할 땐 이미지·텍스트를 직접 만든 것으로 바꾸는 습관을 들여야겠다

**다음에 실전 적용할 것**

1. hover 효과 넣을 때 `transition-property`를 `all`로 퉁치지 말고 필요한 속성만 지정해서 성능 챙기기
2. 크기 변화가 필요한 애니메이션은 `width/height`보다 `transform: scale()` 먼저 고려하기
3. 외부에 공유하는 실습 결과물은 이미지·로고·텍스트를 개인 저작물로 교체하고 올리기
