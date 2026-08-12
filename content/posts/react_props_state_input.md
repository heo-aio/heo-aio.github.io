+++
title = "[React] Props vs State, 그리고 제어 컴포넌트 - 데이터는 누가 갖고, 화면은 언제 다시 그려지나"
date = 2026-08-13
draft = false
tags = ["React", "Props", "State", "useState", "제어컴포넌트"]
categories = ["dev"]
math = false
+++

1편에서는 "화면을 어떻게 그리는가(JSX, 스타일)"를 봤다면, 이번 실습은 "데이터가 어디 있고 어떻게 흐르는가"에 대한 이야기였다. `StateBtn.jsx`와 `StateBtn_2.jsx`가 거의 똑같이 생겼는데 하나는 버튼이 눌러도 화면이 안 바뀐다는 걸 직접 보고 나서야, "리렌더링이 대체 왜 되고 안 되고 하는지"가 감으로 잡혔다.

**목차**
1. [Props - 부모가 자식에게 내려주는 값](#-props---부모가-자식에게-내려주는-값)
2. [State - 왜 let cnt-- 로는 화면이 안 바뀌는가](#-state---왜-let-cnt--로는-화면이-안-바뀌는가)
3. [제어 컴포넌트 - input의 value와 onChange는 세트다](#-제어-컴포넌트---input의-value와-onchange는-세트다)
4. [여러 개의 input, state를 하나로 묶을까 나눌까](#-여러-개의-input-state를-하나로-묶을까-나눌까)
5. [감이 안 왔던 부분, 다시 짚어보기](#-감이-안-왔던-부분-다시-짚어보기)
6. [정리하면](#-정리하면)

---

## 🎁 Props - 부모가 자식에게 내려주는 값

```jsx
// App.jsx
<PropBtn name = "This is Prop Button"/>

// PropBtn.jsx
const PropBtn = ({name}) => {
    const sendMsg = (name) => {
        alert(`Your name is ${name}`);
    }
    return (<div>
        <button onClick={()=>{sendMsg(name)}}>{name}</button>
    </div>);
}
```

`App`이 `<PropBtn name="This is Prop Button"/>`처럼 태그의 속성(attribute)처럼 값을 넘기면, `PropBtn` 쪽에서는 `({name})`으로 그 값을 매개변수처럼 받는다. 이게 Props다. 주석에 적혀 있던 "Props : 부모가 자식에게 보내는 값"이라는 한 줄이 정확한 정의였다.

중요한 건 **방향이 한쪽뿐**이라는 점이다. `PropBtn` 안에서 `name`을 마음대로 바꿀 방법이 없다. 만약 표시되는 이름을 바꾸고 싶다면, `PropBtn`이 스스로 바꾸는 게 아니라 부모인 `App`이 새로운 값을 다시 내려줘야 한다. 함수에 인자를 넘기는 것과 비슷한데, 그 인자를 함수 내부에서 재할당할 수 없는 것과 같은 느낌이라고 생각하니 이해가 빨랐다.

---

## 🔄 State - 왜 `let cnt--` 로는 화면이 안 바뀌는가

이번 실습에서 제일 인상 깊었던 비교가 이거였다. 코드가 아주 살짝만 다른 두 컴포넌트를 나란히 두고 버튼을 눌러봤다.

```jsx
// StateBtn.jsx - useState 사용
import {useState} from 'react'
export default function StateBtn(){
    const[cnt, setCnt] = useState(100)
    const updateCnt = () => { setCnt(cnt - 1); }
    return (<button onClick={()=>updateCnt()}>down count : {cnt}</button>);
}
```

```jsx
// StateBtn_2.jsx - 그냥 변수
export default function StateBtn(){
    let cnt = 100;
    const updateCnt = () => {
        cnt--;
        console.log(cnt); // 콘솔에는 잘 찍힌다
    }
    return (<button onClick={()=>updateCnt()}>down count : {cnt}</button>);
}
```

`StateBtn_2`도 버튼을 누르면 `cnt`가 99, 98, 97... 로 분명히 줄어드는 게 `console.log`로 확인된다. 그런데 화면의 숫자는 100에서 꼼짝을 안 한다. 처음엔 "값이 바뀌는데 왜 화면은 그대로지?"가 이해가 안 됐는데, 답은 **"React는 값이 바뀐다고 화면을 다시 그리지 않는다. `setState` 함수가 호출되어야만 그 컴포넌트를 다시 실행(render)한다"**는 것이었다.

`let cnt`는 그냥 JS 변수라서, 값은 메모리 어딘가에서 바뀌고 있지만 React 입장에서는 "다시 그려야 할 이유"를 전혀 모른다. 반면 `setCnt(cnt - 1)`을 호출하면 React가 그걸 신호로 받아서 컴포넌트 함수를 처음부터 다시 실행하고(리렌더링), 그 결과로 화면의 `{cnt}` 부분이 새 값으로 바뀐다.

![state를 통해 setter를 호출하면 리렌더링이 일어나지만, 일반 변수는 값이 바뀌어도 화면에 반영되지 않는 과정 비교](/images/react/props_vs_state.svg)

주석에 있던 "hooks는 갈고리처럼 돌아서 간다는 뜻(편법)"이라는 표현도 재밌었는데, 원래 클래스형 컴포넌트에서만 가능했던 "이 컴포넌트가 자기 값을 기억하고, 바뀌면 다시 그리는" 기능을 함수형 컴포넌트에서도 쓸 수 있게 만든 게 `useState` 같은 hook이라는 배경까지 같이 이해됐다.

> ✅ **핵심**: 화면을 다시 그리게 만드는 건 "값이 바뀜"이 아니라 "`setState` 함수의 호출"이다. `useState`로 관리하지 않는 변수는 값이 바뀌어도 React가 알 방법이 없어서 화면에 반영되지 않는다.

---

## ⌨️ 제어 컴포넌트 - input의 value와 onChange는 세트다

```jsx
// input.jsx
const [text, setText] = useState()

const getText = (e) => {
    setText(e.target.value); // state안에 값을 넣어줘야 UI에 적용
}

<input value={text} onChange={(e)=>{getText(e)}} />
```

처음엔 `value={text}`만 있으면 되는 줄 알았는데, 그렇게만 하면 **입력 자체가 안 먹힌다.** 타자를 쳐도 아무 반응이 없는 걸 보고서야 이유를 이해했다 - `value`를 state로 고정해버리면, 사용자가 타이핑을 해도 React 입장에서는 "화면에 보여줄 값은 여전히 state 값"이라고 우겨서 입력이 튕겨 나온다. `onChange`가 있어야 "사용자가 입력한 값을 즉시 state에 반영"해주고, 그 반영된 state가 다시 `value`로 화면에 표시되는 순환 구조가 완성된다.

![state가 value로 화면에 표시되고, 사용자 입력이 onChange를 통해 다시 state로 들어가는 제어 컴포넌트의 순환 구조](/images/react/controlled_input_loop.svg)

- `value`: state → 화면 (state가 진짜 값의 주인)
- `onChange`: 화면 입력 → state (사용자 입력을 state에 반영)

이런 식으로 **input의 값을 React state가 완전히 통제(control)하는 패턴**을 "제어 컴포넌트(Controlled Component)"라고 부른다는 것도 이번에 알게 됐다. HTML의 순수 `<input>`은 자기 자신의 값을 스스로 기억하지만(비제어), React에서는 그 값을 state로 옮겨서 "진짜 값의 주인은 항상 state"라는 규칙을 만든 것이다.

> ✅ **핵심**: `value`만 있고 `onChange`가 없으면 입력이 막힌다. `value`(state→화면)와 `onChange`(화면→state)가 짝을 이뤄야 사용자가 입력한 값이 진짜로 반영된다. 이게 제어 컴포넌트 패턴이다.

---

## 🧩 여러 개의 input, state를 하나로 묶을까 나눌까

`inputs.jsx`와 `inputs_review.jsx`를 나란히 보면서 같은 문제(아이디+닉네임 입력받기)를 두 가지 방식으로 푼 걸 비교해볼 수 있었다.

```jsx
// inputs.jsx - 객체 하나로 묶기 + 동적 key
const [inputs, setInputs] = useState({nick : "", id : ""});

const typing = function(key, e){
    setInputs({
        ...inputs,               // 기존 값 전부 복사
        [key] : e.target.value   // 그 중 key에 해당하는 것만 덮어쓰기
    })
}
// onChange = {(e)=>{typing('id', e)}}
// onChange = {(e)=>{typing('nick', e)}}
```

```jsx
// inputs_review.jsx - state를 필드별로 나누기
const [id, setId] = useState("");
const [nick, setNick] = useState("");

const login_Id = (e) => { setId(e.target.value); };
const login_Nick = (e) => { setNick(e.target.value); };
```

`inputs.jsx`의 주석 "요소가 늘어날 때마다 state를 추가해 줄 수 없다"가 이 코드의 의도를 정확히 설명해줬다. 입력 필드가 2개면 `useState`를 2번 쓰면 되지만, 10개, 20개가 되면 `typing` 함수 하나로 모든 필드를 처리하고 싶어진다. 그래서 상태를 객체 하나(`inputs`)로 묶고, `[key]`라는 **계산된 속성명(computed property name)** 문법으로 "지금 어떤 필드가 바뀌었는지"를 함수 호출 시점에 결정한다.

여기서 `...inputs`(스프레드 연산자)가 왜 꼭 필요한지도 짚였다. `setInputs({[key]: e.target.value})`처럼 스프레드 없이 바로 넣으면 방금 바뀐 필드 하나만 남고 나머지 필드(`id` 입력 중이면 `nick`)는 통째로 사라져 버린다. `...inputs`로 기존 객체를 먼저 펼쳐 넣고 그 위에 `[key]`만 덮어써야, "나머지는 그대로 두고 이 필드만 바꾼다"는 게 실현된다.

반면 `inputs_review.jsx`는 필드별로 `useState`를 따로따로 뒀다. 코드가 더 단순하고 직관적이지만, 필드가 많아지면 `useState` 선언과 핸들러 함수가 그만큼 늘어난다. 둘을 비교해보니, **필드가 몇 개 안 되고 로직이 단순하면 나눠서 관리하는 쪽이 읽기 쉽고, 필드가 많고 패턴이 반복되면 객체+동적 key로 묶는 쪽이 코드량을 줄여준다**는 트레이드오프가 보였다.

> ✅ **핵심**: input이 여러 개면 state를 객체 하나로 묶고 `[key]`(계산된 속성명)로 어떤 필드가 바뀌는지 함수 호출 시점에 결정할 수 있다. 이때 `...기존객체`로 먼저 펼쳐야 나머지 필드가 안 날아간다. 필드 수가 적으면 그냥 나눠서 관리하는 쪽이 더 읽기 쉬울 수도 있다.

---

## 🤔 감이 안 왔던 부분, 다시 짚어보기

- **state 객체를 바꿀 때 왜 `inputs.id = e.target.value`처럼 직접 수정하면 안 되나?** → 이번 실습엔 명시적으로 안 나왔지만 `setInputs({...inputs, [key]: ...})` 패턴을 보면서 든 궁금증이다. 추측으로는 React가 "값이 바뀌었는지"를 판단할 때 객체를 통째로 새로 만들어서 비교하는 방식(불변성, immutability)을 쓰기 때문에, 기존 객체를 직접 건드리면 React가 변화를 감지 못 할 것 같다. 다음에 이 부분을 직접 실험해서 확인해봐야겠다.
- **Props로 받은 값을 자식이 "바꾸고 싶을 때"는 실무에서 보통 어떻게 하나?** → 오늘 배운 규칙대로면 자식은 못 바꾸고 부모가 새로 내려줘야 하는데, 그러면 자식에서 부모의 값을 바꾸도록 요청하는 방법이 있을 것 같다(콜백 함수를 Props로 내려주는 방식?). 다음 실습에서 나오면 State와 엮어서 다시 정리해야겠다.
- **`typing('id', e)`처럼 함수를 호출할 때, `e`(이벤트 객체)는 왜 신경 안 써도 자동으로 들어오나?** → `inputs.jsx`의 `init` 함수 주석("매개변수를 아무것도 안 줘도 이벤트 객체는 무조건 받아올 수 있다")이 힌트였는데, 정확히 왜 그런지는 다음에 이벤트 핸들러 등록 방식(`onClick={init}` vs `onClick={()=>init()}`)을 비교하면서 더 파봐야 할 것 같다.

---

## 💡 정리하면

이번 실습으로 "React에서 데이터가 어떻게 움직이는가"에 대한 큰 그림이 잡혔다. Props는 위에서 아래로 흐르는 읽기 전용 값이고, State는 컴포넌트가 스스로 들고 있으면서 setter로 바꿔야만 화면이 갱신되는 값이다. 그리고 input처럼 사용자가 직접 값을 바꾸는 요소는, `value`(state→화면)와 `onChange`(화면→state)를 세트로 묶어야 진짜로 동작한다는 걸 실습으로 확인했다.

**오늘의 핵심 5가지**

- 🎁 Props는 부모→자식 단방향 전달, 자식은 받은 값을 바꿀 수 없다
- 🔄 화면을 다시 그리게 만드는 건 "값이 바뀜"이 아니라 "setter 함수 호출" - 일반 변수는 값이 바뀌어도 화면엔 반영 안 됨
- ⌨️ 제어 컴포넌트: `value`(state→화면)와 `onChange`(화면→state)가 짝을 이뤄야 입력이 실제로 동작한다
- 🧩 입력 필드가 많으면 state를 객체로 묶고 `[key]` 계산된 속성명으로 관리 - 이때 `...기존객체` 스프레드는 필수
- ⚖️ state를 필드별로 나눌지 객체로 묶을지는 필드 개수와 반복 패턴에 따른 트레이드오프

**다음에 확인해볼 것**

1. state 객체를 직접 수정(`inputs.id = ...`)했을 때 실제로 리렌더링이 안 되는지 콘솔로 확인해보기
2. 자식이 부모의 state를 바꾸고 싶을 때 쓰는 패턴(콜백을 Props로 내려주기) 다음 실습에서 확인하고 정리하기
3. `onClick={init}`과 `onClick={()=>init()}`의 차이, 이벤트 객체가 자동으로 전달되는 원리 파보기
