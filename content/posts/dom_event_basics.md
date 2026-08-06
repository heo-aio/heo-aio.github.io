+++
title = "[JavaScript] DOM과 이벤트 1 - 선택자, 속성 제어, addEventListener, this vs target까지"
date = 2026-08-06T14:00:00+09:00
draft = false
tags = ["JavaScript", "DOM", "이벤트", "addEventListener", "웹개발"]
categories = ["dev"]
math = false
+++

이번엔 05장, DOM과 이벤트 실습이었다. 문법보다는 "DOM API를 어떻게 조합해서 실제로 화면을 조작하는가"에 가까운 내용이라, 개념 하나하나보다는 **실습 코드에서 반복되는 패턴**을 잡는 게 중요해 보였다. 특히 `this`와 `evt.target`을 헷갈렸던 부분, `removeEventListener`가 왜 안 먹었는지 삽질했던 부분은 꼭 짚고 넘어가고 싶었다.

**목차**
1. [DOM 요소 선택하기 - 4가지 선택자](#-dom-요소-선택하기---4가지-선택자)
2. [요소 속성 제어하기 - innerHTML, style, setAttribute](#-요소-속성-제어하기---innerhtml-style-setattribute)
3. [DOM 요소 새로 만들기 - createElement, appendChild](#-dom-요소-새로-만들기---createelement-appendchild)
4. [인라인 이벤트 vs addEventListener](#-인라인-이벤트-vs-addeventlistener)
5. [this vs evt.target - 헷갈리는 지점](#-this-vs-evttarget---헷갈리는-지점)
6. [이벤트 등록/해제 - addEventListener / removeEventListener](#-이벤트-등록해제---addeventlistener--removeeventlistener)
7. [form 이벤트와 preventDefault](#-form-이벤트와-preventdefault)
8. [실전 적용 - 색상 순환, 다중 요소 이벤트 바인딩](#-실전-적용---색상-순환-다중-요소-이벤트-바인딩)
9. [감이 안 왔던 부분, 다시 짚어보기](#-감이-안-왔던-부분-다시-짚어보기)
10. [정리하면](#-정리하면)

---

## 🔎 DOM 요소 선택하기 - 4가지 선택자

```js
var elemTag = document.getElementsByTagName('h1');       // HTMLCollection (여러개, live)
var elemClass = document.getElementsByClassName('head2'); // HTMLCollection (여러개, live)
var elemId = document.getElementById('head3');            // 단일 Element

console.log(elemId);     // 태그 자체가 그대로 출력
console.log([elemId]);   // 배열로 감싸면 배열 형태로 출력 - "단일이냐 배열이냐"의 차이가 드러남

console.log(document.querySelector('.head2'));    // CSS 선택자로 첫 번째 하나만
console.log(document.querySelectorAll('.head2')); // CSS 선택자로 전체 (NodeList)
```

![DOM 선택자 4종 비교 - 단일 요소 반환 vs 컬렉션 반환, live vs static](/images/js_dom/dom_selectors_map.svg)

여기서 제일 헷갈렸던 건 "왜 `elemId`는 그냥 찍히는데 `elemTag`는 배열처럼 나오지?"였다. 이름에 **Elements(복수형)**가 들어간 메서드는 여러 개를 담은 컬렉션(HTMLCollection/NodeList)을 반환하고, `getElementById`처럼 단수형이면 요소 하나만 반환한다는 규칙을 알고 나니 이름만 보고도 반환 타입을 예측할 수 있게 됐다. `getElementsByTagName`/`ClassName`은 **live**(DOM이 바뀌면 실시간 반영)라서 `querySelectorAll`의 **static**(호출 시점 스냅샷) 결과와 동작이 다를 수 있다는 것도 기억해둬야 할 부분이다.

> ✅ **핵심**: 이름이 복수형(Elements)이면 컬렉션, 단수형(Element)이면 단일 요소를 반환한다. `getElementsBy...`는 live, `querySelectorAll`은 static이라는 차이도 있다.

---

## 🎨 요소 속성 제어하기 - innerHTML, style, setAttribute

```js
var elem = document.getElementById('p1');
elem.innerHTML = 'dom practcie';           // 내용(HTML) 자체를 바꾸기
elem.style.backgroundColor = 'red';        // 인라인 스타일 직접 수정

var img = document.querySelector('img');
// img.src = 'image4.png';   // 기본 속성은 직접 대입도 되지만 안 먹는 속성도 있음
img.setAttribute('src', 'image3.png');    // 모든 속성을 안전하게 다루는 방법

var input = document.getElementsByTagName('input');
input[0].setAttribute('type', 'button');   // 기본 속성 변경
input[0].value = 'click';                  // value는 대입으로도 바뀜

// 사용자 정의 속성(my-attr)은 getAttribute로만 읽을 수 있다
var tags = document.getElementsByTagName('a');
for (var tag of tags) {
    var attr = tag.getAttribute('my-attr');
    tag.innerHTML = '<h1>' + attr.toUpperCase() + '</h1>';
}
```

`img.src = '...'`처럼 **직접 대입**하는 방식과 `setAttribute()`로 바꾸는 방식이 둘 다 존재해서 처음엔 뭘 써야 하나 싶었는데, 실습 주석에 "직접 속성 지정(안 되는 속성들도 있음)"이라고 적혀 있던 게 힌트였다. `id`, `src`, `value`처럼 흔히 쓰는 기본 속성은 `elem.속성 = 값`으로도 되지만, **`my-attr`처럼 브라우저가 모르는 사용자 정의 속성은 `getAttribute`/`setAttribute`로만 안전하게 다룰 수 있다**. 그래서 뭘 써야 할지 애매하면 `setAttribute`/`getAttribute`가 더 범용적인 선택이라는 결론을 내렸다.

> ✅ **핵심**: 기본 속성은 `.속성 = 값`으로도 되지만, 사용자 정의 속성이나 안 먹는 속성이 있을 수 있어 `setAttribute`/`getAttribute`가 더 안전한 방법이다.

---

## 🏗️ DOM 요소 새로 만들기 - createElement, appendChild

```js
var li = document.createElement('li');
li.innerHTML = '새로운 아이템';
var ul = document.getElementById('list');
ul.appendChild(li);

var input = document.createElement('input');
input.setAttribute('type', 'submit');
input.setAttribute('value', '로그인');
document.getElementsByTagName('body')[0].appendChild(input);
```

패턴이 명확했다: **① `createElement`로 태그를 메모리상에 만들고 → ② 필요한 속성/내용을 채운 다음 → ③ `appendChild`로 실제 문서(DOM 트리)에 붙인다.** ②까지만 하고 ③을 안 하면 화면에 아무것도 안 보인다는 걸(만들기만 하고 문서에 안 붙였을 때) 직접 콘솔로 확인해봐야 확실히 체감될 것 같다.

> ✅ **핵심**: 요소 생성은 "만들기(createElement) → 채우기(innerHTML/setAttribute) → 붙이기(appendChild)" 3단계로 이루어진다.

---

## 🖱️ 인라인 이벤트 vs addEventListener

```html
<!-- 인라인 이벤트: 태그 속성에 직접 함수 호출을 적는 방식 -->
<div onmouseover="mouseEvt('over')" onmouseout="mouseEvt('out')"></div>
<input type="button" onclick="getDay()" value="날짜 추출"/>
<input type="text" onkeydown="going(event)" value=""/>
<select onchange="selectOne(this)">...</select>
```

```js
// addEventListener 방식: JS 코드에서 요소를 찾아 이벤트를 "붙인다"
var zone = document.getElementById('eventZone');
zone.addEventListener('click', callBack);

zone.addEventListener('dblclick', function(evt){       // 익명함수 - 여기서만 쓸 거라 이름 불필요
    evt.target.style.backgroundColor = 'yellow';
    evt.target.innerHTML = 'double click';
});
```

두 방식을 나란히 실습해보니 차이가 명확해졌다. **인라인 이벤트**는 HTML 태그 안에 `onclick="함수명()"` 형태로 바로 적는 방식이라 간단하지만, HTML과 JS 로직이 뒤섞인다. **addEventListener**는 JS 쪽에서 요소를 선택한 뒤 이벤트를 붙이는 방식이라 관심사가 분리되고, 한 요소에 같은 이벤트를 여러 개 등록하는 것도 가능하다(인라인은 속성 하나에 함수 하나뿐).

또 하나 짚을 점은 `onchange="selectOne(this)"`처럼 인라인에서는 `this`를 넘겨줘야 하지만, `addEventListener`의 콜백 안에서는 별도로 넘기지 않아도 `this`가 자동으로 "이벤트가 걸린 요소"를 가리킨다는 것이다.

> ✅ **핵심**: 인라인 이벤트는 HTML 속성에 직접 적는 방식, `addEventListener`는 JS에서 요소를 찾아 붙이는 방식. 실무에서는 관심사 분리를 위해 `addEventListener`를 주로 쓴다.

---

## 🎯 this vs evt.target - 헷갈리는 지점

```js
// test2.html - 아이템(.item)을 클릭하면 안의 카운터(.cnt)를 1 증가
var item = document.getElementsByClassName('item');
for (var i = 0; i < item.length; i++){
    item[i].addEventListener('click', function(evt){
        var cur_cnt = this.querySelector('.cnt');  // this: 리스너가 걸린 .item
        var cnt = parseInt(cur_cnt.innerHTML);
        cur_cnt.innerHTML = cnt + 1;
    });
}
```

`.item` 안에는 텍스트를 담은 `div`와 카운트를 담은 `div.cnt`가 같이 들어있다. 그런데 리스너는 `.item`에 걸려 있고, 실제 클릭은 안쪽 텍스트 `div`에서 일어난다. 이때 **`evt.target`은 실제로 클릭된 안쪽 요소**고, **`this`는 이벤트 리스너가 걸려 있는 `.item` 자신**이다. 이 둘을 헷갈리면 `this.querySelector('.cnt')`처럼 "리스너가 걸린 요소 기준으로" 안전하게 하위 요소를 찾는 코드를 못 짜게 된다.

![this(리스너가 걸린 요소) vs evt.target(실제 클릭된 요소) - 이벤트 버블링 예시](/images/js_dom/this_vs_target.svg)

28_page.html의 `going(evt)`/`typing(evt)` 함수에서 `evt.target.value`로 입력값을 가져오는 것도 같은 맥락이다. **`evt.target`은 "이벤트를 당한 당사자"**라는 실습 주석 그대로, 이벤트가 실제로 발생한 요소를 가리킨다.

> ✅ **핵심**: `this`는 리스너가 등록된 요소, `evt.target`은 실제로 이벤트가 발생한 요소다. 자식 요소를 클릭해도 이벤트가 부모까지 전파(버블링)되기 때문에 부모에 걸어둔 리스너 안에서 이 둘이 다를 수 있다.

---

## 🔌 이벤트 등록/해제 - addEventListener / removeEventListener

```js
function printPos(evt){
    document.getElementById('pos').innerHTML = evt.clientX + '/' + evt.clientY;
}
var div = document.querySelector('div');

function evtAdd(){ div.addEventListener('mousemove', printPos); }
function evtDel(){ div.removeEventListener('mousemove', printPos); }
```

37_page.html에서 등록/해제 버튼을 따로 만들어본 이유가 있었다. `removeEventListener`가 제대로 동작하려면 **등록할 때 넘긴 함수와 정확히 같은 참조**를 넘겨야 한다. 그래서 `printPos`처럼 **이름이 있는 함수로 미리 선언**해두고, 등록과 해제 양쪽에서 그 이름을 그대로 재사용해야 한다.

![addEventListener/removeEventListener는 같은 함수 참조가 있어야 제거된다](/images/js_dom/add_remove_listener.svg)

반대로 30_page.html의 `dblclick`, `mouseenter`, `mouseout`은 전부 **익명함수**로 등록했는데, 이 경우엔 애초에 나중에 지울 계획이 없는 "한 번 걸고 쭉 쓰는" 이벤트였기 때문에 괜찮은 선택이었다. 즉 **나중에 지울 가능성이 있는 이벤트라면 반드시 이름 붙인 함수로 등록**해야 한다는 걸 이번에 확실히 알았다.

> ✅ **핵심**: `removeEventListener`는 등록 때와 "똑같은 함수 참조"가 있어야 동작한다. 익명함수로 등록한 이벤트는 나중에 지울 수 없다.

---

## 📮 form 이벤트와 preventDefault

```js
var id = document.querySelector('input[name = "id"]');
var pw = document.querySelector('input[name = "pw"]');
var form = document.getElementById('login-form');

form.addEventListener('submit', function(evt){
    if (id.value == '' || pw.value == ''){
        alert('아이디와 비밀번호를 입력하세요 !');
        evt.preventDefault(); // 폼 전송이라는 "기본 동작"을 미리 막는다
    }
});
```

`<form>`은 `submit` 이벤트가 발생하면 기본적으로 `action`에 지정된 주소로 페이지를 이동시키며 전송한다. 조건이 안 맞을 때(아이디/비밀번호 공백) 그 **기본 동작 자체를 막는 게** `evt.preventDefault()`다. "이벤트 발생은 막을 수 없지만, 이벤트가 원래 하려던 동작은 막을 수 있다"는 식으로 이해하니 다른 기본 동작(예: `<a>` 태그의 페이지 이동)에도 같은 방식으로 응용할 수 있겠다는 감이 왔다.

> ✅ **핵심**: `preventDefault()`는 이벤트 자체가 아니라 그 이벤트의 "기본 동작"(폼 전송, 링크 이동 등)을 막는다.

---

## 🛠️ 실전 적용 - 색상 순환, 다중 요소 이벤트 바인딩

**test1.html - 버튼 클릭마다 무지개색을 순환**

```js
var colors = ['red', 'orange', 'yellow', 'green', 'blue', 'navy', 'purple'];
var color_idx = 0;

function chagecolor(){                 // (원문 오타 그대로 - changeColor의 오타로 보임)
    var txt = document.getElementById('txt');
    txt.style.color = colors[color_idx % 7];
    color_idx++;
}
```

처음엔 `if (color_idx >= colors.length) color_idx = 0;`으로 인덱스를 리셋하는 방식을 주석처리하고, **나머지 연산자(`% 7`)** 로 바꾼 흔적이 남아있었다. `color_idx`가 계속 증가만 해도 `color_idx % 7`이 항상 0~6 사이 값을 만들어주기 때문에 리셋 조건문 자체가 필요 없어진다 - 배열을 순환시킬 때 꽤 자주 쓰는 패턴이라 기억해둘 만하다.

**test2.html - 여러 개의 같은 클래스 요소에 이벤트 바인딩**

```js
var item = document.getElementsByClassName('item');
for (var i = 0; i < item.length; i++){
    item[i].addEventListener('click', function(evt){
        var cur_cnt = this.querySelector('.cnt');
        cur_cnt.innerHTML = parseInt(cur_cnt.innerHTML) + 1;
    });
}
```

주석 처리된 코드를 보면 처음엔 `for (var btn of btns)`로 시도했다가 지금 방식으로 바꾼 흔적이 있다. 핵심은 **여러 개의 같은 클래스 요소를 순회하면서 각각에 이벤트를 등록**하되, 콜백 안에서는 `this`(=클릭된 그 `.item`)를 기준으로 `.cnt`를 찾아야 "클릭한 아이템의 카운터만" 정확히 올릴 수 있다는 점이다. 만약 `document.querySelector('.cnt')`처럼 전역으로 찾았다면 항상 첫 번째 아이템의 카운터만 올라갔을 것이다.

> ✅ **핵심**: 여러 요소에 이벤트를 반복 등록할 때는, 콜백 안에서 `document` 전체가 아니라 `this`(또는 `evt.currentTarget`) 기준으로 하위 요소를 찾아야 "클릭한 그 요소"만 정확히 처리된다.

---

## 🤔 감이 안 왔던 부분, 다시 짚어보기

- **왜 `this`가 항상 리스너 건 요소를 가리키지?** → `addEventListener`의 콜백 안에서 `this`는 "누가 이 함수를 호출했는가"에 의해 결정되는데, 브라우저가 이벤트를 발생시킬 때 **리스너가 걸린 요소를 `this`로 세팅해서 호출**해주기 때문이다. 단, 이 콜백을 화살표 함수로 썼다면 화살표 함수는 자신만의 `this`가 없으니 상위 스코프의 `this`(보통 `window`나 `undefined`)를 그대로 물려받아서 完全히 다른 값이 나온다. test2.html에서 굳이 `function(evt){}`로 쓴 이유가 이거였겠다 싶다.
- **live 컬렉션(HTMLCollection)이 실제로 문제 되는 상황은?** → 예를 들어 `getElementsByClassName`으로 가져온 컬렉션을 순회하면서 그 반복문 안에서 클래스를 떼거나 새 요소를 추가하면, 컬렉션 길이가 실시간으로 바뀌어서 **의도치 않게 일부 요소를 건너뛰거나 무한루프**에 빠질 수 있다고 한다. 다음엔 이 상황을 직접 재현해서 `querySelectorAll`(static)로 바꿨을 때 결과가 어떻게 달라지는지 비교해봐야겠다.
- **preventDefault()와 stopPropagation()은 다른 건가?** → `preventDefault()`는 "기본 동작 취소", `stopPropagation()`은 "이벤트 전파(버블링) 중단"으로 서로 다른 역할이다. 이번 실습엔 `stopPropagation`이 안 나왔는데, this vs target을 이해한 김에 다음 실습에서 버블링을 의도적으로 막아보는 것도 해봐야겠다.

---

## 💡 정리하면

이번 장은 "DOM을 찾고(선택자) → 바꾸고(속성 제어) → 새로 만들고(createElement) → 반응하게 만든다(이벤트)"는 하나의 흐름이었다. 특히 이벤트 파트에서 `this`/`evt.target`/버블링/`removeEventListener`처럼 **눈에 보이는 결과는 비슷한데 내부 동작이 미묘하게 다른 개념들**이 몰려 있어서, 이번 정리에서 그 차이를 확실히 짚어두는 게 나중에 디버깅할 때 큰 도움이 될 것 같다.

**오늘의 핵심 6가지**

- 🔎 선택자 이름이 복수형(Elements)이면 컬렉션, 단수형이면 단일 요소를 반환한다
- 🎨 사용자 정의 속성이나 안 먹는 속성은 `setAttribute`/`getAttribute`로 다루는 게 안전하다
- 🏗️ 요소 생성은 "만들기 → 채우기 → appendChild로 붙이기" 3단계다
- 🖱️ 인라인 이벤트보다 `addEventListener`가 관심사 분리와 다중 등록 면에서 유리하다
- 🎯 `this`는 리스너가 걸린 요소, `evt.target`은 실제 이벤트 발생 요소 - 버블링 때문에 다를 수 있다
- 🔌 `removeEventListener`는 등록 때와 같은 함수 참조가 있어야 하므로, 나중에 지울 이벤트는 이름 붙인 함수로 등록해야 한다

**다음에 확인해볼 것**

1. HTMLCollection(live) 순회 중 요소를 추가/삭제했을 때 실제로 어떤 문제가 생기는지 직접 재현해보기
2. addEventListener 콜백을 화살표 함수로 바꿔서 `this`가 어떻게 달라지는지 test2.html로 비교해보기
3. `stopPropagation()`으로 버블링을 의도적으로 막아보고 `this`/`target` 동작이 어떻게 바뀌는지 확인하기
