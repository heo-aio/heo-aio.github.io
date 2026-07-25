+++
title = "[HTML/CSS] 웹사이트의 정보와 디자인, 기초부터 정리"
date = 2026-07-25
draft = false
tags = ["HTML", "CSS", "웹개발", "프론트엔드"]
categories = ["dev"]
math = false
+++

오늘부터 HTML/CSS 기초 수업이 시작됐다. 백엔드나 AI 쪽으로 가더라도 결국 웹 서비스를 만드는 이상 HTML/CSS 없이는 아무것도 못 만든다는 게 첫 시간 강조 포인트였다. 1장이라 그런지 다루는 태그 수가 꽤 많았는데, 오늘 배운 걸 정리하지 않으면 태그 이름들이 금방 섞일 것 같아서 정리해둔다.

---

## 🧱 웹을 구성하는 세 가지 요소

프로그래밍은 결국 컴퓨터(브라우저)와 소통하는 방법이고, 웹 개발에서는 그 소통을 HTML, CSS, JavaScript 세 가지 언어가 나눠서 담당한다.

| 언어 | 역할 |
|---|---|
| HTML | 정보 또는 설계도 |
| CSS | 디자인 또는 스타일링 |
| JavaScript | 기능과 효과 |

세 언어를 층으로 쌓아서 생각하면 이해가 편했다. 가장 아래에 HTML이 구조를 잡고, 그 위에 CSS가 스타일을 입히고, 맨 위에서 JavaScript가 동작을 붙이는 순서. 구조 없이 스타일을 입힐 수 없듯, HTML을 먼저 잡고 CSS로 꾸미는 순서가 자연스러운 이유가 있었다.

![HTML, CSS, JavaScript가 층층이 쌓여 웹페이지를 이루는 구조도](/images/html_css/web-layer-roles.png)

웹사이트를 만들 때 고려해야 할 세 가지도 짚고 넘어갔다.

- **웹 표준**: 웹사이트를 작성할 때 따라야 하는 공식 표준이나 기술 규격
- **웹 접근성**: 장애 여부와 상관없이 모두가 웹사이트를 이용할 수 있게 하는 방식
- **크로스 브라우징**: 모든 브라우저 또는 기기에서 사이트가 제대로 작동하도록 하는 기법

---

## 🏷️ HTML 기본 태그

HTML은 Hyper Text Markup Language의 약자로, 웹사이트에서 눈에 보이는 정보나 특정 구역을 설정할 때 쓰는 언어다.

### 태그 구성 요소

```
<열린태그 속성 = "속성값"> 컨텐츠 </닫힌태그>
```

- **태그명**: HTML이 갖고 있는 고유의 기능. `<열린태그></닫힌태그>` 형태로 입력한다.
- **컨텐츠**: 열린 태그와 닫힌 태그 사이에 있는 내용물.
- **속성**: HTML 태그가 갖고 있는 추가 정보.
- **속성값**: 어떤 역할을 수행할지 구체적인 명령을 내리는 것.

### HTML 문서의 기본 구조

```html
<!DOCTYPE html> <!-- HTML5 문서 선언 -->
<html> <!-- HTML 문서의 시작과 끝 -->
<head> <!-- 문서와 관련된 요약 정보 정리 -->
    <meta charset="UTF-8"> <!-- 문자 코드 -->
    <title>웹프로그래밍</title> <!-- 웹사이트 제목 -->
</head>
<body> <!-- 웹사이트 내용 -->
</body>
</html>
```

`<html>` 태그 안에 `<head>`와 `<body>`가 중첩되는 구조라고 생각하면 헷갈리지 않는다.

![html 태그 안에 head와 body가 중첩되는 구조도](/images/html_css/html-document-structure.png)

- `<!DOCTYPE html>`: HTML5라는 신조어로 문서를 선언하는 태그
- `<html> … </html>`: HTML 문서의 시작과 끝. 모든 태그는 이 안에 입력
- `<head> … </head>`: 웹사이트의 간단한 요약 정보를 담는 영역. 웹사이트에서 노출되지 않는 정보
- `<body> … </body>`: 웹사이트에서 눈에 보이는 정보를 담는 영역
- `<meta charset="UTF-8">`: character setting의 약자로, 문자를 깨짐 없이 표시하겠다는 의미
- `<title> … </title>`: 웹사이트 탭에 나타나는 제목을 적는 태그

참고로 HTML도 시간이 지나면서 자주 안 쓰이는 태그는 사라지고 새로운 태그가 등장한다고 한다. `<acronym>`, `<applet>`, `<basefont>` 같은 태그가 HTML5에서는 지원되지 않는 게 그 예시.

### 자주 쓰는 기본 태그들

```html
<!-- 링크 이동 -->
<a href="https://www.naver.com" target="_blank"> 네이버 </a>

<!-- 이미지 삽입, 닫힌 태그 없음 -->
<img src="logo.png" alt="회사로고">

<!-- 제목/부제목 -->
<h1>Hello World</h1>
<h2>Hello World</h2>
<h3>Hello World</h3>

<!-- 본문 -->
<p>Nice to meet you</p>

<!-- 순서 있는 리스트 -->
<ol>
    <li>메뉴1</li>
    <li>메뉴2</li>
</ol>

<!-- 순서 없는 리스트 -->
<ul>
    <li>메뉴1</li>
    <li>메뉴2</li>
</ul>
```

