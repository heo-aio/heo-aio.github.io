+++
title = "[HTML/CSS] 박스 모델과 레이아웃 - margin, padding, display, float까지"
date = 2026-07-26T10:00:00+09:00
draft = false
tags = ["HTML", "CSS", "박스모델", "레이아웃"]
categories = ["dev"]
math = false
+++

오늘은 웹사이트 레이아웃을 잡는 방법을 배웠다. 태그 이름 외우는 건 할 만했는데, 박스 모델이랑 마진 병합 현상 부분에서 한 번 막혔다. 특히 마진 병합은 실습 파일을 직접 열어보면서 "왜 숫자가 저렇게 나오지?" 하고 몇 번 다시 읽었다. 오늘 배운 내용 + 실습 코드를 같이 정리해둔다.

**목차**
1. [박스 모델](#-박스-모델box-model)
2. [Block 요소와 Inline 요소](#-block-요소와-inline-요소-실습으로-다시-확인)
3. [마진 병합 현상](#-마진-병합-현상margin-collapse)
4. [레이아웃에 영향을 미치는 속성 (display, float/clear)](#️-레이아웃에-영향을-미치는-속성)
5. [정리](#-정리하면)

---

## 📦 박스 모델(Box Model)

HTML 요소는 전부 사각형 박스로 취급되는데, 이 박스는 안쪽부터 content - padding - border - margin, 네 겹으로 이루어져 있다.

![content, padding, border, margin 네 겹으로 이루어진 박스 모델 구조도](/images/html_css/box-model-diagram.png)

- **content**: 실제 내용물이 들어가는 영역. `width`, `height`로 크기 결정
- **padding**: border 안쪽의 여백
- **border**: 테두리
- **margin**: border 바깥쪽의 여백

오늘 실습(`실습1.html`)에서 쓴 코드는 이렇다.

```html
<style>
  div {
    width: 300px;
    height: 300px;
    background-color: yellow;

    border: 10px solid red;
    margin-left: 100px;
    padding-left: 100px;
  }
</style>

<div>Hello World</div>
```

`margin-left`와 `padding-left`를 나란히 써보면 차이가 확실해진다.

- `margin-left`: border **바깥쪽**에서 왼쪽에 여백을 만든다 (요소 자체가 오른쪽으로 밀림)
- `padding-left`: border **안쪽**에서 왼쪽에 여백을 만든다 (콘텐츠가 안에서 오른쪽으로 밀림, 박스 자체 크기는 그만큼 커짐)

즉 겉보기엔 둘 다 "왼쪽 여백"처럼 보이지만, 기준선이 border 안쪽이냐 바깥쪽이냐가 다르다는 게 핵심이었다.

> ✅ **핵심**: `margin`은 박스 바깥 여백(요소가 밀림), `padding`은 박스 안쪽 여백(콘텐츠가 밀리고 박스 크기가 커짐)

top/right/bottom/left를 하나하나 쓰는 대신 시계방향(12시 → 3시 → 6시 → 9시) 순서로 한 줄에 축약해서 쓸 수도 있다.

```css
/* margin-top margin-right margin-bottom margin-left 순서 */
div { margin: 0 0 0 100px; }
```

---

## 📏 Block 요소와 Inline 요소, 실습으로 다시 확인

Block/Inline 개념 자체는 지난 시간에 배웠는데, 오늘은 각 성격이 실제로 CSS 속성 적용에 어떤 차이를 만드는지 실습으로 확인했다.

| 구분 | 줄바꿈 | width / height | margin / padding |
|---|:---:|:---:|:---:|
| Block 요소 | O | 적용 가능 | 적용 가능 (상하 배치 O) |
| Inline 요소 | X | 적용 불가 | 좌우만 적용, 상하는 무시 |

`실습2.html`에서 `<p>`(Block)와 `<a>`(Inline)에 똑같이 `width`, `height`, `margin-top`을 줘보고 결과를 비교했다.

```html
<style>
  /* inline은 높이와 넓이 조정 불가 */
  p, a {
    width: 200px;
    height: 200px;
  }
  p { background-color: yellow; }
  a { background-color: pink; }

  /* inline은 margin 등 여백 요소를 사용할 수 없다 */
  p, a { margin-top: 100px; }
</style>

<p>Block 요소</p>
<a href="#">Inline 요소</a>
```

똑같은 코드를 줬는데도 `<p>`는 200×200 노란 박스로 잘 그려지고 위쪽에 여백도 생기는 반면, `<a>`는 width/height와 margin-top이 전부 무시된 채로 글자 크기만큼만 차지한다. Inline 요소는 애초에 "박스"로 취급되지 않는다는 걸 눈으로 확인한 셈.

> ✅ **핵심**: Inline 요소(`<a>` 등)는 width/height, 상하 margin이 전부 무시된다. "박스"가 아니라 글자 크기만큼만 차지하기 때문.

---

## 🌊 마진 병합 현상(Margin Collapse)

오늘 가장 헷갈렸던 부분. 마진은 항상 내가 준 값 그대로 더해지는 게 아니라, 특정 상황에서는 두 마진이 하나로 합쳐져 버린다.

![형제간 마진 병합과 부모 자식간 마진 병합을 비교한 그림](/images/html_css/margin-collapse.png)

### 1) 형제간 마진 병합

```css
.box1 { margin-bottom: 150px; }
.box2 { margin-top: 100px; }
```

`box1`의 `margin-bottom: 150px`와 `box2`의 `margin-top: 100px`가 있으면, 두 값이 더해져서 250px 간격이 생길 것 같지만 실제로는 **둘 중 더 큰 값인 150px 하나만** 적용된다. 이게 형제 요소 사이의 마진 병합이다.

> 참고로 원래 강의 노트에는 `box2`쪽 속성이 `bottom-top`으로 적혀 있었는데, CSS에는 그런 속성명이 없다. 아마 `margin-top`을 옮겨 적으면서 생긴 오타로 보여서 위 코드는 `margin-top`으로 고쳐서 정리했다.

`실습3.html`에는 이 형제 마진 병합 예시가 주석으로 남아 있었다.

```html
<!-- 형제 지간에 발생하는 마진 병합 현상 -->
<div id="box1"></div>
<div id="box2"></div>

<style>
  /* #box1 { margin-bottom: 150px; }  -> 150 + 100이 아니라 150으로 흡수된다 */
  /* #box2 { margin-top: 100px; } */
</style>
```

> ✅ **핵심**: 형제 요소의 margin-bottom + margin-top은 더해지지 않고, **더 큰 값 하나만** 적용된다.

### 2) 부모 자식간 마진 병합

더 헷갈렸던 케이스. 자식 요소의 margin이 부모에게도 영향을 준다.

```html
<main role="main">
  <article></article>
</main>
```

```css
main { width: 100%; height: 400px; background-color: yellow; }
article { width: 100px; height: 100px; background-color: red; margin-top: 10000px; }
```

`article`에만 `margin-top`을 줬는데도, `main`이 그 여백만큼 같이 밀려 내려간다. 자식의 상단 마진이 부모 바깥으로 새어 나가서 부모 자신의 마진처럼 작동하는 것.

`실습3.html`에는 이걸 막는 방법도 주석으로 힌트가 남아 있었다.

```css
main {
  /* 부모와 자식이 각각 마진을 사용하고 싶다면?
     부모에게 테두리(border)를 부여해 독립적으로 만들어주면 된다 */
  /* border: 1px solid black; */
}
```

부모에게 `border`(테두리가 안 보이게 하고 싶으면 `border: 1px solid transparent;`)를 주면 부모와 자식 사이에 명확한 경계가 생겨서 자식의 마진이 더는 부모 바깥으로 새어 나가지 않는다. 실무에서 "분명 margin-top을 자식한테만 줬는데 왜 부모까지 밀리지?" 싶을 때 써먹으면 될 것 같다.

> ✅ **핵심**: 자식의 margin-top이 부모 바깥으로 새어 나가 부모까지 같이 밀린다. → 부모에 `border`(또는 투명 border)를 주면 경계가 생겨서 방지된다.

덧붙여 `실습3.html`에는 `position: absolute`와 `position: fixed`도 같이 등장했다. `article`에 준 `position: absolute`는 요소를 일반적인 문서 흐름에서 빼내 버리고(강의 노트 표현으로는 "부모를 기준으로 움직임, 안하무인"), `.fixed` 클래스의 `position: fixed`는 스크롤해도 화면에 고정되는 속성이라 보통 따라다니는 메뉴 같은 데 쓴다고 한다. 이 둘은 오늘 배운 범위를 살짝 벗어나긴 하는데, 다음에 `position` 속성을 제대로 배울 때를 위해 일단 메모만 해둔다.

---

## 🎛️ 레이아웃에 영향을 미치는 속성

### display

`display`는 Block/Inline이라는 태그의 기본 성격 자체를 바꿔버리는 속성이다.

```html
<style>
  p { display: inline; }       /* 기본값 block -> inline으로 변경 */
  a { display: block; }        /* 기본값 inline -> block으로 변경 */
</style>
```

`실습4.html`에서 이 반전을 직접 확인했다.

```html
<style>
  p {
    width: 300px;
    height: 300px;
    background-color: pink;
    display: inline; /* 기본 속성 block -> inline으로 변경 */
  }

  a {
    width: 300px;
    height: 300px;
    background-color: yellow;
    display: block; /* 기본 속성 inline -> block으로 변경 */
    margin-top: 10px;
  }
</style>

<p>Block Element</p>
<a href="#">Inline Element</a>
```

원래대로면 `<p>`는 세로로 쌓이고 `<a>`는 가로로 나열돼야 하는데, `display`로 성격을 바꿔주니 정반대로 `<p>`가 한 줄에 붙어버리고 `<a>`가 세로로 쌓인다. `display: inline-block`을 쓰면 두 성격을 동시에 가질 수 있다는 것도 필기해뒀다. Block처럼 width/height/margin을 다 쓸 수 있으면서, Inline처럼 줄바꿈 없이 옆으로 나열되는 조합이라 실무에서 버튼이나 메뉴 아이템 만들 때 자주 쓴다고.

> ✅ **핵심**: `display`로 Block/Inline 성격 자체를 바꿀 수 있다. `inline-block`은 "크기 지정 가능(Block) + 줄바꿈 없음(Inline)"을 동시에 가진다.

### float / clear

이 부분은 실습 파일이 따로 없어서, 노트 내용을 바탕으로 직접 예시를 만들어봤다.

![float로 좌우에 배치된 요소와 clear로 흐름을 정리하는 레이아웃](/images/html_css/float-clear-layout.png)

```html
<style>
  header, footer { width: 100%; background-color: #eee; }

  .left  { float: left;  width: 200px; background-color: #ffd28a; }
  .right { float: right; width: 200px; background-color: #9be3d8; }
  main   { background-color: #f7f7f7; }

  footer { clear: both; } /* float된 요소들 아래로 footer를 내려서 겹치지 않게 정리 */
</style>

<header></header>
<aside class="left">Hello World</aside>
<main></main>
<aside class="right">Hello World</aside>
<footer></footer>
```

- **float**: 선택한 요소를 왼쪽 끝 또는 오른쪽 끝으로 띄워서 정렬시킨다. 이름 그대로 요소를 문서 흐름에서 살짝 띄워 새로운 레이어처럼 만드는 방식
- **clear**: float된 요소 때문에 밀린 레이아웃을 다시 정리할 때 사용. `clear: both;`를 주면 float된 요소 아래로 완전히 내려가서, footer 같은 다음 요소가 float 요소와 겹치지 않는다

float를 쓴 요소 다음에 바로 오는 요소에 `clear`를 안 주면 레이아웃이 깨지는 경우가 많다고 하니, float를 쓸 땐 항상 "이 뒤에 clear가 필요한가?"를 같이 생각해야 할 것 같다.

> ✅ **핵심**: `float`는 요소를 문서 흐름에서 띄워 좌/우로 배치한다. `clear: both`로 float 영향을 받지 않게 다음 요소를 내려줘야 레이아웃이 안 깨진다.

### 브라우저 기본 여백 제거하기

`<html>`과 `<body>` 태그는 브라우저마다 기본으로 약간의 margin/padding을 갖고 있어서, 레이아웃을 딱 맞게 잡으려면 이걸 먼저 초기화해줘야 한다.

```css
html, body {
  margin: 0;
  padding: 0;
}

/* 혹은 전체 태그를 한 번에 초기화 */
* {
  margin: 0;
  padding: 0;
}
```

`*`(전체 선택자)로 모든 태그의 margin/padding을 한 번에 0으로 만드는 방법도 있는데, 프로젝트 규모가 커지면 전체 초기화보다는 필요한 태그만 골라서 초기화하는 게 낫다는 얘기도 들었던 것 같아서 나중에 CSS 리셋 관련해서 따로 찾아봐야겠다.

---

## 💡 정리하면

오늘 배운 걸 한 줄로 정리하면, 박스 모델의 4겹 구조(content-padding-border-margin)를 이해하고 나면 마진 병합이나 float 같은 "왜 이렇게 되지?" 싶은 현상들도 결국 마진과 박스 흐름 규칙으로 설명이 된다는 것. 특히 마진 병합은 실습 파일을 직접 열어서 숫자를 바꿔가며 확인하지 않았다면 그냥 넘어갔을 것 같다.

**오늘의 핵심 4가지**

- 📦 박스는 항상 `content → padding → border → margin` 순서로 쌓인다
- 🌊 형제 마진은 더해지지 않고 **더 큰 값 하나만** 적용된다 (마진 병합)
- 🌊 자식의 margin-top은 부모 바깥까지 새어 나간다 → `border`로 경계를 만들면 방지 가능
- 🎛️ `display`로 Block/Inline 성격을 바꿀 수 있고, `float` 뒤엔 `clear`를 챙겨야 레이아웃이 안 깨진다

**다음에 실전 적용할 것**

1. 부모 자식 마진 병합 막을 때 → `border: 1px solid transparent;`
2. float 쓸 때마다 → "이 뒤에 clear 필요한가?" 항상 체크
