+++
title = "[JavaScript] ES6 핵심 문법 정리 - 스코프 체인, 구조분해, spread/rest, 배열 고차함수까지"
date = 2026-08-06T10:00:00+09:00
draft = false
tags = ["JavaScript", "ES6", "스코프체인", "구조분해할당", "배열메서드"]
categories = ["dev"]
math = false
+++

오늘 실습은 범위가 꽤 넓었다. `let/const/var`부터 시작해서 스코프 체인, 화살표 함수, optional chaining, 구조분해 할당, spread/rest, 객체 축약, 고차함수, 그리고 `map/filter/reduce`까지. 하나하나는 문법 자체가 어렵진 않은데, **왜 이렇게 동작하는지 이유를 안 짚고 넘어가면 나중에 응용할 때 막힌다**는 걸 실습 코드를 다시 보면서 느꼈다. 그래서 이번에도 문법 나열이 아니라, 실습 중에 헷갈렸거나 "아 이래서 이렇게 동작하는구나" 싶었던 지점 위주로 정리한다.

**목차**
1. [let, var, const - 재할당의 차이](#-let-var-const---재할당의-차이)
2. [Scope Chain - 변수는 어떻게 찾아지는가](#-scope-chain---변수는-어떻게-찾아지는가)
3. [화살표 함수 - 두 가지 함정 (this, hoisting)](#-화살표-함수---두-가지-함정-this-hoisting)
4. [Optional Chaining - null/undefined 안전하게 접근하기](#-optional-chaining---nullundefined-안전하게-접근하기)
5. [구조분해 할당 - 객체/배열, 함수 인자까지](#-구조분해-할당---객체배열-함수-인자까지)
6. [Spread/Rest 연산자 - 펼치기와 모으기](#-spreadrest-연산자---펼치기와-모으기)
7. [객체 축약 표현](#-객체-축약-표현)
8. [고차함수와 콜백, 그리고 map/filter/reduce](#-고차함수와-콜백-그리고-mapfilterreduce)
9. [감이 안 왔던 부분, 다시 짚어보기](#-감이-안-왔던-부분-다시-짚어보기)
10. [정리하면](#-정리하면)

---

## 📦 let, var, const - 재할당의 차이

```js
let global = '전역 변수';

function test(){
    let global = '지역 변수'; // let은 철저히 영역에 제한을 받는다.
    console.log(global);
}
test();          // 지역 변수
console.log(global); // 전역 변수

function test2(){
    if (true) {
        var scope = '함수 영역'; // 블록 영향을 받긴 하는데, 함수 안에서는 안 받는다.
        let local = '블록 영역'; // 철저히 블록 영역을 벗어나지 못한다.
    }
    console.log(scope); // 정상 출력
    // console.log(local); // ReferenceError
}
test2();

const x = 10;
x = 30; // TypeError: Assignment to constant variable.
```

- `let`은 **블록({}) 단위**로 스코프가 갇힌다. `if` 블록 안에서 선언해도 블록을 벗어나면 못 씀
- `var`는 **함수 단위**로만 스코프가 나뉜다. `if` 블록 안에서 선언해도 함수 전체에서 접근 가능
- `const`는 재할당 자체가 불가능. 마지막 줄 `x = 30`은 실행하면 바로 `TypeError`가 난다 - 실습 코드에 일부러 넣어둔 에러 케이스인 듯

여기서 처음엔 "그럼 그냥 다 `let` 쓰면 되지 않나" 싶었는데, 실습을 다시 보면서 `var`가 왜 위험한지 감이 왔다. `var scope`는 `if` 블록 밖에서도 살아남는데, 이게 **의도한 동작이 아니라 "새어나간" 것**이다. 반면 `let local`은 블록을 나가면 확실히 사라진다. 그래서 요즘은 기본을 `const`로 두고, 재할당이 필요할 때만 `let`을 쓰는 게 정석이라고 한다.

> ✅ **핵심**: `let`/`const`는 블록 스코프, `var`는 함수 스코프. `const`는 재할당하면 바로 에러가 난다.

---

## 🔍 Scope Chain - 변수는 어떻게 찾아지는가

```js
const global = 10;
console.log(global); // 10

outerFunc();

function outerFunc(){
    const global = 20;
    const local1 = 10;
    console.log(global); // 20
    console.log(local1); // 10
    innerFunc();

    function innerFunc(){
        const local1 = 40;
        const local2 = 50;
        console.log(global); // 20 ← innerFunc엔 없어서 outerFunc 것을 가져온다
        console.log(local1); // 40
        console.log(local2); // 50
    }
}
```

Scope Chain은 **변수를 찾는 순서**에 대한 이야기다. `innerFunc`에서 `global`을 출력하면 `innerFunc` 안에는 `global`이 없으니, 한 단계 바깥인 `outerFunc`의 `global = 20`을 찾아서 쓴다. 만약 `outerFunc`에도 없었다면 그다음 바깥인 전역까지 나가서 찾았을 것이다.

![스코프 체인 - 변수를 안쪽부터 바깥쪽으로 검색하는 과정](/images/js_dom/scope_chain.svg)

여기서 헷갈렸던 부분은 "그럼 `innerFunc`이 `outerFunc`을 거쳐서 전역까지 매번 다 확인하나?"였는데, 정확히는 **가까운 스코프부터 순서대로 한 단계씩** 확인하다가 찾으면 그 즉시 멈춘다. 이름이 겹치면 항상 **가장 가까운 것이 우선**이라는 점도 다시 짚었다.

> ✅ **핵심**: 변수 검색은 안쪽 → 바깥쪽 순서로 진행되고, 이름이 겹치면 가장 가까운 스코프의 값이 우선한다.

---

## ➡️ 화살표 함수 - 두 가지 함정 (this, hoisting)

```js
const x1 = (x, y) => { console.log(x + y); };       // 규칙1: function 생략
const x2 = (x, y) => console.log(x + y);              // 규칙2: 실행문 1개면 {} 생략
const x3 = x => console.log(x * x);                    // 규칙3: 매개변수 1개면 () 생략

// 차이점 1: this가 이벤트 당사자가 아닌 window(상위 스코프의 this)를 가리킨다
btn.addEventListener('click', e => console.log(this));

// 차이점 2: hoisting을 지원하지 않는다
hoisting();               // 정상 실행 - function 선언은 호이스팅됨
function hoisting(){
    console.log('hoist : 끌어올려지다');
}

x4(3);                    // ReferenceError!
const x4 = x => console.log(x * x);
```

화살표 함수는 문법 줄이기 그 이상으로, **동작 자체가 일반 함수와 다르다는 걸** 이번에 제대로 짚었다. 특히 실습 코드 마지막에 `x4(3)`을 먼저 호출하고 나중에 `const x4 = ...`를 정의해서 일부러 에러를 내는 부분이 있는데, 이게 "화살표 함수는 hoisting이 안 된다"를 눈으로 확인시켜주는 장치였다.

![function 선언식과 const 화살표함수의 실행 순서 차이 - hoisting과 TDZ 비교](/images/js_dom/hoisting_vs_tdz.svg)

- `function hoisting(){...}` 처럼 **함수 선언식**은 함수 전체가 코드 실행 전에 메모리에 먼저 올라간다 → 정의보다 먼저 호출해도 정상 동작
- `const x4 = x => ...` 는 **변수 선언**이라서, 선언은 되지만 그 줄에 도달하기 전까지는 `TDZ(Temporal Dead Zone)` 상태 → 먼저 호출하면 `ReferenceError`
- `this` 문제도 마찬가지로 자주 걸리는 함정인데, 화살표 함수는 자신만의 `this`를 만들지 않고 **정의된 시점의 상위 스코프 this**를 그대로 쓴다. 그래서 이벤트 핸들러에서 "이벤트를 발생시킨 요소"를 기대하고 `this`를 썼다가 `window`가 나와서 당황하기 쉽다

> ✅ **핵심**: 화살표 함수는 (1) 자신만의 `this`가 없고 상위 스코프의 `this`를 그대로 쓰며, (2) hoisting되지 않아서 정의 이전에 호출하면 에러가 난다.

---

## ❓ Optional Chaining - null/undefined 안전하게 접근하기

```js
const user = {
    profile: { name: 'Alice' },
    number: [1, 2, 3]
};

console.log(user.profile.name);        // 'Alice'
console.log(user.profile.age);         // undefined - profile까진 있는데 age만 없음
console.log(user.contact?.email);      // undefined - contact 자체가 없어도 에러 없이 undefined
console.log(user.number[1]);           // 2
console.log(user.number[4]?.toString()); // undefined
console.log(user.number?.[4]);         // undefined
```

`user.contact.email`처럼 `?.` 없이 접근하면 `contact` 자체가 없기 때문에 **"Cannot read properties of undefined"** 에러가 난다. 반면 `user.contact?.email`은 `contact`가 없으면 그 자리에서 조회를 멈추고 `undefined`를 반환한다. `user.number?.[4]`처럼 대괄호 접근에도 `?.[]` 형태로 쓸 수 있다는 걸 이번에 새로 알았다.

> ✅ **핵심**: `?.`은 중간 경로가 없을 때 에러 대신 `undefined`를 반환해서 코드가 죽는 걸 막아준다. 객체 접근(`?.`)과 배열 인덱스 접근(`?.[]`) 둘 다 지원한다.

---

## 🧩 구조분해 할당 - 객체/배열, 함수 인자까지

```js
const user = { name: '홍길동', age: '25', city: 'Seoul' };

// 이름 바꿔서 꺼내기 (키:임시이름)
const { name: userName, city } = user;

// 중첩 객체 + 기본값
const person = {
    name: 'Jhone', age: 30, city: 'NewYork',
    contact: { email: 'Jhone@elice.io', phone: '02-6954-4788' }
};
const { name: p_name, contact: { email }, grade = 'A' } = person;
// grade는 person에 없어서 기본값 'A'가 들어간다

// 나머지(rest) 문법
const { name: n, age: a, ...etc } = person;
console.log(etc); // { city: 'NewYork', contact: {...} }
```

구조분해는 **"속성명(키)이 곧 변수명이 되어야 한다"**는 규칙만 이해하면 나머지는 응용이다. 이름이 겹치면 `키: 임시이름`으로 바꿔 쓰고, 없는 키를 꺼내려 하면 `기본값 = ...`으로 대비할 수 있다. `...etc`처럼 나머지를 한 번에 묶는 것도 "선택 안 된 것들을 모은다"는 개념으로 이해하니 자연스러웠다.

함수 인자에서도 똑같이 쓸 수 있다는 게 실무적으로 제일 유용했다:

```js
function getUser({ name, age }) {
    console.log(name, age);
}
getUser(user); // 객체 전체를 넘겨도 함수 안에서 필요한 것만 뽑아 쓴다

// 이벤트 객체에서도 바로 구조분해
p.addEventListener('click', ({ target: { innerHTML, className } }) => {
    console.log(innerHTML, className);
});

// 배열도 동일한 원리, 다만 키가 아니라 "순서"로 매칭된다
const numbers = [1, 2, 3, 4, 5];
const [first, second, ...rest] = numbers;
// first=1, second=2, rest=[3,4,5]
```

객체 구조분해는 "키 이름"으로 매칭되고, 배열 구조분해는 "순서(인덱스)"로 매칭된다는 차이를 확실히 짚어야 헷갈리지 않는다.

> ✅ **핵심**: 구조분해는 객체=키 매칭, 배열=순서 매칭이며, 함수 인자·이벤트 객체에도 그대로 적용된다.

---

## 🌊 Spread/Rest 연산자 - 펼치기와 모으기

같은 `...`인데 위치에 따라 반대로 동작한다. **배열/객체 리터럴 안에서는 펼치기(spread)**, **함수 매개변수에서는 모으기(rest)**다.

```js
const arr1 = [1, 2, 3];
const arr2 = [10, 20, 30];

console.log([...arr1, ...arr2]);   // [1,2,3,10,20,30] - concat과 같은 결과

// 얕은 복사(shallow copy)로 원본과 분리된 새 배열이 생긴다
let x = [1, 2, 3, 4, 5];
let y = [...x];
x[4] = 'X';
y[0] = 'Y';
console.log(x); // [1,2,3,4,'X']  - y의 변경과 무관
console.log(y); // ['Y',2,3,4,5]  - x의 변경과 무관
```

![spread로 배열을 복사하면 서로 다른 메모리를 가진 독립된 배열이 된다](/images/js_dom/spread_copy.svg)

이 부분이 실습에서 제일 "아하"했던 지점이다. `y = x`였다면 `x`와 `y`가 **같은 배열을 참조**해서 한쪽을 바꾸면 다른 쪽도 같이 바뀌었을 텐데, `y = [...x]`는 **새로운 배열을 만들어서 값만 복사**하기 때문에 완전히 독립적으로 움직인다. React에서 상태(state)를 직접 수정하지 않고 `[...state]`로 복사해서 바꾸는 이유가 바로 이거였구나 싶었다.

함수 쪽 rest 문법도 짚어보면:

```js
function total(...params) {       // 파이썬의 *params와 같은 개념
    let result = 0;
    for (const p of params) result += parseInt(p);
    return result;
}
total(10, 20);       // 30
total(1, 2, 3, 4, 5); // 15

// spread copy - 객체 안에 객체로 들어가지 않고 같은 선상의 속성으로 병합된다
const user = { name: '홍길동', age: 25 };
let person = { user, city: '서울' };       // { user: {...}, city: '서울' } - 중첩됨
person = { ...user, city: '서울' };        // { name, age, city } - 평탄하게 병합됨
```

> ✅ **핵심**: `...`는 배열/객체 리터럴에서는 "펼치기", 함수 매개변수에서는 "모으기"로 동작한다. spread로 만든 복사본은 원본과 메모리가 분리된다.

---

## ✂️ 객체 축약 표현

```js
const name = 'Elice';
const age = 20;

let person = { name: name, age: age };  // 기존 방식
let person2 = { name, age };             // 축약 - 키와 변수명이 같으면 생략 가능
```

변수명과 넣고 싶은 키 이름이 같을 때만 쓸 수 있는 문법이라, 큰 개념은 아니지만 실무 코드에서 정말 자주 보이는 패턴이라 짚어둔다.

> ✅ **핵심**: `{ name: name }`처럼 키와 값 변수명이 같으면 `{ name }`으로 줄여 쓸 수 있다.

---

## 🔁 고차함수와 콜백, 그리고 map/filter/reduce

```js
function repeat(n, action) {   // action 자리에 함수를 넘겨받는 "고차함수"
    for (let i = 0; i < n; i++) action(i);
}
repeat(3, n => console.log(`Hello - ${n}`));

btn.addEventListener('click', e => console.log('버튼 클릭')); // addEventListener도 고차함수다
```

`repeat` 함수를 보고서야 `addEventListener`, `map`, `filter` 같은 것들이 왜 다 "함수를 인자로 넘긴다"는 공통점을 갖는지 이해가 됐다. **함수를 값처럼 다른 함수에 전달할 수 있다(고차함수)**는 개념이 먼저 있고, 그 위에 `map/filter/reduce`가 얹혀 있는 구조였다.

```js
const info_list = [
    { id: 1, name: '허정모', grade: 'A', score: 95 },
    { id: 2, name: '박하영', grade: 'B', score: 82 },
    { id: 3, name: '윤규리', grade: 'D', score: 68 },
    { id: 4, name: '정효준', grade: 'A', score: 93 },
    { id: 5, name: '이승혁', grade: 'F', score: 50 },
    { id: 6, name: '이영호', grade: 'C', score: 72 }
];

let students = info_list.map(item => { item.name += '학생'; return item; });
let ids = info_list.map(item => item.id);

let supple = students.filter(info => info.score < 70);
let students_name = supple.map(item => item.name);

let scores = [1,2,3,4,5,6,7,8,9,10];
let sum = scores.reduce((acc, curr) => acc + curr);
```

![info_list를 map, filter, map으로 이어서 가공하는 데이터 파이프라인](/images/js_dom/array_pipeline.svg)

세 메서드를 헷갈릴 때마다 "**결과 배열의 길이가 어떻게 되어야 하는가**"를 먼저 생각하니 구분이 쉬워졌다.

- `map` - 개수는 그대로, 각 요소를 원하는 형태로 가공 (`item.id`만 뽑기, `name`에 문자열 붙이기 등)
- `filter` - 조건을 만족하는 것만 남겨서 개수가 줄어들 수 있음 (`score < 70`인 학생만)
- `reduce` - 배열 전체를 하나의 값으로 누적 (합계, 최댓값, 평균 등)

`reduce`의 `(acc, curr)`가 처음엔 헷갈렸는데, "초기값을 안 주면 배열의 0번째 값이 `acc`의 시작값이 되고, 1번째 요소부터 `curr`로 순회한다"는 걸 콘솔 로그로 직접 찍어보고서야 감이 왔다.

> ✅ **핵심**: 고차함수는 "함수를 값처럼 주고받는 함수"이고, `map/filter/reduce`는 그 위에서 각각 변환·필터링·누적이라는 역할로 나뉜다. 결과 배열의 길이 변화로 셋을 구분하면 헷갈리지 않는다.

---

## 🤔 감이 안 왔던 부분, 다시 짚어보기

- **왜 화살표 함수는 hoisting이 안 될까?** → 정확히는 "화살표 함수라서"가 아니라 **`const`/`let`으로 선언된 변수라서**다. 화살표 함수든 일반 함수 표현식이든, `함수 선언문`이 아니라 `변수에 함수를 대입하는 형태`라면 전부 TDZ의 적용을 받는다. "화살표 함수 = hoisting 안 됨"이 아니라 "변수 선언 = hoisting 안 됨"으로 이해하는 게 더 정확했다.
- **spread로 복사하면 진짜 완전히 독립적인가?** → 정확히는 **1depth(바로 아래 단계)까지만** 독립적이다(얕은 복사). `info_list.map(item => {...})`에서 `item.name += '학생'`처럼 배열 안의 객체 자체를 직접 수정하면, 원본 `info_list`의 객체도 같이 바뀐다. 오늘 실습 코드에서 `students`를 만든 뒤에도 `info_list`의 `name`이 이미 바뀌어 있던 이유가 이거였다. 다음엔 `item => ({...item, name: item.name + '학생'})`처럼 객체 자체도 새로 만들어야 완전히 분리된다는 걸 직접 콘솔로 확인해봐야겠다.
- **구조분해와 spread는 뭐가 다른가?** → 구조분해(`const {a} = obj`)는 "꺼내서 변수에 담는 것"이고, spread(`{...obj}`)는 "펼쳐서 새 객체/배열에 합치는 것"이다. 방향이 반대라고 생각하니 헷갈리지 않았다.

---

## 💡 정리하면

오늘 다룬 문법들은 따로따로 보면 각각 외워야 할 규칙처럼 보이지만, 실습 코드를 순서대로 다시 보니 하나의 흐름으로 이어져 있었다: **스코프(변수가 어디서 보이는가) → 함수 표현 방식(화살표 함수의 hoisting/this 차이) → 데이터를 안전하게 꺼내는 법(optional chaining, 구조분해) → 데이터를 복사·재조합하는 법(spread/rest) → 데이터를 가공하는 법(map/filter/reduce)**. 결국 다 "데이터를 어떻게 안전하고 예측 가능하게 다루는가"로 수렴하는 느낌이다.

**오늘의 핵심 5가지**

- 📦 `const`는 재할당 불가, `let`은 블록 스코프, `var`는 함수 스코프
- 🔍 변수 검색은 스코프 체인을 타고 안쪽 → 바깥쪽으로, 가장 가까운 값이 우선한다
- ➡️ 화살표 함수는 자신만의 `this`가 없고, `const`/`let` 선언이라 hoisting도 안 된다
- 🌊 spread로 만든 복사본은 원본과 메모리가 분리되지만, **얕은 복사**라 내부 객체까지는 분리되지 않는다
- 🔁 `map`은 개수 유지, `filter`는 개수 감소 가능, `reduce`는 값 하나로 누적 - 셋 다 "함수를 인자로 받는 고차함수"라는 공통 뿌리를 갖는다

**다음에 확인해볼 것**

1. `item => ({...item, name: ...})` 형태로 얕은 복사의 한계를 직접 콘솔로 재현해보기
2. `reduce`에 초기값을 명시했을 때/안 했을 때 `acc`, `curr`가 어떻게 달라지는지 로그 찍어 비교하기
3. 화살표 함수의 `this` 문제를 실제 이벤트 핸들러에서 `bind`, 일반 함수와 비교해서 정리해보기
