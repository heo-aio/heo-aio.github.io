+++
title = "[JavaScript] 실행 컨텍스트와 this - 호출 스택, apply/call/bind, 화살표 함수의 this"
date = 2026-08-06T22:30:00+09:00
draft = false
tags = ["JavaScript", "실행컨텍스트", "this", "apply", "call", "bind"]
categories = ["dev"]
math = false
+++

이번 실습은 `this`였는데, 막상 정리하려고 보니 실행 컨텍스트(execution context) 이야기부터 짚고 가야 순서가 맞겠다 싶었다. 지금까지 "왜 스코프 체인이 안쪽에서 바깥쪽으로 검색되는지", "왜 화살표 함수는 hoisting이 안 되는지"를 현상으로만 외웠는데, 이번에 그 뒤에 있는 **실행 컨텍스트**라는 개념을 보고 나니 조각들이 하나로 이어졌다.

**목차**
1. [실행 컨텍스트란 - 호출 스택](#-실행-컨텍스트란---호출-스택)
2. [호이스팅의 진짜 의미 - 생성 단계에서 벌어지는 일](#-호이스팅의-진짜-의미---생성-단계에서-벌어지는-일)
3. [this 기본 규칙 - 누가 호출했는가](#-this-기본-규칙---누가-호출했는가)
4. [this를 강제로 바꾸기 - apply, call, bind](#-this를-강제로-바꾸기---apply-call-bind)
5. [콜백에서 this를 잃어버리는 문제](#-콜백에서-this를-잃어버리는-문제)
6. [addEventListener에서 this - 일반 함수 vs 화살표 함수](#-addeventlistener에서-this---일반-함수-vs-화살표-함수)
7. [감이 안 왔던 부분, 다시 짚어보기](#-감이-안-왔던-부분-다시-짚어보기)
8. [정리하면](#-정리하면)

---

## 📚 실행 컨텍스트란 - 호출 스택

```js
const global = 10;
console.log(global); // 10 [전역 context]

outerFunc(); // [전역 context, outerFunc]

function outerFunc(){
    const global = 20;
    console.log(global); // 20
    innerFunc(); // [전역 context, outerFunc, innerFunc]

    function innerFunc(){
        const local2 = 50;
        console.log(global); // 20
    } // 종료 -> [전역 context, outerFunc]
} // 종료 -> [전역 context]
```

실습 코드 옆에 달려 있던 `// [전역 context, outerFunc, innerFunc]` 같은 주석이 힌트였다. 함수를 호출할 때마다 **실행 컨텍스트**라는 게 하나씩 만들어져서 쌓이고, 함수가 끝나면 그 컨텍스트만 쏙 빠진다. 이게 정확히 **스택(Stack, LIFO - 나중에 들어간 게 먼저 나감)** 구조라는 걸 그림으로 그려보고 나서야 지난번 정리했던 스코프 체인이 왜 "가까운 데부터 바깥으로" 검색되는지도 같이 이해됐다 - 지금 스택 맨 위에 있는 컨텍스트부터, 그 아래 컨텍스트들을 순서대로 살펴보는 것이었다.

![실행 컨텍스트가 함수 호출마다 스택에 쌓이고 종료 순서대로 빠지는 과정](/images/js_dom/execution_context_stack.svg)

> ✅ **핵심**: 함수를 호출할 때마다 실행 컨텍스트가 스택에 쌓이고, 함수가 끝나면 가장 최근에 쌓인 것부터 빠진다. 스코프 체인 검색도 결국 이 스택 구조 위에서 일어나는 일이다.

---

## 🎁 호이스팅의 진짜 의미 - 생성 단계에서 벌어지는 일

```js
test(); // 정상 실행
function test(){ console.log('test function !'); }

console.log('선언 전 : ', elice);  // undefined (에러 아님!)
var elice = 'hello';

console.log('선언 전 : ', elice3); // ReferenceError
const elice3 = 'hello';

// let과 const는 호이스팅이 지원되지 않네? -> 아니야, 되는거야
// var는 선언부를 등록하고 기본값을 undefined로 넣는 것이고,
// let과 const는 선언부를 등록하고 기본값을 안 넣는거야
```

이 실습 코드의 주석이 그동안의 오해를 정확히 짚어줬다. **"let/const는 hoisting이 안 된다"는 말은 반은 틀렸다.** 정확히는, 실행 컨텍스트가 만들어지는 **생성 단계**에서 `var`/`let`/`const` **셋 다 선언부는 미리 등록**된다. 차이는 그 시점에 값을 넣어주느냐다:

![var는 undefined로 초기화까지, let/const는 선언만 등록하고 초기화는 안 하는 차이](/images/js_dom/hoisting_creation_phase.svg)

- `var elice` → 생성 단계에서 선언 등록 + **`undefined`로 초기화까지** 미리 해둠 → 선언 전에 읽어도 에러 없이 `undefined`
- `const elice3` → 생성 단계에서 선언만 등록, **초기화는 안 함** (TDZ 상태) → 선언 전에 읽으면 `ReferenceError`

지난 ES6 정리에서 "화살표 함수는 hoisting이 안 된다"고 썼던 것도 사실은 이 얘기였다 - 화살표 함수 자체가 아니라, 그게 `const`/`let`으로 선언된 변수라서 TDZ의 적용을 받는 것이었다. 함수 선언식(`function test(){}`)만 유일하게 **선언 + 함수 본문 전체**까지 미리 등록되기 때문에 정의보다 먼저 호출해도 정상 동작한다.

> ✅ **핵심**: `var`/`let`/`const` 모두 선언부는 호이스팅된다. 차이는 `var`만 `undefined`로 초기화까지 되고, `let`/`const`는 초기화를 안 해서 TDZ 구간이 생긴다는 것.

---

## 🙋 this 기본 규칙 - 누가 호출했는가

```js
const obj = {
    name: 'Elice', age: 20,
    introduce: function(){
        console.log(`My name is ${this.name}`); // this = obj
    }
};
obj.introduce(); // this는 자신이 소속된 상위 객체

function outerFunc(){
    console.log(this); // window - 소속 객체가 없으니까
    innerFunc();
    function innerFunc(){
        console.log(this); // window - 중첩 함수도 마찬가지
    }
}
```

`this`는 "그 함수가 어디에 정의됐는가"가 아니라 **"누가 그 함수를 호출했는가"**로 결정된다는 게 핵심이었다. `obj.introduce()`처럼 객체를 통해 호출하면 `this`는 그 객체(`obj`)가 되고, `outerFunc()`처럼 아무 객체도 없이 그냥 호출하면 소속이 없으니 최상위 객체인 `window`가 된다. 파이썬의 `self`와 비슷한 역할이라고 생각하니 감이 잡혔는데, 차이는 파이썬은 `self`를 항상 명시적으로 첫 인자로 받는 반면 JS의 `this`는 **호출 방식에 따라 암묵적으로 정해진다**는 점이었다.

> ✅ **핵심**: `this`는 함수가 정의된 위치가 아니라 호출된 방식(누가 호출했는가)에 따라 결정된다. 객체를 통해 호출하면 그 객체, 그냥 호출하면 `window`.

---

## 🔧 this를 강제로 바꾸기 - apply, call, bind

```js
function introduce(){
    console.log(`My name is ${this.name} and I am ${this.age} years old`);
}

introduce.apply(hellobit);           // 바로 실행, 인자는 배열로
introduce.call(caterpillar);         // 바로 실행, 인자는 콤마로 나열

const func = introduce.bind(cheshire); // 실행 X, 함수만 반환
func(); // 나중에 이걸 호출해야 실행됨

function addArgs(a, b){ console.log(this, a, b); }
addArgs.apply(hellobit, [1, 2]);
addArgs.call(caterpillar, 3, 4);
addArgs.bind(cheshire)(5, 6);        // bind 즉시 호출도 가능
```

세 개가 다 "this를 강제로 지정한다"는 같은 목적을 갖는데, **인자를 넘기는 방식**과 **즉시 실행 여부**가 다르다.

![apply(배열, 즉시실행) vs call(콤마, 즉시실행) vs bind(즉시실행 안함, 함수 반환)](/images/js_dom/apply_call_bind.svg)

- `apply(this로쓸객체, [인자들])` - 인자를 배열로 묶어서, **즉시 실행**
- `call(this로쓸객체, 인자1, 인자2, ...)` - 인자를 콤마로 나열해서, **즉시 실행**
- `bind(this로쓸객체)` - 인자는 나중에, **실행하지 않고 "실행 가능한 함수"만 반환**

`apply`는 Array(배열), `call`은 Comma(콤마)로 외우면 헷갈리지 않는다는 실습 주석이 실제로 잘 외워졌다. `bind`는 이벤트 핸들러 등록처럼 **"지금 당장은 아니고 나중에 호출될 함수"**를 만들어둘 때 특히 유용하다는 것도 다음 파트(콜백 문제)에서 이어서 확인했다.

> ✅ **핵심**: `apply`/`call`은 즉시 실행되고 인자 전달 방식만 다르다(배열 vs 콤마). `bind`는 실행하지 않고 this가 고정된 새 함수를 돌려준다.

---

## 🧩 콜백에서 this를 잃어버리는 문제

```js
function square(callback){
    const result = callback();
    return result * result;
}

// 문제 상황
const calc = {
    a: 3, b: 5,
    calculate: function(){
        return square(function(){
            return this.a + this.b;  // this가 window를 봄 -> NaN
        });
    }
};
console.log(calc.calculate()); // NaN
```

이 코드가 왜 `NaN`이 나오는지 처음엔 이해가 안 됐는데, **"콜백 함수는 `square()` 안에서 `callback()`으로 호출되는 것"**이라는 걸 깨닫고 나서야 감이 왔다. `calc.calculate()`를 호출했다고 해서 그 안의 콜백 함수까지 `calc`에 소속되는 게 아니다 - 콜백은 `square` 함수 내부에서 그냥 `callback()`으로 호출되니, 앞서 배운 규칙대로 `this`는 `window`가 된다.

```js
// 해결방법 1 - bind로 강제 연결
calculate: function(){
    return square((function(){
        return this.a + this.b;
    }).bind(this));  // 여기의 this는 calc (calculate가 calc.calculate()로 호출됐으므로)
}

// 해결방법 2 - 화살표 함수 (더 깔끔)
calculate: function(){
    return square(() => this.a + this.b);  // 화살표 함수는 상위(calculate)의 this를 그대로 씀
}
```

두 해결법을 나란히 보니 **화살표 함수가 왜 만들어졌는지** 이제야 확실히 이해됐다. `bind(this)`로 매번 강제 연결하는 대신, 애초에 "자신만의 `this`를 만들지 않는" 화살표 함수를 쓰면 자연스럽게 상위 스코프(`calculate`가 호출됐을 때의 `this`, 즉 `calc`)를 그대로 물려받는다. 콜백 안에서 `this`를 쓰고 싶을 땐 화살표 함수가 정답에 가깝다는 걸 실감했다.

> ✅ **핵심**: 콜백 함수는 자신을 호출한 쪽(보통 라이브러리/헬퍼 함수 내부)을 기준으로 `this`가 정해지기 때문에 의도한 객체를 잃어버리기 쉽다. `bind`로 강제 연결하거나, 화살표 함수로 애초에 그 문제를 피할 수 있다.

---

## 🖱️ addEventListener에서 this - 일반 함수 vs 화살표 함수

```js
btn.addEventListener('click', function(evt){
    console.log(this); // this는 evt.currentTarget과 연결 -> btn
});

btn.addEventListener('click', evt => console.log(this)); // this는 window (정의된 시점의 상위)
```

1부 정리 글에서 "화살표 함수는 이벤트 당사자가 아닌 `window`를 가리킨다"고 짚었던 부분이 사실 이 콜백 `this` 문제와 같은 원리였다는 걸 이번에 명확히 연결할 수 있었다. `function(evt){}`로 등록하면 브라우저가 호출할 때 `this`를 리스너가 걸린 요소(`btn`)로 세팅해주지만, 화살표 함수는 그 세팅을 무시하고 **자기가 정의된 위치(전역)의 `this`**를 그대로 쓴다.

> ✅ **핵심**: `addEventListener`의 콜백은 일반 함수면 `this = 리스너가 걸린 요소`, 화살표 함수면 `this = 정의된 위치의 상위 this`(보통 `window`)다.

---

## 🤔 감이 안 왔던 부분, 다시 짚어보기

- **스코프 체인과 실행 컨텍스트 스택은 같은 건가, 다른 건가?** → 처음엔 같은 이야기인 줄 알았는데, 이번에 정리하면서 관계가 좀 더 명확해졌다. **실행 컨텍스트 스택**은 "지금 어떤 함수들이 호출되어 있는가"에 대한 이야기(시간 순서, 호출 순서)이고, **스코프 체인**은 "변수를 어디까지 찾아볼 수 있는가"에 대한 이야기(코드가 어디에 작성되어 있는가, 즉 정적인 구조)다. 대부분 겹쳐 보이지만, 클로저처럼 함수가 실행 컨텍스트 스택에서는 사라졌는데도 스코프 체인(그 함수가 정의된 환경)은 여전히 참조되는 경우도 있다고 하니, 다음엔 클로저 예제로 이 차이를 직접 확인해봐야겠다.
- **`bind`와 화살표 함수 중 뭘 언제 써야 하나?** → 이번 실습을 보니 "미리 만들어서 여러 번 재사용할 함수"라면 `bind`, "그 자리에서 바로 쓰고 말 콜백"이라면 화살표 함수가 더 간결한 것 같다. 다만 이건 아직 감으로만 정리한 거라, 다음엔 실제 프로젝트 코드에서 어떤 기준으로 선택하는지 좀 더 찾아봐야겠다.
- **화살표 함수를 객체의 메서드로 쓰면 어떻게 될까?** → 이번 실습엔 없었지만 규칙대로 유추해보면, `const obj = { name: 'A', say: () => console.log(this.name) }`처럼 메서드 자체를 화살표 함수로 만들면 `say()`를 호출해도 `this`가 `obj`가 아니라 화살표 함수가 정의된 시점의 상위(전역)를 가리킬 것 같다. 다음에 직접 콘솔로 확인해서 "객체 메서드는 화살표 함수로 쓰면 안 되는 이유"를 실습해봐야겠다.

---

## 💡 정리하면

이번 실습을 계기로 그동안 따로따로 외웠던 것들 - 스코프 체인, 화살표 함수의 hoisting, `this` - 이 사실 **실행 컨텍스트**라는 하나의 개념 위에서 벌어지는 현상들이었다는 걸 알게 됐다. "함수가 호출되면 컨텍스트가 스택에 쌓이고, 그 컨텍스트가 생성될 때 변수 등록/초기화(hoisting)와 `this` 결정이 함께 일어난다"는 큰 그림으로 이해하니 각각의 규칙들이 왜 그렇게 동작하는지가 훨씬 납득이 갔다.

**오늘의 핵심 6가지**

- 📚 함수 호출마다 실행 컨텍스트가 스택에 쌓이고, 가장 최근 것부터 빠진다 (LIFO)
- 🎁 `var`/`let`/`const` 모두 선언은 호이스팅되지만, `var`만 `undefined`로 초기화까지 되고 `let`/`const`는 TDZ에 걸린다
- 🙋 `this`는 함수가 호출된 방식(누가 호출했는가)으로 결정된다
- 🔧 `apply`/`call`은 즉시 실행 + 인자 전달 방식(배열 vs 콤마)만 다르고, `bind`는 실행 없이 함수를 반환한다
- 🧩 콜백 안에서 `this`가 예상과 다르면 `bind`로 강제 연결하거나 화살표 함수로 상위 `this`를 물려받아 해결한다
- 🖱️ `addEventListener` 콜백은 일반 함수면 리스너가 걸린 요소, 화살표 함수면 상위 스코프의 `this`

**다음에 확인해볼 것**

1. 클로저 예제로 "실행 컨텍스트는 스택에서 사라졌는데 스코프 체인은 남아있는" 상황을 직접 만들어보기
2. 객체 메서드를 화살표 함수로 정의했을 때 `this`가 실제로 어떻게 되는지 콘솔로 확인해보기
3. `bind`와 화살표 함수 중 어떤 걸 언제 쓰는 게 관례인지 실제 오픈소스/프로젝트 코드에서 찾아보기
