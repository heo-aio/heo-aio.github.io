+++
title = "[JavaScript] DOM과 이벤트 2 - 캡처링/버블링, 이벤트 위임, setInterval/setTimeout"
date = 2026-08-06T18:00:00+09:00
draft = false
tags = ["JavaScript", "DOM", "이벤트", "이벤트위임", "타이머", "웹개발"]
categories = ["dev"]
math = false
+++

DOM과 이벤트 2부는 지난번(1부)에서 짚었던 `this`/`evt.target` 이야기가 왜 그렇게 동작하는지의 **원리**를 다루는 느낌이었다. 이벤트가 캡처링-타겟-버블링 3단계로 전파된다는 걸 알고 나니, 지난번에 이해가 잘 안 갔던 이벤트 위임(delegation) 패턴도 자연스럽게 설명이 됐다. 여기에 `setInterval`/`setTimeout` 타이머까지 더해져서, 실습(실습1~7)들을 보면 결국 이 개념들의 조합이라는 게 보였다.

**목차**
1. [이벤트 전파 3단계 - 캡처링, 타겟, 버블링](#-이벤트-전파-3단계---캡처링-타겟-버블링)
2. [stopPropagation - 버블링 끊기](#-stoppropagation---버블링-끊기)
3. [이벤트 위임(delegation) - 부모 하나로 자식 전체 처리하기](#-이벤트-위임delegation---부모-하나로-자식-전체-처리하기)
4. [setInterval / setTimeout - 반복과 지연 실행](#-setinterval--settimeout---반복과-지연-실행)
5. [스크롤 제어 - scrollIntoView, scrollTo](#-스크롤-제어---scrollintoview-scrollto)
6. [classList로 상태 표현하기](#-classlist로-상태-표현하기)
7. [실전 적용 모음 - 출석부, 로그인 활성화, 카레이싱](#-실전-적용-모음---출석부-로그인-활성화-카레이싱)
8. [감이 안 왔던 부분, 다시 짚어보기](#-감이-안-왔던-부분-다시-짚어보기)
9. [정리하면](#-정리하면)

---

## 🌊 이벤트 전파 3단계 - 캡처링, 타겟, 버블링

```js
var p1 = document.getElementById('parent_1');
var p2 = document.getElementById('parent_2');
var p3 = document.getElementById('parent_3');
var btn = document.getElementById('btn');

// 캡처링 확인 (3번째 인자에 true를 줘야 캡처링 단계에서 잡힌다)
p1.addEventListener('click', target_name, true);
p2.addEventListener('click', target_name, true);
p3.addEventListener('click', target_name, true);

// 버블링 (3번째 인자 생략/false가 기본값)
btn.addEventListener('click', target_name);
p1.addEventListener('click', target_name);
p2.addEventListener('click', target_name);
p3.addEventListener('click', target_name);

function target_name(evt){
    console.log(this.getAttribute('id') + ' click');
    console.log('이벤트 단계 : ', evt.eventPhase);   // 1=캡처링, 2=타겟, 3=버블링
    console.log('현재 지점 : ', evt.currentTarget);   // 리스너가 걸린 요소 (this와 동일)
    console.log('타겟 : ', evt.target);               // 실제 클릭된 요소
}
```

`button#btn`을 클릭하면 콘솔에 로그가 여러 번 찍히는 걸 보고 처음엔 "왜 한 번 클릭했는데 여러 번 찍히지?" 싶었다. HTML 주석에 적혀 있던 3단계 설명을 실습 코드와 맞춰보니 이해가 됐다.

![이벤트 전파 3단계 - 캡처링(내려감) → 타겟(실행) → 버블링(올라감)](/images/js_dom/event_phases.svg)

- **캡처링**: `window`에서 시작해서 클릭 지점까지 트리를 타고 **내려오는** 단계 (`parent_1 → parent_2 → parent_3`)
- **타겟**: 실제 클릭된 지점(`btn`)에서 실행되는 단계
- **버블링**: 클릭 지점에서 다시 `window`까지 **올라가는** 단계 (`parent_3 → parent_2 → parent_1`)

`addEventListener`의 3번째 인자를 `true`로 주면 캡처링 단계에서, 생략하거나 `false`를 주면(기본값) 버블링 단계에서 콜백이 실행된다는 것도 이번에 명확해졌다. `evt.currentTarget`은 `this`와 같은 값(리스너가 걸린 요소)이고, `evt.target`은 실제 클릭이 일어난 요소라는 걸 1부 내용과 이어서 확인할 수 있었다.

> ✅ **핵심**: 이벤트는 캡처링(내려감) → 타겟 → 버블링(올라감) 순서로 전파된다. `addEventListener`의 3번째 인자로 캡처링 단계에서 잡을지 버블링 단계에서 잡을지 정할 수 있다.

---

## ✋ stopPropagation - 버블링 끊기

```js
function target_name(evt){
    console.log(this.getAttribute('id') + ' click');
    evt.stopPropagation();  // 여기서 전파를 끊는다
}
```

`10_stopPropagation.html`은 `index.html`과 거의 같은 코드에 `evt.stopPropagation()` 한 줄만 추가되어 있었다. 이 한 줄이 있으면 `btn`을 클릭했을 때 `btn`에서만 로그가 찍히고, `parent_3`, `parent_2`, `parent_1`까지 이어지던 버블링이 그 자리에서 멈춘다. **"이벤트 발생 자체를 막는 게 아니라, 그 이벤트가 부모로 전파되는 것만 막는다"**는 점에서 1부에서 다룬 `preventDefault()`(기본 동작을 막음)와는 역할이 다르다는 걸 나란히 비교하니 확실히 구분됐다.

> ✅ **핵심**: `stopPropagation()`은 이벤트가 상위(또는 하위)로 전파되는 것을 막는다. "기본 동작을 막는" `preventDefault()`와는 다른 역할이다.

---

## 🎯 이벤트 위임(delegation) - 부모 하나로 자식 전체 처리하기

```js
// 실습7.html - section 안의 색상 div 4개, section에만 이벤트를 건다
section.addEventListener('mouseover', function(evt){
    if (evt.target.className !== ''){
        class_name = evt.target.className;
    }
    document.querySelector('.selected').innerHTML = class_name;
    document.querySelector('.car1').style.backgroundColor = class_name;
});
```

```js
// 실습5.html - 아이콘 3개, 각각이 아니라 반복문으로 한 번에 등록 + this로 분기
var btns = document.getElementsByTagName('i');
for (var btn of btns){
    btn.addEventListener('click', toggle);
}
function toggle(e){
    for (var item of btns){ item.classList.remove('on'); }
    this.classList.add('on');  // 클릭된 그 아이콘에만 클래스 추가
}
```

이벤트 위임이 왜 필요한지, 버블링을 배우고 나서야 제대로 이해가 됐다. **자식 요소 각각에 이벤트를 거는 대신, 공통 부모에 이벤트를 하나만 걸고, 버블링으로 올라온 이벤트에서 `evt.target`으로 "실제 눌린 게 누구인지" 구분**하는 방식이다.

![이벤트 위임 - 자식마다 등록하지 않고 부모 하나에 걸어 evt.target으로 구분](/images/js_dom/event_delegation.svg)

실습5의 주석 처리된 코드를 보면 원래는 `box1`, `box2`, `box3` 각각에 클릭 이벤트를 따로 걸고, 클릭될 때마다 나머지 두 개의 `on` 클래스를 일일이 지우는 방식으로 짜여 있었다. 지금 방식(`for...of` + `toggle` 함수 하나)으로 바꾸면서 **"아이콘이 100개여도 코드가 안 늘어난다"**는 장점이 확 와닿았다. 다만 이건 반복문으로 각 요소에 리스너를 개별 등록한 것이라 완전한 "위임"은 아니고, 실습7의 `section.addEventListener`처럼 **부모 하나에만 걸고 자식엔 아예 이벤트를 걸지 않는 것**이 진짜 이벤트 위임에 가깝다. 두 방식의 공통점은 결국 "반복 등록을 줄이고 `evt.target`/`this`로 대상을 구분한다"는 것이었다.

> ✅ **핵심**: 이벤트 위임은 버블링을 역이용해서 부모에 이벤트를 한 번만 걸고, `evt.target`으로 실제 대상을 구분하는 패턴이다. 요소가 많거나 동적으로 추가될 때 특히 유리하다.

---

## ⏱️ setInterval / setTimeout - 반복과 지연 실행

```js
// time.html
var time;
function start(elem){
    elem.disabled = true;
    time = setInterval(function(){       // 1초마다 반복 실행
        i++;
        h1.innerHTML = i;
    }, 1000);
}
function stop(){
    clearInterval(time);   // 반복을 멈추려면 저장해둔 id가 필요하다
}

function setTime(){
    setTimeout(function(){                // 3초 뒤에 딱 1번 실행
        alert('3초가 경과되었습니다.');
    }, 3000);
}
```

![setInterval(반복) vs setTimeout(1회 지연) 타임라인 비교](/images/js_dom/timer_functions.svg)

`setInterval`은 지정한 주기로 **계속** 실행되고, `setTimeout`은 지정한 시간이 지난 뒤 **딱 한 번만** 실행된다. 여기서 놓치기 쉬운 포인트는 **`setInterval`을 멈추려면 시작할 때 반환값을 변수(`time`)에 저장해둬야 한다**는 것이다. 저장 안 해두면 `clearInterval`에 넘길 id 자체가 없어서 영원히 멈출 수 없다.

`test.html`의 다운로드 진행바 예제가 두 함수의 조합을 잘 보여준다:

```js
function count(){
    time = setInterval(function(){ cnt--; span.innerHTML = cnt; }, 1000); // 카운트다운

    setTimeout(function(){       // 5초 뒤에 카운트다운을 멈추고 진행바 시작
        clearInterval(time);
        download();
    }, 5000);
}
```

카운트다운은 `setInterval`로 계속 돌리고, 정확히 5초 뒤(`setTimeout`) 그 반복을 끊은 뒤(`clearInterval`) 다음 동작(`download()`)으로 넘어가는 구조다. **"반복해야 하면 setInterval, 한 번만 늦게 실행하면 setTimeout, 반복을 끊고 싶으면 반드시 id를 clearInterval에 넘긴다"**는 3줄 규칙으로 정리하니 헷갈리지 않았다.

> ✅ **핵심**: `setInterval`은 반복, `setTimeout`은 1회 지연 실행. 둘 다 반환값(id)을 변수에 저장해둬야 `clearInterval`/`clearTimeout`으로 멈출 수 있다.

---

## 🧭 스크롤 제어 - scrollIntoView, scrollTo

```js
// 특정 요소가 화면에 보이도록 부드럽게 스크롤
document.querySelector('.third').scrollIntoView({ behavior: 'smooth' });

// 페이지 맨 위로 부드럽게 스크롤
scrollTo({ top: 0, left: 0, behavior: 'smooth' });
```

`scrollIntoView`는 "이 요소가 보이도록 스크롤해줘", `scrollTo`는 "이 좌표로 스크롤해줘"라는 차이로 구분하면 헷갈리지 않는다. `behavior: 'smooth'` 하나로 순간이동 대신 부드러운 스크롤 애니메이션이 공짜로 생긴다는 것도 실습에서 확인한 부분 - CSS 애니메이션을 따로 안 짜도 되는 게 편리했다.

> ✅ **핵심**: `scrollIntoView()`는 요소 기준, `scrollTo()`는 좌표 기준 스크롤이다. `behavior: 'smooth'` 옵션으로 부드러운 이동을 쉽게 넣을 수 있다.

---

## 🏷️ classList로 상태 표현하기

```js
loginButton.classList.add('activatedColor');
loginButton.classList.remove('deactivatedColor');
```

`element.className = '...'`처럼 문자열 전체를 덮어쓰는 방식과 달리, `classList.add`/`remove`는 **다른 클래스는 건드리지 않고 원하는 클래스만** 붙이거나 뗄 수 있다. 실습4(로그인 버튼 활성화)와 실습5(아이콘 선택 표시) 둘 다 결국 "조건에 따라 클래스를 붙였다 뗐다 하는" 같은 패턴이었다 - 상태를 색깔이나 스타일로 표현할 때 `classList`가 `style.backgroundColor`를 직접 바꾸는 것보다 훨씬 다루기 편하다는 걸 느꼈다.

> ✅ **핵심**: 상태에 따른 스타일 변경은 `style` 직접 수정보다 `classList.add`/`remove`로 클래스를 토글하는 방식이 더 안전하고 관리하기 쉽다.

---

## 🛠️ 실전 적용 모음 - 출석부, 로그인 활성화, 카레이싱

**실습2, 실습3 - 폼 제출과 목록 누적**

```js
form.addEventListener('submit', function(evt){
    evt.preventDefault();               // 새로고침 막기
    if (input.value == '') { alert('입력 내용을 넣어주세요.'); }
    else { answer.innerHTML = input.value; input.value = ''; }
});
```

출석부(실습3)는 클릭할 때마다 `createElement('p')`로 새 태그를 만들어 `appendChild`로 계속 쌓는 방식이었다. 1부에서 다룬 "만들기 → 채우기 → 붙이기" 패턴이 여기서도 그대로 반복되는 걸 확인했다.

**실습4 - 입력값에 따라 버튼 실시간 활성화**

```js
id.addEventListener('keyup', activateBtn);
pw.addEventListener('keyup', activateBtn);

function activateBtn(){
    if (id.value !== '' && pw.value !== ''){
        loginButton.classList.add('activatedColor');
        loginButton.classList.remove('deactivatedColor');
    } else {
        loginButton.classList.add('deactivatedColor');
        loginButton.classList.remove('activatedColor');
    }
}
```

`keyup`(키를 뗄 때마다)마다 두 입력창의 값을 체크해서 버튼 상태를 갱신하는 구조. **"입력 이벤트가 일어날 때마다 조건을 다시 검사한다"**는 흐름이 실시간 유효성 검사의 기본형이라는 걸 느꼈다.

**실습7 - 카레이싱 게임 (setInterval + Math.random)**

```js
var race = setInterval(function(){
    x1 += parseInt(Math.random() * 50 + 1);
    x2 += parseInt(Math.random() * 50 + 1);
    car1.style.marginLeft = x1 + 'px';
    car2.style.marginLeft = x2 + 'px';

    if (x1 >= target_line || x2 >= target_line){
        clearInterval(race);
        alert(x1 > x2 ? 'car1 승리' : 'car2 승리');
        reset();
    }
}, 100);
```

`setInterval`로 100ms마다 위치를 랜덤하게 전진시키다가, 둘 중 하나가 도착선(`target_line`)을 넘으면 `clearInterval`로 멈추고 승자를 알리는 구조다. 타이머 + 랜덤 + 조건부 종료가 합쳐지니 간단한 애니메이션 게임이 되는 게 재밌었다. `Math.random() * 50 + 1`이 왜 1~50 사이 값을 만드는지도 - `Math.random()`이 0~0.99...를 반환하니 `*50`하면 0~49.xx, `+1`하면 1~50.xx 범위가 된다는 걸 다시 짚었다.

> ✅ **핵심**: 실전 예제들은 결국 "이벤트로 트리거 → DOM 조작(생성/클래스/스타일) → 필요하면 타이머로 반복"이라는 조합의 반복이다.

---

## 🤔 감이 안 왔던 부분, 다시 짚어보기

- **캡처링은 실무에서 언제 쓰나?** → 대부분의 이벤트 처리는 버블링(기본값)만으로 충분하다고 한다. 캡처링을 굳이 쓰는 경우는 "자식이 이벤트를 가로채기(stopPropagation) 전에 부모가 먼저 손을 써야 할 때"처럼 특수한 상황이라는데, 이번 실습만으로는 감이 완전히 오진 않아서 다음에 캡처링이 실제로 필요한 예제를 따로 찾아봐야겠다.
- **이벤트 위임과 "그냥 반복문으로 각각 등록"의 차이가 뭔가?** → 실습5처럼 `for...of`로 각 요소에 개별 등록하는 것도 코드는 짧아지지만, 이건 여전히 **요소 개수만큼 리스너가 걸려 있는 상태**다. 반면 실습7처럼 부모 하나에만 거는 진짜 위임은 **리스너 개수 자체가 1개**라서, 나중에 자식이 동적으로 추가돼도 별도 등록 없이 자동으로 처리된다는 차이가 있다. 이 둘을 같은 걸로 착각하기 쉬운데, 명확히 구분해서 기억해야겠다.
- **`clearInterval`을 안 부르면 정말 안 멈추나?** → 실습(`time.html`)에서 `stop()`을 안 누르면 카운터가 계속 올라간다는 걸 직접 확인했다. 페이지를 벗어나기 전까지 리소스를 계속 쓴다는 뜻이라, 컴포넌트/페이지가 사라질 때 타이머를 정리하는 습관(예: `clearInterval`)이 왜 중요한지 체감이 됐다. 다음엔 리액트 등 프레임워크에서 `useEffect` cleanup으로 타이머를 정리하는 패턴과 연결해서 봐야겠다.

---

## 💡 정리하면

DOM과 이벤트 2부는 1부에서 "왜 그렇게 동작하는지 몰랐던" `this`/`evt.target`의 원리(버블링)를 채워주는 내용이었다. 여기에 타이머(`setInterval`/`setTimeout`)가 더해지면서, 실습들이 전부 **"이벤트로 시작해서 → DOM을 조작하고 → 필요하면 시간을 두고 반복하거나 멈추는"** 하나의 패턴으로 수렴하는 걸 느꼈다.

**오늘의 핵심 6가지**

- 🌊 이벤트는 캡처링(내려감) → 타겟 → 버블링(올라감) 3단계로 전파된다
- ✋ `stopPropagation()`은 전파를 막고, `preventDefault()`는 기본 동작을 막는다 - 역할이 다르다
- 🎯 이벤트 위임은 버블링을 이용해 부모 하나에만 리스너를 걸고 `evt.target`으로 대상을 구분하는 패턴이다
- ⏱️ `setInterval`은 반복, `setTimeout`은 1회 지연 - 둘 다 id를 저장해둬야 멈출 수 있다
- 🧭 `scrollIntoView`(요소 기준) / `scrollTo`(좌표 기준) + `behavior: 'smooth'`로 부드러운 스크롤을 쉽게 구현
- 🏷️ 상태에 따른 스타일 변경은 `classList.add`/`remove`가 `style` 직접 수정보다 안전하다

**다음에 확인해볼 것**

1. 캡처링 단계가 실제로 유용한 상황(자식의 stopPropagation을 부모가 먼저 처리해야 하는 경우)을 예제로 만들어보기
2. "반복문으로 개별 등록"과 "부모 하나에 진짜 위임"의 성능 차이를 요소 개수를 늘려가며 체감해보기
3. `useEffect` cleanup 함수와 `clearInterval`/`clearTimeout`을 연결해서, 프레임워크에서는 타이머 정리를 어떻게 하는지 찾아보기