몇 가지 헷갈릴 만한 포인트를 정리하면:

- `<a>`의 `target` 속성: `"_blank"`는 새 탭으로, `"_self"`는 현재 탭에서 전환(디폴트 값)
- `<img>`는 `src`(파일 경로)와 `alt`(이미지 출력 실패 시 대체 텍스트) 속성을 가짐
- `<h1>` ~ `<h6>`는 숫자가 클수록 폰트 사이즈가 작아짐. 즉 숫자는 정보의 중요도를 나타내고, `<h1>`은 가장 중요한 정보이므로 하나의 문서에서 한 번만 사용
- `<ol>`은 Ordered list, `<ul>`은 Unordered list의 약자. `<li>`는 이 둘의 항목을 나열할 때 공통으로 사용

---

## 📚 구조를 잡을 때 사용하는 태그

여기서부터는 페이지 전체의 큰 틀을 잡는 태그들이다. 실제 웹페이지 레이아웃으로 와이어프레임을 그려보면 각 태그가 어느 영역을 담당하는지 한눈에 들어온다.

![header, nav, main, article, footer 태그로 이루어진 웹페이지 와이어프레임](/images/html_css/page-layout-wireframe.png)

```html
<header> <!-- 상단 영역 -->
    <img src="logo.png" alt="로고">
    <nav> <!-- 메뉴 버튼 영역 -->
        <ul>
            <li>홈</li>
            <li>전체 목록</li>
        </ul>
    </nav>
</header>

<main role="main"> <!-- 본문 영역 -->
    <article> <!-- 정보 영역 -->
        <h1>…</h1>
        ...
    </article>
</main>

<footer> <!-- 하단 영역 -->
    <div>
        <p>주소: 대전광역시 유성구 문지로 193 KAIST</p>
    </div>
</footer>
```

- `<header>`: 웹사이트의 머리글을 담는 공간
- `<nav>`: 메뉴 버튼을 담는 공간(navigation). `<ul>`, `<li>`, `<a>`와 함께 쓰임
- `<main>`: 문서의 주요 내용을 담는 태그. Internet Explorer는 지원하지 않으므로 `role="main"` 속성을 꼭 넣어야 함
- `<article>`: 문서의 주요 정보를 담고 구역을 설정하는 태그. 태그 내에 구역을 대표하는 `<h#>` 태그가 있어야 함. 위 그림처럼 `<main>` 하나 안에 `<article>`이 여러 개 들어갈 수 있음
- `<footer>`: 가장 하단에 들어가는 정보를 표기할 때 사용
- `<div>`: 임의의 공간을 만들 때 사용하는, 특별한 의미 없는 범용 컨테이너

---

## 📐 HTML 태그의 두 가지 성격: Block과 Inline

태그는 출력될 때 성격이 두 가지로 나뉜다. 이 둘을 구분 짓는 특징은 줄바꿈 현상, 가로·세로 정렬, 상하 배치 가능 여부다.

![Block 요소는 세로로 쌓이고 Inline 요소는 한 줄에 나열되는 비교도](/images/html_css/block-vs-inline.png)

```html
<!-- Block 요소: p 태그 -->
<p>Hello Elice</p>
<p>Hello Elice</p>
<!-- y축 정렬로 출력, 줄바꿈 O, 공간 생성/상하 배치 가능 -->

<!-- Inline 요소: a 태그 -->
<a>Bye Elice</a>
<a>Bye Elice</a>
<!-- x축 정렬로 출력(한 줄), 공간 생성/상하 배치 불가능 -->
```

`<p>`, `<div>`, `<h1>` 같은 태그는 Block 요소, `<a>`, `<img>` 같은 태그는 Inline 요소에 해당한다.

---

## 🎨 CSS, 정보와 디자인의 분리

CSS는 Cascading Style Sheet의 약자로, HTML로 작성된 정보를 꾸며주는 언어다. 정보(HTML)와 디자인(CSS)을 분리해서 관리한다는 게 핵심 포인트.

```
선택자 { 속성 : 속성값; }
```

- **선택자**: 디자인을 적용할 HTML 영역
- **속성**: 어떤 디자인을 적용할지 정의
- **속성값**: 어떤 역할을 수행할지 구체적으로 명령. 세미콜론(`;`) 필수 입력

### CSS를 연동하는 세 가지 방법

```html
<!-- 1. Inline: 태그 안에 직접 -->
<h1 style="color: red;"> coding 101 </h1>

<!-- 2. Internal: style 태그 안에 -->
<head>
    <style>
        h1 { background-color: yellow; }
    </style>
</head>

<!-- 3. External: link 태그로 불러오기 -->
<head>
    <link rel="stylesheet" href="style.css">
</head>
```

External 방식은 html, css 문서를 각각 따로 관리해서 상대적으로 가독성이 높고 유지보수가 쉽다는 장점이 있다. 실무에서는 거의 이 방식을 쓴다고.

