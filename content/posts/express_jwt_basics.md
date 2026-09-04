+++
title = "[Express] JWT로 로그인 인증 흐름 정리 - sign, verify, 그리고 KEY 문제"
date = 2026-08-28
draft = false
tags = ["Express", "Node.js", "JWT", "백엔드"]
categories = ["dev"]
math = false
+++

저번 게시판 프로젝트에서 JWT를 붙이긴 했는데, 그땐 미들웨어 만들다가 `next()` 빠뜨린 거 고치기 바빠서 JWT 자체가 어떻게 동작하는지는 제대로 못 짚었다. 이번엔 로그인/토큰검증만 딱 떼어서 작은 예제로 다시 만들어보면서, `jwt.sign`과 `jwt.verify`가 각각 뭘 하는지, `.http` 파일로 실제 토큰을 눈으로 까보면서 정리했다.

**목차**

1. [🔑 로그인하면 토큰이 왜 발급되는가](#-로그인하면-토큰이-왜-발급되는가)
2. [🧩 실제 토큰을 까보니 - JWT의 3부분 구조](#-실제-토큰을-까보니---jwt의-3부분-구조)
3. [🛂 /check에서 토큰을 검증하는 방식](#-check에서-토큰을-검증하는-방식)
4. [⚠️ KEY를 서버 켤 때마다 새로 만들면 생기는 문제](#-key를-서버-켤-때마다-새로-만들면-생기는-문제)
5. [🧹 return 이후에 남아있던 죽은 코드](#-return-이후에-남아있던-죽은-코드)
6. [💡 정리하면](#-정리하면)

---

## 🔑 로그인하면 토큰이 왜 발급되는가

`/login` 라우터는 이렇게 생겼다.

```js
app.post('/login', (req, res) => {
    const {id, pw} = req.body;
    console.log(`${id}와 ${pw}를 이용해 db안에 회원이 있는지 확인`);
    // 로그인 했다고 가정하고 실습

    const token = jwt.sign({id, pw}, KEY, {expiresIn: '30m'});
    res.json({'success': true, 'token': token})
});
```

이번 실습에서는 DB 조회를 생략하고 "로그인 성공했다고 치고" 바로 토큰을 발급하는 구조였다. `jwt.sign(payload, key, options)` 이 세 인자가 각각 뭔지 헷갈렸는데, 정리하면:

- **payload** : 토큰 안에 담을 데이터 (`{id, pw}`)
- **key** : 이 토큰이 진짜인지 나중에 검증할 때 쓸 서명용 비밀값
- **options** : `expiresIn` 같은 부가 설정 (여기선 30분 후 만료)

`.http` 파일로 실제 로그인 요청을 보내보니, 응답으로 `token`이라는 긴 문자열이 왔다.

```http
### 1. 로그인
POST http://localhost/login
Content-Type: application/json

{
  "id":  "admin",
  "pw": "pass"
}
```

> ✅ **핵심**: `jwt.sign()`은 "이 사람이 로그인했다"는 정보(payload)를 KEY로 서명해서 하나의 문자열(토큰)로 만들어주는 함수다. 이 토큰을 클라이언트가 들고 있다가, 이후 요청마다 제출해서 "나 로그인한 사람 맞다"를 증명하는 방식이다.

---

## 🧩 실제 토큰을 까보니 - JWT의 3부분 구조

발급받은 토큰을 `.http` 파일에 그대로 복사해서 다음 요청에 썼는데, 자세히 보니 점(`.`) 두 개로 세 부분이 나뉘어 있었다.

```
eyJhbGciOiJIUzI1NiJ9.eyJpZCI6ImFkbWluIiwicHciOiJwYXNzIn0.jjdC6UU2TgQYgZ...
```

![JWT 토큰은 점 두 개로 Header, Payload, Signature 세 부분이 나뉜다. Header와 Payload는 Base64로 인코딩만 된 것이라 누구나 디코딩해서 내용을 읽을 수 있다](/images/express/express_jwt_structure.svg)

디코딩해보니 가운데(Payload) 부분에 `jwt.sign()`에 넣었던 `{id, pw}`가 그대로 들어있고, 발급 시각(`iat`)이랑 만료 시각(`exp`)도 자동으로 추가돼 있었다. 여기서 좀 놀랐던 건, Header랑 Payload는 **암호화가 아니라 그냥 Base64 인코딩**이라서 누구나 디코딩해서 읽을 수 있다는 거였다. 그래서 비밀번호처럼 진짜 민감한 값을 payload에 그대로 넣으면 안 된다는 걸 이번에 알게 됐다 — 지금 예제는 실습용이라 `pw`를 그대로 넣었지만, 실제 서비스에서는 `id`나 `role` 정도만 넣는 게 맞을 것 같다.

진짜 "검증"의 역할을 하는 건 마지막 Signature 부분이었다. Header와 Payload를 합쳐서 KEY로 서명한 값이라서, 중간에 누가 Payload 내용을 조작하면 이 서명 값이 달라지고, 검증할 때 걸러진다.

> ✅ **핵심**: JWT는 Header.Payload.Signature 세 부분으로 이뤄져 있고, 앞 두 개는 그냥 인코딩이라 내용이 노출된다. 실제 위변조를 막는 건 KEY로 만든 Signature 부분이다.

---

## 🛂 /check에서 토큰을 검증하는 방식

`/check` 라우터에서는 `jwt.verify()`로 받은 토큰이 진짜인지 확인하고 있었다.

```js
app.post('/check', (req, res) => {
    const token = req.headers.authorization;

    if (token == null){
        return res.json({'loginYN': false, 'msg': '토큰이 없습니다.'});
    }

    try{
        const info = jwt.verify(token, KEY);
        return res.json({'loginYN': true, 'data': '추가작업 결과'});
    }catch(e){
        return res.json({'loginYN': false, 'msg':'유효하지 않은 토큰입니다.'})
    }
});
```

`jwt.verify(token, KEY)`는 "이 토큰, 진짜로 이 KEY로 서명된 게 맞아?"를 확인하는 함수였다. 서명이 맞고 만료 시간도 안 지났으면 payload(`{id, pw, iat, exp}`)를 그대로 돌려주고, 둘 중 하나라도 문제 있으면(서명 불일치, 만료) 에러를 던진다. 그래서 이 함수를 `try/catch`로 감싸서, 에러가 나면 "유효하지 않은 토큰"으로 처리하는 구조였다.

`.http`에 있던 토큰을 그대로 붙여서 요청해보니 `loginYN: true`가 왔는데, 시간이 좀 지난 뒤(만료 시각 이후) 같은 토큰으로 다시 요청하니 `loginYN: false`로 바뀌었다. `expiresIn: '30m'`이 실제로 동작한다는 걸 눈으로 확인한 셈이다.

> ✅ **핵심**: `jwt.verify(token, key)`는 서명과 만료 여부를 확인해서, 문제없으면 payload를, 문제 있으면 에러를 던진다. 로그인 여부 체크는 결국 "이 토큰이 우리 서버가 발급한 게 맞고, 아직 안 만료됐는가"를 확인하는 것이다.

---

## ⚠️ KEY를 서버 켤 때마다 새로 만들면 생기는 문제

코드 맨 위에 있던 이 줄이 계속 마음에 걸렸다.

```js
// 서버를 켤 때 마다 새로 생성
const KEY = crypto.randomBytes(64).toString('hex');
console.log("sing key : ", KEY);
```

주석에도 적혀있듯이, 이 `KEY`는 서버가 켜질 때마다 랜덤하게 새로 만들어진다. 처음엔 "매번 새로 만드는 게 더 안전한 거 아닌가?" 싶었는데, 직접 서버를 껐다 켜보고 나서야 문제를 체감했다.

![로그인 시 서명한 KEY와, 서버가 재시작되어 새로 생성된 KEY가 달라지면 기존에 발급된 모든 토큰이 무효 처리된다. KEY는 .env 같은 곳에 고정값으로 저장해서 재사용해야 한다](/images/express/express_jwt_key_problem.svg)

- 서버 A 상태에서 로그인 → KEY-1로 서명된 토큰 발급
- 서버를 재시작하면 코드가 처음부터 다시 실행되면서 → KEY-2로 새로 생성
- 아까 받은 토큰을 그대로 들고 `/check`를 호출하면 → `jwt.verify(token, KEY-2)`인데 서명은 KEY-1로 됐던 거라 검증 실패

즉 서버를 재시작하는 순간, 그 전까지 로그인했던 사용자 전부가 강제로 로그아웃되는 것과 같은 상황이 벌어진다. 실습이니까 넘어갔지만, 실제 서비스라면 `.env` 파일 같은 곳에 KEY를 고정값으로 저장해두고, 서버가 재시작돼도 항상 같은 값을 읽어오게 만들어야 한다는 걸 확실히 이해했다. 매번 랜덤하게 만드는 게 "안전"이 아니라 오히려 "서명 키가 안 바뀌고 유지되는 것" 자체가 이 시스템이 정상 작동하기 위한 전제 조건이었다.

> ✅ **핵심**: JWT 서명 KEY는 로그인 시점과 검증 시점에 항상 같아야 한다. 코드 실행마다 랜덤 생성하면 서버 재시작 때마다 기존 토큰이 전부 무효화된다. 실제 서비스에서는 `.env` 등으로 고정해서 관리해야 한다.

---

## 🧹 return 이후에 남아있던 죽은 코드

작은 부분인데 눈에 띄어서 같이 짚어본다.

```js
try{
    const info = jwt.verify(token, KEY);
    return res.json({'loginYN': true, 'data': '추가작업 결과'});
}catch(e){
    return res.json({'loginYN': false, 'msg':'유효하지 않은 토큰입니다.'})
}

res.json({loginYN: true}); // 여기는 절대 실행 안 됨
```

이번엔 `try`와 `catch` 양쪽 다 `return`으로 확실히 끝내고 있어서, 저번에 겪었던 "이중 응답" 버그는 안 났다. 근데 그 아래 남아있는 `res.json({loginYN: true})`는 `try/catch` 양쪽 모두 `return`으로 끝나기 때문에 **어떤 경로로도 도달할 수 없는 코드**였다. 실행에는 문제가 없지만(그냥 안 쓰이는 코드일 뿐), 나중에 누가 이 코드를 보고 "어? 이건 왜 있지" 헷갈릴 수 있을 것 같아서 지우는 게 나아 보였다. 저번엔 `return`을 빠뜨려서 문제였다면, 이번엔 반대로 `return`을 다 붙이고 나니 그 뒤에 있던 코드가 무의미해진 경우였다.

> ✅ **핵심**: 모든 분기에서 `return`으로 끝나면 그 함수의 마지막에 남은 코드는 절대 실행되지 않는 죽은 코드(dead code)다. 에러는 안 나지만 가독성을 위해 정리하는 게 좋다.

---

## 💡 정리하면

이번엔 JWT를 로그인 기능 안에 섞어서 배우는 대신 `/login`, `/check` 두 개짜리 최소 예제로 따로 떼어보니, `sign`과 `verify`가 각각 뭘 하는지, KEY가 왜 그렇게 중요한지가 훨씬 선명하게 잡혔다. 특히 `.http` 파일로 실제 토큰을 직접 발급받고 디코딩해본 게, 그냥 개념 설명 듣는 것보다 훨씬 오래 기억에 남을 것 같다.

**오늘의 핵심 4가지**

- 🔑 `jwt.sign(payload, key, options)`은 로그인 정보를 KEY로 서명해서 토큰 문자열로 만든다
- 🧩 토큰은 Header.Payload.Signature 구조이고, 앞 두 부분은 암호화가 아니라 인코딩이라 누구나 읽을 수 있다 - 민감한 값은 payload에 넣지 않아야 한다
- 🛂 `jwt.verify(token, key)`는 서명과 만료 여부를 확인하고, 문제 있으면 에러를 던지므로 `try/catch`로 감싼다
- ⚠️ 서명 KEY는 로그인 시점과 검증 시점에 항상 같아야 하며, 실행마다 새로 만들면 서버 재시작 시 기존 토큰이 전부 무효화된다
