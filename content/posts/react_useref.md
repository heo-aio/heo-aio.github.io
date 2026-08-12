+++
title = "[React] useRef - state와 뭐가 다른가, DOM에 직접 접근하기"
date = 2026-08-12
draft = false
tags = ["React", "useRef", "hooks", "클로저"]
categories = ["dev"]
math = false
+++

State를 배우고 나서 바로 이어지는 실습이 `useRef`였는데, 처음 코드만 봐서는 "그냥 state랑 비슷한 거 아닌가?"였다. `const refVal = useRef(0)`도 값을 담고, `refVal.current`도 바뀌길래 헷갈렸는데, 두 컴포넌트를 나란히 클릭해보고 나서야 결정적인 차이가 눈에 보였다.

**목차**
1. [useRef vs useState - 리렌더링을 일으키는가](#-useref-vs-usestate---리렌더링을-일으키는가)
2. [useRef의 진짜 쓸모 - DOM에 직접 접근하기](#-useref의-진짜-쓸모---dom에-직접-접근하기)
3. [setTimeout 안에서 state는 옛날 값을, ref는 최신 값을 본다](#-settimeout-안에서-state는-옛날-값을-ref는-최신-값을-본다)
4. [감이 안 왔던 부분, 다시 짚어보기](#-감이-안-왔던-부분-다시-짚어보기)
5. [정리하면](#-정리하면)

---

## 🔍 useRef vs useState - 리렌더링을 일으키는가

```jsx
import {useRef, useState} from "react";

export default function RefVar(){
    const[val, setVal] = useState(0)
    const refVal = useRef(0);

    const updateState = ()=>{ setVal(val + 1); }
    const updateRef = ()=>{ refVal.current += 1; }

    return(
        <div>
            <button onClick={updateState}>State Count : {val}</button>
            <button onClick={updateRef}>Ref Count : {refVal.current}</button>
        </div>
    )
}
```

두 버튼을 각각 여러 번 눌러보면 차이가 바로 보인다. `State Count` 버튼은 누를 때마다 숫자가 화면에서 바로 올라간다. 그런데 `Ref Count` 버튼은 아무리 눌러도 화면엔 계속 0으로 보인다 (콘솔로 `refVal.current`를 찍어보면 실제로는 값이 올라가고 있다). 2편에서 정리했던 "화면을 다시 그리게 만드는 건 값이 바뀜이 아니라 setter 호출"이라는 규칙이 여기서도 그대로 적용됐다 - `useRef`는 값을 담는 상자(`{current: 값}`)를 하나 만들어줄 뿐, **그 값을 바꾼다고 리렌더링을 일으키지는 않는다.**

![useState는 setter 호출 시 리렌더링을 일으켜 화면이 즉시 갱신되지만, useRef는 값이 바뀌어도 리렌더링을 일으키지 않아 화면이 갱신되지 않는 차이](/images/react/ref_vs_state_rerender.svg)

그럼 "화면에 안 보이는 값을 왜 굳이 관리하나?" 싶었는데, 답은 오히려 그 반대였다. **"리렌더링과 상관없이 값을 계속 유지하고 싶을 때"** 쓰는 게 `useRef`다. 예를 들어 렌더링과 무관하게 누적해야 하는 카운터, 이전 렌더링 때의 값을 기억해두는 용도, 혹은 타이머 ID처럼 "화면에 표시할 필요는 없지만 다음 렌더링에서도 잃어버리면 안 되는 값"을 다룰 때 유용하다는 걸 알게 됐다.

> ✅ **핵심**: `useState`는 값이 바뀌면 리렌더링이 일어나 화면이 즉시 갱신된다. `useRef`는 값이 바뀌어도 리렌더링을 일으키지 않는다 - "화면 갱신과 무관하게 값을 기억"하고 싶을 때 쓴다.

---

## 🎯 useRef의 진짜 쓸모 - DOM에 직접 접근하기

```jsx
import {useRef, useState} from "react";

function GetElem(){
    const [text, setText] = useState();
    let inputRef = useRef(null);

    const chgText = (e) => { setText(e.target.value); }

    const inputFocus = () => {
        console.log([inputRef.current]);
        inputRef.current.focus();
    }

    return(
        <div>
            <h3>입력값 : {text}</h3>
            <input type={"text"} value={text}
                   onChange={(e)=>{chgText(e)}} ref={inputRef}/>
            <button onClick={inputFocus}>TAB</button>
        </div>
    )
}
```

`useRef`의 더 흔한 용도가 이거였다. `ref={inputRef}`처럼 JSX 태그에 `ref` 속성으로 연결해두면, `inputRef.current`에 **그 태그의 실제 DOM 요소**가 담긴다. `inputRef.current.focus()`를 호출하면 진짜 브라우저 DOM API인 `.focus()`가 그대로 실행되면서 입력창에 커서가 이동한다.

이게 왜 필요한가 싶었는데, 생각해보니 순수 JS였다면 `document.getElementById('inputStr')`로 DOM을 직접 가져왔을 부분이다. React는 "화면을 state로 관리하고, 직접 DOM을 건드리지 않는다"는 게 기본 철학이라 `document.querySelector` 같은 걸 함부로 쓰지 않는데, focus 이동이나 스크롤 위치, 특정 요소의 크기 측정처럼 **state로는 표현이 안 되고 DOM 자체를 만져야 하는 경우**에 한해 `useRef`가 "탈출구" 역할을 해준다는 걸 이해했다.

> ✅ **핵심**: JSX 태그에 `ref={변수}`를 연결하면 `변수.current`에 실제 DOM 요소가 담긴다. `focus()`, 스크롤 제어처럼 state만으로는 안 되는 DOM 직접 제어가 필요할 때 쓴다.

---

## ⏱️ setTimeout 안에서 state는 옛날 값을, ref는 최신 값을 본다

```jsx
import {useRef, useState} from "react";

export default function Comp(){
    const [count, setCount] = useState(0)
    let refVal = useRef(0);

    const updateCount = () => {
        setCount(count + 1);
        refVal.current += 1;
    }

    const alertCount = () => {
        setTimeout(()=>{
            console.log(`3초 동안 ${count}번 클릭 !`);        // 처음 값 그대로
            console.log(`3초 동안 ${refVal.current}번 클릭 !`); // 최신 값
        }, 3000);
    }
    // ...
}
```

이 코드가 이번 실습에서 제일 뇌를 자극한 부분이었다. "Show alert me" 버튼을 누른 뒤 3초 안에 "Click Me"를 여러 번 더 누르면, 3초 뒤 콘솔에 찍히는 두 줄이 서로 다른 숫자를 보여준다. `count`는 버튼을 누른 **그 순간의 값**에서 멈춰 있고, `refVal.current`는 3초 사이에 계속 늘어난 **최신 값**을 그대로 보여준다.

![setTimeout 콜백 실행 시점에, state는 클로저에 캡처된 클릭 당시의 값을 그대로 보여주지만 ref는 실행되는 순간의 최신값을 참조하는 차이](/images/react/closure_state_vs_ref.svg)

이게 왜 이렇게 되는지가 처음엔 이해가 안 됐는데, 저번에 실행 컨텍스트/클로저를 정리했던 게 여기서 그대로 답이 됐다. `setTimeout`에 넘긴 콜백 함수는 **클로저**라서, `alertCount`가 호출된 그 순간의 `count` 값을 그대로 "기억"해서 3초 뒤에도 그 값을 쓴다 - `count`는 그 함수가 호출될 때 스냅샷처럼 고정된 지역 변수인 것이다. 반면 `refVal`은 값 자체가 아니라 `{current: ...}`라는 **객체(참조)**를 클로저가 기억하는 것이라서, 3초 뒤 콜백이 실행되는 시점에 `refVal.current`를 다시 읽으면 그 사이 바뀐 최신 값이 그대로 나온다.

즉 `count`는 "값을 복사해서 기억"하는 셈이고, `refVal`은 "상자를 기억해서, 실행되는 순간 상자 안을 다시 들여다보는" 셈이다. `state`가 왜 "그 렌더링 시점의 스냅샷"이라고 불리는지, `ref`가 왜 "항상 최신값을 참조"한다고 하는지 이 실습으로 확실히 체감했다.

> ✅ **핵심**: state는 클로저에 그 순간의 값이 그대로 캡처되어(스냅샷), 나중에 실행돼도 옛날 값을 본다. ref는 객체(`{current}`)를 클로저가 기억하기 때문에, 나중에 실행되는 시점에 `.current`를 다시 읽어 최신 값을 본다.

---

## 🤔 감이 안 왔던 부분, 다시 짚어보기

- **`let refVal = useRef(0)`와 `const refVal = useRef(0)`, 왜 실습 코드에 둘 다 나오나?** → `RefVar.jsx`는 `const`, `Comp.jsx`는 `let`을 썼는데 동작에 차이가 없어서 헷갈렸다. 생각해보면 `refVal` 자체(그 객체를 가리키는 변수)는 재할당하지 않고 `refVal.current`만 바꾸니까 사실 `const`가 더 정확한 표현일 것 같다. 다음엔 강의에서 `let`을 쓴 이유가 있는지 확인해봐야겠다.
- **state의 "스냅샷" 개념이 다른 곳(예: 이벤트 핸들러)에도 똑같이 적용되나?** → 이번엔 `setTimeout`으로만 확인했는데, `addEventListener`나 다른 비동기 콜백(Promise, API 응답 등) 안에서도 같은 원리로 "그 시점의 state"가 캡처될 것 같다. 다음에 API 호출 실습이 나오면 이 부분을 직접 재현해서 확인해봐야겠다.
- **ref를 남용하면 안 되는 이유가 정확히 뭘까?** → "state로 표현 안 되는 경우의 탈출구"라고 이해했는데, 그럼 반대로 "웬만하면 state를 써야 하는 이유"가 궁금해졌다. 짐작으로는 ref로 값을 바꾸면 화면이 안 바뀌니까, 그 값에 의존하는 UI가 있으면 버그처럼 보일 수 있어서인 것 같다. 다음 실습에서 실제로 ref 값을 화면에 표시하려다 실패하는 경우를 만들어서 확인해봐야겠다.

---

## 💡 정리하면

`useRef`를 배우고 나니 State가 "화면에 보여줄 값"이라면, Ref는 "화면과 상관없이 계속 붙잡고 있어야 하는 값"이라는 역할 구분이 명확해졌다. 그리고 setTimeout 클로저 예제 덕분에, state가 왜 "그 순간의 스냅샷"으로 동작하는지 실행 컨텍스트/클로저 개념과 완전히 연결해서 이해할 수 있었다.

**오늘의 핵심 4가지**

- 🔍 `useState`는 값이 바뀌면 리렌더링을 일으키지만, `useRef`는 값이 바뀌어도 리렌더링을 일으키지 않는다
- 🎯 `ref={변수}`로 DOM 요소에 직접 접근할 수 있다 - focus, 스크롤처럼 state로 표현 안 되는 DOM 제어에 사용
- ⏱️ state는 클로저에 그 순간의 값이 스냅샷처럼 캡처되고, ref는 객체(`.current`)를 참조하기 때문에 실행 시점의 최신값을 본다
- 🧩 "화면에 보여줄 값 = state", "화면과 무관하게 유지할 값 = ref"로 역할을 구분해서 골라 쓴다