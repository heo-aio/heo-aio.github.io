+++
title = "[Redux] action, payload 개념 정리 - 실습 카운터 코드 리뷰까지"
date = 2026-08-25
draft = false
tags = ["Redux", "Redux Toolkit", "React", "상태관리"]
categories = ["dev"]
math = false
+++

useEffect 정리할 때는 그래도 "아 이런 느낌이구나" 하고 감이 왔는데, Redux는 확실히 결이 다르다. 지금까지 배운 건 컴포넌트 안에서 상태를 어떻게 다루느냐였다면, Redux는 컴포넌트 밖에 상태를 통째로 떼어놓고 관리하는 방식이라 처음 봤을 때 "이걸 왜 이렇게까지 복잡하게 하지?" 싶었다. 강사님이랑 카운터 예제로 실습을 하긴 했는데, 코드는 따라 쳤어도 dispatch랑 action, 특히 payload가 정확히 뭘 하는 건지는 설명을 들어도 계속 헷갈려서 오늘 배운 내용을 실습 코드 기준으로 다시 정리해본다.

**목차**
1. [Redux가 낯설게 느껴진 이유](#-redux가-낯설게-느껴진-이유)
2. [action과 payload 제대로 이해하기](#-action과-payload-제대로-이해하기)
3. [실습 코드 리뷰 - 잘한 점과 아쉬운 점](#-실습-코드-리뷰---잘한-점과-아쉬운-점)
4. [개선한 버전으로 다시 짜보기](#-개선한-버전으로-다시-짜보기)
5. [감이 안 왔던 부분, 다시 짚어보기](#-감이-안-왔던-부분-다시-짚어보기)
6. [정리하면](#-정리하면)

---

## 🌀 Redux가 낯설게 느껴진 이유

지금까지는 상태를 바꿀 때 `setState(newValue)`처럼 직접 값을 넣어주는 방식에 익숙했다. 근데 Redux는 그렇게 안 하고,

1. "이런 일이 일어났어요"라는 **action**을 던지고 (dispatch)
2. 그 action을 받은 **reducer**가 새로운 state를 계산해서
3. **store**라는 한 곳에 저장된 상태를 통째로 교체한다

라는 절차를 거친다. 값을 직접 바꾸는 게 아니라 "무슨 일이 있었는지"를 알리고, 그걸 받아서 처리하는 함수가 따로 있는 구조다 보니 처음엔 왜 이렇게 한 단계를 더 거치는지 이해가 안 갔다. 근데 실습 코드를 다시 읽으면서, 이게 "상태가 왜 이렇게 바뀌었는지 추적 가능하게 만들기 위한 규칙"이라는 걸 알고 나니 조금 납득이 됐다.

> ✅ **핵심**: Redux는 상태를 직접 바꾸지 않고, "무슨 일이 일어났는지"(action)를 먼저 알린 뒤 reducer가 새 상태를 계산해서 store를 교체하는 방식으로 동작한다.

---

## 📦 action과 payload 제대로 이해하기

오늘 제일 헷갈렸던 게 이 부분이다. action은 그냥 아래처럼 생긴 **평범한 객체**다.

```js
{ type: 'counter/increment' }
```

`type`은 필수고, "무슨 종류의 일이 일어났는지"를 나타내는 문자열이다. 실습에서는 `counter/increment`, `counter/decrement`처럼 `슬라이스이름/리듀서함수이름` 형태로 자동 생성됐는데, 이게 `createSlice`의 `name`(`counter`)이랑 `reducers` 안의 함수 이름(`increment`)이 합쳐진 결과였다.

문제는 **"값을 같이 넘기고 싶을 때"**다. `+1`, `-1`은 정해진 값이라 상관없었는데, "5만큼 더하기" 같은 걸 하려면 action한테 숫자 5를 실어 보내야 한다. 이때 쓰는 게 `payload`다.

```js
{ type: 'counter/addByAmount', payload: 5 }
```

즉 `payload`는 **action이 들고 다니는 짐(데이터)**이다. type이 "어떤 사건이 일어났는지"를 설명한다면, payload는 "그 사건에 딸려오는 구체적인 값"인 셈이다. Redux 공식 컨벤션(Flux Standard Action)에서도 액션은 보통 `{ type, payload }` 형태를 기본으로 삼는다.

Redux Toolkit에서는 `dispatch(addByAmount(5))`처럼 action creator 함수를 호출하면, 내부적으로 `{ type: 'counter/addByAmount', payload: 5 }`를 자동으로 만들어준다. 그리고 reducer 쪽에서는 `action.payload`로 그 값을 꺼내 쓴다.

```js
addByAmount: (state, action) => {
  state.value += action.payload; // 여기서 payload가 5
}
```

> ✅ **핵심**: `type`은 "무슨 일이 일어났는지", `payload`는 "그 일에 딸려온 데이터"다. 값을 넘기지 않는 액션(`increment` 등)은 payload가 없어도 되지만, 값이 필요한 순간부터는 payload가 필수가 된다.

글로만 보면 계속 헷갈려서, `dispatch(addByAmount(5))`를 눌렀을 때 payload가 실제로 어디를 거쳐서 화면까지 도달하는지 직접 그려봤다.

![컴포넌트에서 dispatch(addByAmount(5))를 호출하면 type과 payload를 가진 action 객체가 만들어지고, reducer가 action.payload를 읽어 state.value에 더한 뒤 store에 저장하고, useSelector로 구독한 컴포넌트가 리렌더링되는 과정](/images/react/redux_action_payload_flow.svg)

action 객체 안에 `type`이랑 `payload`가 나란히 들어가 있는 걸 보니, "payload가 reducer한테 넘겨줄 짐"이라는 게 그림으로는 확실히 이해가 됐다. reducer 박스에서 `action.payload(5)`를 꺼내 쓰는 부분이 지난번 다이어그램에서 뭉뚱그려 놨던 "새 state 계산" 단계의 실제 내용이었던 거다.

---

## 🔍 실습 코드 리뷰 - 잘한 점과 아쉬운 점

강사님이랑 같이 친 실습 코드를 다시 천천히 읽어봤다. 구조 자체(slice → store → Provider → 컴포넌트)는 지난번에 정리한 흐름대로 잘 맞아떨어졌는데, 다시 보니 아쉬운 지점이 몇 개 보였다.

**잘한 점**
- `createSlice`로 reducer랑 action을 한 번에 관리한 것 (state, reducers를 따로 안 만들어도 됨)
- `Provider`로 `layout.jsx`(최상위)를 감싸서 앱 전체에 store를 공급한 것
- `useSelector`로 필요한 값(`state.counter.value`)만 정확히 구독한 것

**아쉬운 점 1 - action creator를 안 쓰고 문자열을 직접 씀**

`page.jsx`를 보면 이렇게 되어 있다.

```js
const upHit = function(){
    store.dispatch({type:'counter/increment'});
};
```

`createSlice`를 쓰면 `counterSlicer.actions.increment`처럼 action creator가 자동으로 만들어지는데, 그걸 안 쓰고 `type` 문자열을 손으로 직접 적었다. 지금이야 짧아서 괜찮지만, action이 많아지면 오타 한 글자로 아무 일도 안 일어나는데 에러도 안 뜨는 상황이 생길 수 있다. `counterSlicer.jsx`에서 액션을 export 안 해서 이렇게 된 것 같은데, 아래처럼 export를 추가하고 가져다 쓰는 게 안전하다.

```js
// counterSlicer.jsx
export const { increment, decrement } = counterSlicer.actions;
export default counterSlicer.reducer;
```

**아쉬운 점 2 - `useDispatch` 훅 대신 store를 직접 import해서 씀**

```js
import {store} from "@/redux/store";
// ...
store.dispatch({type:'counter/increment'});
```

이렇게 store를 직접 불러다 `dispatch`를 호출해도 동작은 한다. 근데 원래 react-redux 쓰는 방식은 `useDispatch()` 훅으로 dispatch 함수를 받아오는 거다. store를 컴포넌트마다 직접 import하면 나중에 테스트 코드를 짜거나, store 구조가 바뀌었을 때 손댈 곳이 늘어난다는 걸 오늘 찾아보고 알았다.

**아쉬운 점 3 - payload를 쓰는 예제가 하나도 없음**

`increment`/`decrement`만 있다 보니 딱 정해진 값(1)만 왔다갔다 해서, payload가 왜 필요한지 체감이 안 됐다. 값을 받아서 처리하는 액션을 하나 추가해봐야 감이 올 것 같다.

**아쉬운 점 4 - reducer 안에 디버깅용 console.log가 남아있음**

`console.log('state', state)`, `console.log('action', action)`은 값 확인용으로 넣은 것 같은데, 실제로 배포할 코드라면 지우는 게 맞다. 대신 개발 중에 state/action 흐름을 보고 싶으면 Redux DevTools 확장 프로그램을 쓰는 게 더 편하다는 것도 오늘 새로 알았다.

> ✅ **핵심**: 동작은 하지만 컨벤션과 다른 부분(action creator 미사용, store 직접 import)은 나중에 코드가 커질수록 문제가 될 수 있는 지점이다. "돌아간다"랑 "제대로 짰다"는 다른 얘기라는 걸 다시 느꼈다.

---

## 🔧 개선한 버전으로 다시 짜보기

위에서 짚은 것들을 반영해서 다시 짜봤다. payload를 쓰는 `addByAmount`도 추가했다.

```js
// counterSlicer.jsx
import { createSlice } from "@reduxjs/toolkit";

const counterSlicer = createSlice({
  name: "counter",
  initialState: {
    value: 0,
  },
  reducers: {
    increment: (state) => {
      state.value += 1; // Immer가 알아서 새 state를 만들어주므로 return 안 해도 됨
    },
    decrement: (state) => {
      state.value -= 1;
    },
    addByAmount: (state, action) => {
      state.value += action.payload; // dispatch할 때 같이 보낸 값
    },
  },
});

export const { increment, decrement, addByAmount } = counterSlicer.actions;
export default counterSlicer.reducer;
```

```jsx
// page.jsx
'use client';
import { useSelector, useDispatch } from "react-redux";
import { increment, decrement, addByAmount } from "@/redux/counterSlicer";

export default function App() {
  const dispatch = useDispatch();
  const count = useSelector((state) => state.counter.value);

  return (
    <div>
      <h3>COUNT : {count}</h3>
      <button onClick={() => dispatch(increment())}>증가</button>
      <button onClick={() => dispatch(decrement())}>감소</button>
      <button onClick={() => dispatch(addByAmount(5))}>5만큼 증가</button>
    </div>
  );
}
```

`addByAmount(5)`를 호출하면 내부적으로 `{ type: 'counter/addByAmount', payload: 5 }`가 만들어지고, 이게 `dispatch`로 store에 전달된다. reducer는 `action.payload`(=5)를 꺼내서 `state.value`에 더한다. 이 한 줄 추가만으로 payload 개념이 훨씬 몸에 붙었다.

`store.jsx`랑 `layout.jsx`는 원래 구조가 정석대로 되어 있어서 그대로 둬도 될 것 같다.

> ✅ **핵심**: reducer 함수에서 `state.value += 1`처럼 직접 수정하는 것처럼 코드를 짜도 되는 이유는 Redux Toolkit이 내부적으로 Immer를 쓰기 때문이다. 겉보기엔 mutate 같지만 실제로는 새로운 state 객체를 만들어서 교체해준다.

---

## 🤔 감이 안 왔던 부분, 다시 짚어보기

- **Immer가 정확히 어떻게 "직접 수정하는 것처럼" 코드를 짜도 불변성을 지켜주는 건지** → state를 감싸는 draft 객체를 만들어서 변경 사항을 추적한 다음 새 객체를 생성해준다고 알고 있는데, 내부 동작 원리까지는 오늘 코드로 확인은 못 했다. 다음엔 `state.value += 1` 전후로 실제 객체 참조(`===` 비교)가 달라지는지 직접 찍어봐야겠다.
- **`useDispatch()` 훅과 store를 직접 import해서 `store.dispatch()`를 쓰는 것의 실질적인 차이** → 둘 다 동작은 하는데, 왜 공식 문서가 `useDispatch()`를 권장하는지 (테스트, SSR, Provider로 store를 교체하는 상황 등) 구체적인 케이스로 아직 체감을 못 했다. Next.js SSR 환경에서 store를 직접 import했을 때 실제로 문제가 생기는 예시를 다음에 찾아봐야겠다.
- **action type 문자열 네이밍 컨벤션** → `counter/increment`처럼 `슬라이스이름/함수이름` 형태가 `createSlice`가 자동으로 만들어준 건데, 이게 그냥 관례인지 아니면 Redux DevTools 같은 도구가 이 형식에 의존하는 부분이 있는지는 다음에 더 찾아봐야 할 것 같다.

---

## 💡 정리하면

오늘 실습을 다시 리뷰하면서 느낀 건, Redux 자체의 흐름(action → reducer → store → 구독)은 지난번에 얼추 이해했다고 생각했는데 payload처럼 "값을 실어 보내는" 케이스를 안 써봐서 절반만 이해한 상태였다는 거다. 직접 `addByAmount`를 추가해보고 나서야 "아 action이 그냥 데이터를 담아 보내는 봉투구나"라는 감이 확실히 왔다.

**오늘의 핵심 4가지**

- 📦 action은 `{ type, payload }` 형태의 평범한 객체다 - type은 "무슨 일이 일어났는지", payload는 "그 일에 딸려온 데이터"
- 🏭 `createSlice`는 action creator까지 자동으로 만들어주므로, `counterSlicer.actions`에서 꺼내 export해서 쓰는 게 안전하다 (type 문자열 직접 타이핑 금지)
- 🪝 컴포넌트에서 dispatch가 필요하면 store를 직접 import하지 말고 `useDispatch()` 훅을 쓰는 게 정석이다
- 🧬 Redux Toolkit + Immer 덕분에 reducer 안에서 `state.value += 1`처럼 직접 수정하듯 코드를 짜도 실제로는 불변성이 지켜진다
