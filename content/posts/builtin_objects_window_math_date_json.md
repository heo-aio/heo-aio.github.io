+++
title = "[JavaScript] 내장 객체 정리 - window, Math, Date, JSON"
date = 2026-08-06T21:00:00+09:00
draft = false
tags = ["JavaScript", "내장객체", "Date", "JSON", "window"]
categories = ["dev"]
math = false
+++

이번엔 내장 객체(built-in object) 실습이었다. `window`, `Math`, `Date`, `JSON` 네 가지를 다뤘는데, 각각 독립적인 내용처럼 보여도 실습 코드를 다시 보니 **"자바스크립트가 브라우저/시간/데이터를 다루기 위해 기본으로 들고 있는 도구 상자"** 정도로 묶어서 이해하면 편했다. 특히 `13_page.html`에는 실제로 실행하면 에러가 나는 코드가 그대로 남아있었는데, 이걸 고치려고 하다가 스코프 개념을 다시 복습하게 됐다. 그 부분도 같이 정리해둔다.

**목차**
1. [window 객체 - 브라우저 창 자체를 다루기](#-window-객체---브라우저-창-자체를-다루기)
2. [Math 객체 - 숫자 처리](#-math-객체---숫자-처리)
3. [Date 객체 - 날짜와 시간 다루기](#-date-객체---날짜와-시간-다루기)
4. [JSON 객체 - 직렬화와 역직렬화](#-json-객체---직렬화와-역직렬화)
5. [감이 안 왔던 부분, 다시 짚어보기](#-감이-안-왔던-부분-다시-짚어보기)
6. [정리하면](#-정리하면)

---

## 🪟 window 객체 - 브라우저 창 자체를 다루기

```js
console.log(window);                                   // 브라우저 창 자체를 나타내는 최상위 객체
console.log(window.innerWidth + ' / ' + window.innerHeight); // 현재 창의 가로/세로 크기

let win;
function myWinOpen(){
    win = window.open(
        'https://www.naver.com', '',
        'width=400, height=400, top=100, left=500, resizeble=no, scrollbar=no'
    );
}
function myWinClose(){
    win.close();
}
```

여태 `document`만 계속 다뤄서 `window`는 좀 낯설었는데, 정확히는 `document`도 `window` 객체 안에 들어있는 하위 속성 중 하나다. `window.open(url, 이름, 옵션문자열)`로 새 창을 띄울 수 있는데, 여기서 헷갈렸던 부분은 **`width`/`height`를 안 주면 새 창이 아니라 새 탭으로 열린다**는 점이었다 - 옵션 문자열에 크기 정보가 있어야 브라우저가 "진짜 팝업 창"으로 인식하는 것 같다.

![window.open()의 url/이름/옵션 파라미터와 새 창의 위치·크기 관계](/images/js_dom/window_open_params.svg)

`top`, `left`는 모니터 좌상단(0,0)을 기준으로 한 좌표라는 것도 그림으로 그려보니 확실해졌다. 그리고 `myWinOpen()`에서 연 창을 `win`이라는 **바깥(전역) 변수**에 저장해두기 때문에, `myWinClose()`에서 그 변수를 통해 `win.close()`를 호출해 닫을 수 있다는 구조도 짚어둘 만하다 - 만약 `win`을 함수 지역변수로 선언했다면 `myWinClose()`에서는 그 창을 참조할 방법이 없었을 것이다.

> ✅ **핵심**: `window.open()`에 크기 옵션이 없으면 새 탭으로 열린다. 열린 창을 나중에 제어(닫기 등)하려면 그 참조를 바깥 스코프의 변수에 저장해둬야 한다.

---

## 🔢 Math 객체 - 숫자 처리

```js
console.log(Math); // min, max, ceil, floor, round, random ...

let num = 314.56789;
console.log(num, '->', num.toFixed(2)); // 314.56789 -> "314.57"
```

`Math`는 별도로 `new`할 필요 없이 바로 `Math.random()`, `Math.floor()`처럼 갖다 쓰는 정적 객체라는 걸 다시 짚었다. `toFixed(2)`는 소수점 이하를 지정한 자릿수로 반올림해주는데, **결과가 숫자가 아니라 문자열로 반환된다**는 점이 실습 주석에 강조되어 있었다 - 이후에 이 값으로 다시 연산을 하려면 `Number()`나 `parseFloat()`로 되돌려야 한다는 걸 잊으면 안 될 것 같다.

> ✅ **핵심**: `Math`는 인스턴스를 만들 필요 없이 바로 쓰는 정적 객체다. `toFixed()`의 반환값은 숫자가 아니라 문자열이라는 점을 주의해야 한다.

---

## 📅 Date 객체 - 날짜와 시간 다루기

```js
const date = new Date();
let year = date.getFullYear();   // 연도
let month = date.getMonth() + 1; // 월 (0~11이라서 +1 필요!)
let day = date.getDate();        // 일 (1~31)
let hours = date.getHours();     // 시 (0~23)
let minutes = date.getMinutes(); // 분 (0~59)
let seconds = date.getSeconds(); // 초 (0~59)

let str_month = month.toString().padStart(2, '0'); // 8 -> '08'
```

`Date` 메서드들은 이름이 비슷비슷해서 처음엔 뭐가 뭔지 헷갈렸는데, 실제 날짜 문자열에 각 메서드를 대응시켜서 그려보니 명확해졌다.

![Date 객체의 get 메서드들이 날짜/시간 문자열의 어느 부분을 가져오는지 매핑](/images/js_dom/date_methods_map.svg)

가장 조심해야 할 부분은 **`getMonth()`가 0~11 범위**라는 것이다. 1월이 `0`, 12월이 `11`로 나온다 - 배열 인덱스처럼 0부터 시작한다고 생각하면 왜 `+1`을 붙였는지 이해가 된다. 그리고 `padStart(2, '0')`는 "두 자리로 맞추고 모자란 자리는 `'0'`으로 채워라"는 뜻이라, `8`을 `'08'`로 통일된 형태의 날짜 문자열을 만들 때 자주 쓰는 패턴이라는 걸 알아뒀다.

> ✅ **핵심**: `getMonth()`는 0~11 범위라서 실제 월을 쓰려면 `+1`을 해야 한다. `padStart(2, '0')`로 한 자리 숫자도 두 자리처럼 통일된 형태로 맞출 수 있다.

---

## 🔄 JSON 객체 - 직렬화와 역직렬화

```js
let json = { name: 'Daniel', age: 12, message: 'Hello, My name is Daniel, Lee' };

let str_json = JSON.stringify(json); // 객체 -> 문자열 (직렬화)
console.log(str_json); // '{"name":"Daniel","age":12,"message":"Hello..."}'

// (주석 처리된 정석 방법)
// let json_1 = JSON.parse(str_json); // 문자열 -> 객체 (역직렬화)
// json_1.gender = 'male';
// json_1.married = false;
// let json_2 = JSON.stringify(json_1);
```

`JSON.stringify`/`JSON.parse`가 서로 반대 방향으로 객체와 문자열을 오간다는 걸 그림으로 정리해봤다.

![JSON.stringify(직렬화)와 JSON.parse(역직렬화)가 객체와 문자열 사이를 오가는 관계](/images/js_dom/json_stringify_parse.svg)

여기서 흥미로웠던 지점은, 실습 코드의 주석에는 "정석대로" `JSON.parse`로 객체로 되돌린 다음 속성을 추가하고 다시 `JSON.stringify`하는 방법이 적혀 있었는데, **실제 실행된 코드는 그 방법을 안 쓰고 문자열의 중괄호(`{`, `}`)를 직접 `replace`로 뗐다 붙였다 하면서 문자열을 이어 붙이는 방식**이었다는 것이다.

```js
let values = str_json.replace('{', '').replace('}', '');
function add(){
    let key = document.querySelector('input[name = "key"]').value;
    let val = document.querySelector('input[name = "val"]').value;
    values = values.replace('{', '').replace('}', '');
    values += `,"${key}":"${val}"`;
    values = `{${values}}`;
    let result = JSON.parse(values);
}
```

이 방식도 결과는 나오지만, 문자열을 직접 조작하는 거라 따옴표나 콤마 위치가 하나만 틀려도 `JSON.parse`에서 바로 에러가 나는 위험이 있다. **객체 상태로 바꾸는 것(`JSON.parse`) → 객체에 속성 추가 → 다시 문자열로(`JSON.stringify`)** 하는 주석 처리된 방법이 훨씬 안전하다는 걸, 두 방식을 나란히 놓고 보니 확실히 알겠다.

> ✅ **핵심**: 객체를 수정하고 싶다면 문자열을 직접 조작하지 말고 `JSON.parse`로 객체로 되돌린 뒤 수정하고 다시 `JSON.stringify`하는 게 안전하다.

---

## 🤔 감이 안 왔던 부분, 다시 짚어보기

- **13_page.html은 왜 실행하면 에러가 날까?** → 이 파일은 실습 중 미완성/버그 상태로 남아있던 코드였다. `setInterval(() => { ... })`의 화살표 함수 콜백 **안에서** `let year`, `let str_month` 등을 선언해놓고, 콜백 **밖에서** `console.log(`${year}-${str_month}...`)`로 그 값을 쓰려고 한 게 문제였다. 지난 ES6 정리 글의 스코프 체인 개념 그대로, `year`는 콜백 함수 블록 안에서만 존재하는 지역 변수라서 그 블록을 벗어난 순간 `ReferenceError: year is not defined`가 난다.

![13_page.html의 스코프 버그 - 콜백 안에서 선언한 변수를 콜백 밖에서 참조하면 에러가 난다](/images/js_dom/setinterval_scope_bug.svg)

  고치려면 `year`처럼 바깥에서도 쓸 변수는 콜백 **바깥**에서 먼저 `let`으로 선언해두고, 콜백 안에서는 값만 대입(`year = date.getFullYear()`)하는 형태로 바꿔야 한다. 게다가 `setInterval`은 반복 실행되는데 바깥의 `console.log`는 딱 한 번만 실행되니, 애초에 "매번 갱신되는 시계"를 만들고 싶었다면 `console.log`와 화면 업데이트 코드 자체가 콜백 안에 있어야 한다는 구조적인 문제도 있었다. 다음엔 이 파일을 직접 고쳐서 "1초마다 갱신되는 시계"로 완성해보고 싶다.
- **`window`와 `document`는 정확히 무슨 관계인가?** → `window`가 브라우저 창 자체를 나타내는 최상위 전역 객체이고, `document`는 그 안에 있는 "현재 페이지의 HTML 문서"를 나타내는 하위 객체다. 그동안 `document.getElementById`처럼 `document`만 계속 써왔는데, 사실 전역 스코프에서 선언한 변수나 함수도 결국 `window` 객체의 속성으로 들어간다는 걸 이번에 알았다. `console.log(window)`를 직접 펼쳐서 그 안에 뭐가 들어있는지 다음에 찬찬히 살펴봐야겠다.

---

## 💡 정리하면

내장 객체 4개(`window`, `Math`, `Date`, `JSON`)는 겉보기엔 서로 다른 주제 같지만, **"브라우저가 기본으로 제공하는, 자주 필요한 기능을 이미 객체 형태로 만들어둔 것"** 이라는 공통점으로 묶을 수 있었다. 그리고 이번 실습에서 제일 값진 부분은 사실 정상 동작하는 코드보다, **의도적으로(혹은 실수로) 남겨진 버그 코드(13_page)와 비효율적인 방식(15_page의 문자열 조작)** 이었다 - 정답 코드만 보는 것보다 "왜 이게 안 되는지"를 스스로 짚어보는 게 훨씬 오래 기억에 남을 것 같다.

**오늘의 핵심 5가지**

- 🪟 `window.open()`은 크기 옵션이 없으면 새 탭으로 열리고, 연 창을 제어하려면 참조를 바깥 변수에 저장해야 한다
- 🔢 `Math`는 인스턴스 없이 바로 쓰는 정적 객체이며, `toFixed()`의 결과는 문자열이다
- 📅 `getMonth()`는 0~11 범위라 `+1`이 필요하고, `padStart(2,'0')`로 자릿수를 통일한다
- 🔄 `JSON.stringify`(직렬화)/`JSON.parse`(역직렬화)로 객체와 문자열을 오가며, 수정은 객체 상태에서 하는 게 안전하다
- 🐛 `setInterval` 콜백 안에서 선언한 변수는 콜백 밖에서 못 쓴다 - 스코프 체인 개념이 실전 버그로 그대로 이어진다

**다음에 확인해볼 것**

1. 13_page.html을 직접 고쳐서 "1초마다 갱신되는 실시간 시계"로 완성해보기 (변수를 콜백 바깥에서 선언 + 화면 업데이트도 콜백 안으로)
2. `console.log(window)`를 펼쳐서 `document`, `location`, `history` 등이 실제로 `window`의 하위 속성인지 직접 확인해보기
3. `JSON.parse`/`JSON.stringify` 방식으로 15_page.html의 문자열 조작 코드를 리팩터링해보기