---

## 🎯 CSS 선택자

선택자는 결국 "HTML의 어떤 요소에 CSS를 적용할 것인가"를 정하는 역할이다.

![CSS 규칙의 선택자가 화살표로 특정 HTML 요소를 가리키는 개념도](/images/html_css/css-selector-concept.png)

```html
<!-- Type Selector: 특정 태그에 적용 -->
<h2>Type Hello World</h2>
<style> h2 { color: red; } </style>

<!-- Class Selector: 클래스 이름으로 적용 -->
<h2 class="coding">Class Hello World</h2>
<style> .coding { color: blue; } </style>

<!-- ID Selector: ID로 적용 -->
<h2 id="coding">ID Hello World</h2>
<style> #coding { color: green; } </style>
```

타입 / 클래스 / 아이디, 이 세 가지가 CSS 선택자의 기본이다.

---

## 👪 부모 자식 관계

CSS 선택자를 더 구체적으로 쓰고 싶을 때는 부모 태그를 함께 표기하면 된다.

```html
<header>
    <h1>Header h1</h1>
    <p>Header p</p>
</header>

<footer>
    <h1>Footer h1</h1>
    <p>Footer p</p>
</footer>
```

여기서 `<header>`와 `<h1>`, `<p>`는 부모 자식 관계, `<h1>`과 `<p>`는 형제 관계다. 만약 `<header>` 안의 `<p>`에만 스타일을 주고 싶다면 아래처럼 부모를 구체적으로 표기하면 된다.

```css
header p { color: green; }
```

이렇게 하면 `<footer>` 안의 `<p>`에는 영향을 주지 않고 `<header>` 안의 `<p>`에만 스타일이 적용된다. 원하는 영역에만 정확하게 CSS를 먹이고 싶을 때 유용한 방법.

---

## 🌊 캐스케이딩 — CSS 우선순위

같은 요소에 여러 스타일이 겹칠 때 어떤 게 최종 적용될지를 결정하는 게 캐스케이딩이다. 우선순위를 결정하는 요소는 세 가지.

![CSS 우선순위를 결정하는 순서, 디테일, 선택자 세 단계 흐름도](/images/html_css/css-cascading-priority.png)

**1. 순서** — 나중에 적용한 속성값의 우선순위가 높다.

```css
p { color: red; }
p { color: blue; } /* 이게 최종 적용 */
```

**2. 디테일** — 더 구체적으로 작성된 선택자의 우선순위가 높다.

```css
header p { color: red; } /* 더 구체적이므로 이게 적용 */
p { color: blue; }
```

**3. 선택자** — `style > id > class > type` 순으로 우선순위가 높다.

```html
<h3 style="color: pink" id="color" class="color"> color </h3>
```

```css
#color { color: blue; }
.color { color: red; }
h3 { color: green; }
```

이 경우 인라인 style이 가장 우선순위가 높으므로 최종 색상은 pink가 된다. 세 가지 요소를 순서대로 기억해두면 나중에 "왜 내가 준 스타일이 안 먹지?" 할 때 디버깅이 훨씬 빨라질 것 같다.

---

## 🖌️ CSS 주요 속성

마지막으로 오늘 배운 주요 속성들.

```css
.paragraph {
    /* 크기 */
    width: 500px;   /* 넓이. 고정값(px), 가변값(%) */
    height: 500px;  /* 높이 */

    /* 글자 */
    font-size: 50px;
    font-family: Arial, sans-serif; /* 입력 순서대로 우선순위 적용, sans-serif가 디폴트 */
    font-style: italic;
    font-weight: bold; /* 100~900 사이 숫자도 가능 */

    /* 테두리 */
    border-style: solid; /* 실선: solid, 점선: dotted */
    border-width: 10px;
    border-color: red;
    /* border: solid 10px red; 로 한 줄에 이어 쓸 수도 있음 (쉼표 없이 띄어쓰기만) */

    /* 배경 */
    background-color: yellow;
    background-image: url(이미지 경로);
    background-repeat: no-repeat; /* x축 반복: repeat-x, y축 반복: repeat-y */
    background-position: left;    /* top, bottom, center, left, right 등 */
}
```

속성값 여러 개를 한 줄로 이어 쓸 수 있다는 게(`border`, `background` 단축 속성) 새로 알게 된 부분. 다만 이때는 쉼표 없이 띄어쓰기만 한다는 걸 자꾸 까먹을 것 같아서 따로 적어둔다.

---

## 💡 정리하면

오늘 배운 걸 한 줄로 요약하면, HTML은 정보의 구조를 잡고 CSS는 그 위에 디자인을 입힌다는 것. 태그 하나하나는 단순했는데, `<header>`-`<main>`-`<footer>` 같은 구조 태그와 캐스케이딩 우선순위 규칙을 조합하면 실제 웹페이지 레이아웃이 어떻게 짜이는지 감이 잡히기 시작했다. 다음 시간에는 CSS 레이아웃으로 실제 웹페이지를 배치해본다고 하니, 오늘 정리한 선택자랑 캐스케이딩 개념은 확실히 익히고 넘어가야겠다.
