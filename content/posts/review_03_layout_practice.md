+++
title = "[HTML/CSS] 복습과제 3개로 다시 잡은 레이아웃 감각 - inline-block, float, grid"
date = 2026-07-28T02:00:00+09:00
draft = false
tags = ["HTML", "CSS", "복습", "레이아웃", "grid"]
categories = ["dev"]
math = false
+++

새벽까지 붙잡고 있었던 복습과제 3개(프로필 카드 → 아티클 카드 → 대시보드)를 정리해둔다. 저번 시간에 박스 모델이랑 display 개념을 배웠는데, 이번엔 "그래서 실제로 이 상황에서 display 뭐 써야 하는데?"를 계속 판단해야 하는 문제들이라 머리를 좀 썼다. 특히 대시보드는 오타 하나 때문에 30분을 날렸다...

**목차**
1. [미션1. 프로필 카드](#-미션1-프로필-카드)
2. [미션2. 아티클 카드](#-미션2-아티클-카드-float와-blockify)
3. [미션3. 대시보드 레이아웃](#-미션3-대시보드-레이아웃)
4. [오늘 배운 것 정리](#-오늘-배운-것-정리)

---

## 📇 미션1. 프로필 카드

첫 문제는 프로필 카드 UI를 CSS로 완성하는 거였다. 다 채웠다고 생각했는데 브라우저로 열어보니 이미지가 이상하게 커져 있었다.

```css
/* 처음에 이렇게 써놨었다 (틀림) */
.avatar {
    border-radius: 50px;
    vertical-align: middle;
    padding : 24px 24px 24px 24px;   /* 어? 이거 카드 스펙 아닌가? */
    width : 350px;                    /* 이미지가 카드 너비랑 똑같이 커져버림 */
    background-color: white;
    display : inline-block;
    margin-left : 16px;
}
```

`.profile-card`(카드 전체 컨테이너)에 들어가야 할 `padding: 24px`, `width: 350px`, `background-color: white`가 통째로 `.avatar`(이미지)에 잘못 들어가 있었다. 복사-붙여넣기 하다가 스펙을 헷갈린 것 같다. 이미지는 이미지대로 `width/height: 60px`로 따로 지정해줘야 했다.

```css
/* 수정 후 */
.profile-card {
    border-radius: 16px;
    box-shadow: 0 10px 20px rgba(0, 0, 0, 0.5);
    padding: 24px;
    width: 350px;
    background-color: white;
}

.avatar {
    border-radius: 50px;
    width: 60px;
    height: 60px;
    margin-left: 16px;
    display: inline-block;
}
```

### display 판단 기준 세우기

이 문제 풀면서 제일 도움 됐던 건 "언제 어떤 display를 써야 하는지" 판단 기준을 세워본 거였다.

1. 나란히 있어야 하는가, 혼자 줄을 차지해야 하는가?
2. 나란히 있어야 한다면, width/height/margin을 직접 지정해야 하는가? → 그렇다면 `inline`이 아니라 `inline-block`
3. 자식 요소들을 정렬/배치해야 하는가? → `flex`

`.avatar`랑 `.info-area`(이름+뱃지 감싸는 div)는 둘 다 한 줄에 나란히 있어야 하면서 width, margin 같은 것도 직접 줘야 해서 `inline-block`. 반대로 `.profile-card`, `.bio`, `.btn`은 혼자 줄 전체를 차지하면 되니까 그냥 `block`.

마지막에 `body`에 `display: flex; justify-content: center; align-items: center;`를 주면 카드가 화면 정중앙에 딱 떨어진다. "자식을 정렬해야 하는 상황 = flex" 라는 감이 이때 좀 잡혔다.

> ✅ **핵심**: 나란히 배치 + 크기/여백 직접 지정 필요 → `inline-block`. 혼자 줄 전체 차지 → `block`. 자식 정렬 필요 → `flex`.

완성된 결과는 이렇다.

![프로필 카드 완성 화면 - 이미지, 이름, 뱃지, 소개글, 버튼이 카드 형태로 정렬됨](/images/html_css/profile-card-result.png)

---

## 📰 미션2. 아티클 카드 (float와 blockify)

두 번째 문제는 썸네일 이미지를 `float: left`로 띄우고 텍스트가 자연스럽게 흘러가는 잡지 스타일 레이아웃이었다. 여기서 `.thumbnail`에 display를 뭘 줘야 하는지가 제일 헷갈렸다.

```css
.thumbnail {
    object-fit: cover;
    border-radius: 10px;
    float: left; 
    width: 200px;
    height: 140px;
    margin-right: 20px;
    margin-bottom: 12px;
    display: block;   /* 이걸 왜 써야 하지? inline이랑 차이가 없어보이는데 */
}
```

`block`이랑 `inline` 둘 다 넣어봤는데 결과가 똑같이 나와서 찾아봤다. 이유는 **blockify(블록화)** 라는 현상 때문이었다.

> `float: left`를 정의하는 순간 브라우저는 이 요소를 자동으로 block처럼 취급한다. 따라서 float가 붙은 요소는 `display: inline`이라고 써도 내부적으로는 block처럼 동작한다.

즉 `float`가 걸린 순간 이미 강제로 block 취급이 돼서, `display: block`을 명시하든 안 하든 동작은 똑같다. 그럼에도 명시적으로 `block`을 써주는 이유는 **"이 요소는 block처럼 동작한다"는 의도를 코드만 보고도 알 수 있게** 하기 위해서라고 한다. float의 자동 blockify에 암묵적으로 의존하지 말고 명시적으로 적어주는 게 좋은 습관.

### 반면 `.category`(뱃지)는 다르다

```css
.category {
    font-weight: 700;
    border-radius: 20px;
    margin-bottom: 8px;
    padding: 4px 10px 4px 10px;
    font-size: 12px;
    color: #15803d;
    background-color: #f0fdf4;
    /* block을 쓰니 너비가 부모 크기만큼 쭉 늘어남 */
    /* inline을 쓰자니 위에 정의한 마진 패딩이 의미가 없어짐 -> inline-block으로 판단 */
    display: inline-block; 
}
```

`.thumbnail`이랑 헷갈렸던 이유가 여기 있었다. `.category`는 `float`가 안 걸려 있는 일반 `span`이라 blockify가 안 일어난다. 그래서 `display: block`을 주면 진짜로 부모 너비 전체를 차지해버려서 뱃지가 길쭉하게 늘어난다. "내용물 크기만큼만 상자를 갖되 width/padding/margin은 직접 지정하고 싶다" → 이럴 때가 `inline-block`을 쓰는 진짜 이유였다.

> ✅ **핵심**: `float`가 걸린 요소는 자동으로 blockify 되어 `inline`을 줘도 block처럼 동작한다. 그래도 의도 표현을 위해 `display: block`을 명시하는 게 좋은 습관. float 없는 요소에서 "크기 지정 필요 + 나란히 배치"가 필요하면 `inline-block`.

썸네일이 왼쪽에 뜨고 텍스트가 자연스럽게 흘러가는 걸 확인했다.

![아티클 카드 완성 화면 - 썸네일 이미지가 좌측 float, 카테고리 뱃지와 제목/본문이 옆으로 흘러감](/images/html_css/article-card-result.png)

---

## 🖥️ 미션3. 대시보드 레이아웃

마지막 문제가 제일 오래 걸렸다. `grid-template-areas`로 header/sidebar/main/widgets/footer 5영역을 배치하는 대시보드였는데, 다 맞게 짰다고 생각했는데 화면이 이상하게 나왔다.

### 오타 두 개 때문에 30분 날림

```css
.dashboard-container {
  height: 100vh;
  display : gird;                        /* grid 아니고 gird ㅋㅋ */
  grid-template-columns : 240px 1fr 280px;
  grid-template-rows : 60px 1ft 40px;    /* fr 아니고 ft */
  grid-template-areas : "header header header"
  "sidebar main widgets"
  "footer footer footer";
}
```

`display: gird`, `grid-template-rows: 60px 1ft 40px` — 딱 봐도 오타인데 브라우저 콘솔에는 에러 하나 안 뜬다. CSS는 알 수 없는 속성값을 만나면 그냥 조용히 무시해버려서, 화면이 안 깨지는 게 아니라 **grid 자체가 아예 적용이 안 된 채로 요소들이 위아래로 쭉 쌓여서** 나왔다. 눈으로 봐서 딱 "이건 오타다" 싶으면 좋은데, `gird`/`grid`, `1fr`/`1ft`처럼 글자 하나 차이라 한참을 못 찾았다.

![오타로 grid가 적용되지 않아 요소들이 세로로 쌓인 깨진 화면](/images/html_css/dashboard-broken-typo.png)

```css
/* 고친 후 */
.dashboard-container {
  height: 100vh;
  display: grid;
  grid-template-columns: 240px 1fr 280px;
  grid-template-rows: 60px 1fr 40px;
  grid-template-areas: "header header header"
    "sidebar main widgets"
    "footer footer footer";
}
```

### 시맨틱 태그로 바꾸기

전부 `div`로만 되어 있던 걸 `header`, `main`, `aside`, `footer`로 바꾸는 것도 이번 과제였다. 처음엔 "class 이름 다 있는데 굳이 왜?" 싶었는데, 생각해보니 `div`는 그냥 "의미 없는 상자"라 브라우저나 스크린리더, 검색엔진이 class 이름(`class="header"`)까지 읽어서 "여기가 헤더구나"라고 이해하는 게 아니었다. `<header>`, `<main>` 같은 태그 자체가 "나는 이런 역할이야"라는 의미를 갖고 있는 거고, 눈에 보이는 결과는 CSS만 똑같이 주면 `div`나 시맨틱 태그나 똑같이 나온다. 차이는 화면 뒤쪽, 접근성/SEO/가독성 쪽에서 나는 거였다.

```html
<!-- Before -->
<div class="sidebar">...</div>
<div class="main-content">...</div>
<div class="widgets">...</div>
<div class="footer">...</div>

<!-- After -->
<aside class="sidebar">...</aside>
<main class="main-content">...</main>
<aside class="widgets">...</aside>
<footer class="footer">...</footer>
```

sidebar랑 widgets 둘 다 `aside`를 썼는데, `header`/`main`이랑 다르게 `aside`, `section`, `article` 같은 태그는 한 문서 안에 여러 개 있어도 괜찮다는 것도 이번에 알았다.

### 헤더 스펙이 스크린샷이랑 안 맞는 문제

다 완성하고 브라우저로 열어봤는데, 헤더가 화면 왼쪽 680px까지만 채워지고 오른쪽은 텅 비어 있었다. PDF 문제지에는 분명 `.header`에 `max-width: 680px`를 주라고 적혀 있었는데, 문제지 맨 앞장에 있는 "최종 완성 스크린샷"을 다시 보니까 거긴 헤더가 화면 끝까지 꽉 차 있었다. 텍스트 스펙이랑 결과 이미지가 서로 다른 말을 하고 있었던 것.

곰곰이 생각해보니 이전 문제(아티클 카드)의 `.article-box` 스펙에서 썼던 "최대 크기(max-width)는 680px" 문구가 이번 문제로 복사되면서 실수로 안 지워진 것 같았다. `grid-area: header`로 3개 컬럼을 가로지르는 자리를 미리 잡아놨는데, 거기에 `max-width`로 제한을 걸어버리면 애초에 grid-area를 풀너비로 잡은 의도 자체랑도 안 맞았다.

![max-width: 680px 때문에 헤더가 화면 왼쪽 일부만 채우고 나머지는 빈 배경으로 남은 화면](/images/html_css/dashboard-header-maxwidth-bug.png)

```css
.header {
  grid-area: header;
  color: #ffffff;
  display: flex;
  z-index: 10;
  /* PDF 텍스트 스펙에는 max-width: 680px로 정의하라고 되어있으나,
     최종 결과 스크린샷에서는 header가 풀와이드로 퍼져있어 
     스펙 오류로 판단, 주석 처리함. */
  /* max-width : 680px; */
  padding: 0 24px 0 24px;
  background-color: #0f172a;
  align-items: center;
  justify-content: space-between;
}
```

이런 것도 실무에서 종종 있을 것 같다는 생각이 들었다. 기획서/스펙 문서랑 실제 디자인 시안이 다를 때, 텍스트보다는 최종 결과물(눈에 보이는 정답 이미지) 쪽을 신뢰하는 게 맞는 것 같다. 대신 임의로 지우지 않고 왜 그렇게 판단했는지 주석으로 남겨두는 게 나중에 혼란을 줄여줄 것 같다.

> ✅ **핵심**: `grid-template-areas`로 영역을 미리 정의하면 grid-area 이름만으로 배치할 수 있다. CSS 오타는 콘솔 에러 없이 조용히 무시되니 값 하나하나 의심하며 확인하는 습관이 필요하다. 시맨틱 태그는 눈에 보이는 결과가 아니라 "의미 전달"을 위한 것.

오타 고치고 header 스펙 문제까지 정리하고 나니 최종적으로 이런 화면이 나왔다.

![대시보드 최종 완성 화면 - 헤더 풀와이드, 사이드바/메인/위젯 3단 grid, 하단 footer까지 정상 배치됨](/images/html_css/dashboard-final-result.png)

---

## 💡 오늘 배운 것 정리

세 문제 풀면서 계속 반복된 질문이 결국 "이 요소, display 뭐 써야 하지?" 였다. 오늘까지 종합해보면:

- 📇 나란히 배치 + 크기·여백 직접 지정 필요 → `inline-block`
- 📰 `float`가 걸린 요소는 자동 blockify 되지만, 의도 표현을 위해 `display: block`은 명시하는 게 좋은 습관
- 🖥️ CSS 오타(`gird`, `1ft` 같은)는 에러 없이 조용히 무시되니 레이아웃이 이상하면 속성값부터 의심할 것
- 🖥️ 시맨틱 태그는 결과 화면이 아니라 접근성/SEO/가독성을 위한 것
- 🖥️ 스펙 문서와 결과 이미지가 충돌하면, 왜 그렇게 판단했는지 근거를 주석으로 남기고 최종 결과물을 우선한다

**다음에 실전 적용할 것**

1. display 고민될 때마다 "나란히? 혼자? 크기 지정 필요한가?" 3단계로 판단하기
2. CSS 값 이상하게 안 먹히면 오타부터 의심 (VSCode CSS 자동완성/검증 확장 찾아서 설치해보기)
3. 스펙이랑 결과물이 충돌하면 임의로 고치지 말고 근거 남기고 판단하기
