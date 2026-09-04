+++
title = "[Express] 라우팅 기초 정리 - Router 분리부터 params/query/body, 미들웨어 순서까지"
date = 2026-08-27
draft = false
tags = ["Express", "Node.js", "백엔드"]
categories = ["dev"]
math = false
+++

강의에서 `app.get`, `app.post` 치는 건 금방 따라했는데, 막상 `req.params`랑 `req.query`가 뭐가 다른지, `router.js`로 왜 파일을 나누는지, 미들웨어가 왜 순서대로 실행되는지는 헷갈렸다. 그래서 실습 파일 4개를 다시 열어보면서 하나씩 짚어봤다.

**목차**

1. [🚏 GET/POST/ALL - 같은 경로도 메서드로 나뉜다](#-getpostall---같은-경로도-메서드로-나뉜다)
2. [📦 router.js로 분리하는 이유](#-routerjs로-분리하는-이유)
3. [🔍 req.params vs req.query vs req.body](#-reqparams-vs-reqquery-vs-reqbody)
4. [🚧 정의 안 된 경로 처리하기](#-정의-안-된-경로-처리하기)
5. [🧱 미들웨어는 등록한 순서대로 실행된다](#-미들웨어는-등록한-순서대로-실행된다)
6. [💡 정리하면](#-정리하면)

---

## 🚏 GET/POST/ALL - 같은 경로도 메서드로 나뉜다

첫 실습(`index.js`)에서 제일 먼저 눈에 들어온 건 `/hello`라는 같은 경로인데 메서드에 따라 다른 응답이 나간다는 거였다.

```js
app.get('/hello', (req, res) => {
    res.send('<h1>Hello World, For GET!!!</h1>');
});

app.post('/hello', (req, res) => {
    res.send('Hello World, For Post!!!');
});
```

경로만 보고 라우터가 정해지는 게 아니라 **"경로 + 메서드"** 조합으로 라우터가 정해진다는 걸 여기서 체감했다. 그리고 `app.all('/test', ...)`은 GET이든 POST든 PUT이든 상관없이 `/test`로만 오면 다 받는 거였다. 나중에 인증 체크처럼 메서드 상관없이 공통으로 걸어야 하는 로직에 쓰기 좋겠다는 생각이 들었다.

`res.send()`는 문자열/HTML을 그대로 반환하고, `res.json()`은 JSON 형태로 반환한다는 차이도 같이 정리됐다. 지금 짜는 건 거의 API라서 앞으로는 `res.json()`을 기본으로 쓰게 될 것 같다.

> ✅ **핵심**: Express 라우팅은 경로 하나가 아니라 "경로 + HTTP 메서드"로 구분된다. `app.all()`은 메서드 상관없이 그 경로로 오는 모든 요청을 받는다.

---

## 📦 router.js로 분리하는 이유

`index.js`에 라우트를 계속 늘려 쓰다가, `routers.js`를 따로 만들어서 연결하는 방식도 같이 실습했다.

```js
// routers.js
const router = express.Router();

router.get('/hello', function(req, res){
    res.send("Router Module, GET !!");
});

module.exports = router;
```

```js
// index.js
const router = require('./routers');
app.use('/route', router);
```

`app.use('/route', router)`로 마운트하면, `router.js` 안에서 정의한 `'/hello'`는 실제로는 `/route/hello`로 동작한다. `express.Router()`는 `app`이 하던 라우팅 기능만 따로 떼어낸 미니 인스턴스라서, 기능(회원/게시글 등)별로 파일을 쪼개기 좋다는 걸 알게 됐다. 아직 라우트 개수가 적어서 크게 체감은 안 되지만, 파일 하나에 계속 쌓이면 나중에 관리가 안 될 것 같아서 미리 익혀두길 잘한 것 같다.

> ✅ **핵심**: `express.Router()`로 라우트를 모듈 단위로 쪼갤 수 있고, `app.use(마운트경로, router)`로 연결하면 그 마운트 경로가 라우터 안 경로 앞에 자동으로 붙는다.

---

## 🔍 req.params vs req.query vs req.body

이번 실습에서 제일 헷갈렸던 부분이다. 셋 다 "요청에서 값을 꺼내는 것"인데 값이 어디에 담겨 오느냐가 달랐다.

```js
// params - URL 경로 자체에 값이 있음
// GET /rest/admin/pass
app.get('/rest/:id/:pw', function(req, res){
    const {id, pw} = req.params;
    res.json({'params': {id, pw}});
});

// query - ? 뒤에 key=value로 붙는 값
// GET /get_method?id=admin&pw=pass
app.get('/get_method', function(req, res){
    const {id, pw, mm} = req.query;
    res.json({'query':{id, pw, mm}});
});

// body - 요청 본문에 JSON으로 담겨오는 값
// POST /login  { id:"admin", pw:"pass" }
app.use(express.json());
app.post('/login', function(req, res){
    const {id, pw} = req.body;
    res.json({"body": {id, pw}});
});
```

![req.params, req.query, req.body 비교. params는 URL 경로 자체에, query는 ? 뒤에, body는 요청 본문에 값이 담겨 온다. body를 쓰려면 express.json() 미들웨어가 반드시 필요하다](/images/express/express_params_query_body.svg)

여기서 제일 놓치기 쉬웠던 부분은 `req.body`를 쓰려면 `app.use(express.json())`을 먼저 등록해야 한다는 거였다. 이걸 안 붙이고 `req.body`를 찍어보면 `undefined`가 나오는데, 처음엔 "왜 값이 안 들어오지" 하고 당황했다. 알고 보니 요청 본문을 파싱해서 `req.body`에 넣어주는 역할 자체를 `express.json()` 미들웨어가 하는 거였다.

> ✅ **핵심**: params는 경로 자체, query는 `?` 뒤 문자열, body는 요청 본문. body를 쓰려면 `express.json()` 미들웨어 등록이 먼저다. 이거 하나 빠뜨리면 값이 조용히 `undefined`로 들어온다.

---

## 🚧 정의 안 된 경로 처리하기

`index.js`에서 마지막에 이런 코드가 있었다.

```js
app.use('/*path', function(req, res){
    res.send('잘못된 요청 입니다.');
});
```

처음엔 이게 왜 필요한지 몰랐는데, 이걸 제일 아래에 등록해두면 위에서 정의한 라우트 어디에도 안 걸린 요청이 여기로 흘러와서 처리된다는 걸 알게 됐다. 라우트 순서 위에서부터 매칭된다는 걸 이해하고 나니, "정의 안 된 경로는 맨 아래 와일드카드로 잡는다"는 패턴이 자연스럽게 이해됐다. 만약 이 코드를 라우트들보다 위에 놨으면 모든 요청이 여기서 잡혀버려서 다른 라우트는 실행될 기회도 없었을 거다.

> ✅ **핵심**: 와일드카드로 처리하는 404성 라우트는 반드시 다른 라우트들보다 아래에 위치해야 한다. Express는 등록된 순서대로 매칭을 시도한다.

---

## 🧱 미들웨어는 등록한 순서대로 실행된다

마지막 실습 파일이 제일 흥미로웠다. `app.use()`를 라우터 앞뒤로 하나씩 넣어서 실행 순서를 직접 확인해봤다.

```js
app.use(function (req, res, next) {
    console.log('@pre Handler');
    if (req.query.grade !== 'S') {
        res.status(403).json({'msg': '접근권한이 없습니다.'});
    } else {
        next();
    }
});

app.get('/', (req, res, next) => {
    console.log('@router');
    res.send('라우터 접근과 반환');
    next();
});

app.use((req, res) => {
    console.log('@Post Handler');
    console.log('일처리 후 뒷정리');
})
```

`?grade=S`로 요청을 보내면 콘솔에 `@pre Handler → @router → @Post Handler` 순서로 찍혔다. 등록한 순서 그대로 실행된다는 걸 눈으로 확인한 셈이다.

![미들웨어 실행 순서. app.use()로 등록한 pre-handler가 먼저 실행되고, next()를 호출해야 라우터로 넘어가며, 라우터에서 res.send 이후 next()를 또 호출하면 post-handler도 실행된다](/images/express/express_middleware_order.svg)

근데 좀 이상했던 부분이, 라우터 핸들러에서 이미 `res.send()`로 응답을 보냈는데 그 뒤에 `next()`를 또 호출한다는 거였다. 실행해보니 에러는 안 났다. 왜냐하면 뒤의 post-handler가 `console.log`만 찍고 `res.send`나 `res.json`으로 응답을 다시 보내려는 시도를 안 했기 때문이었다. 만약 여기서 응답을 한 번 더 보내려고 했다면 저번 게시판 프로젝트 때 만났던 `Cannot set headers after they are sent` 에러가 똑같이 났을 거다. 응답을 이미 보낸 뒤에 `next()`를 부르는 게 문법적으로 틀린 건 아니지만, 그 다음 핸들러에서 또 응답을 시도하면 위험하다는 걸 알게 됐다.

`grade`가 `'S'`가 아닐 때는 `res.status(403)`만 보내고 `next()`를 안 부르니까 라우터(`@router`)는 아예 실행되지 않았다. 이걸로 미들웨어가 "관문" 역할을 한다는 게 확실히 이해됐다 — `next()`를 부르지 않으면 그 뒤에 있는 어떤 코드도 실행이 안 된다.

> ✅ **핵심**: `app.use()`로 등록한 미들웨어는 등록한 순서대로 실행되고, `next()`를 호출해야만 다음 단계로 넘어간다. 응답을 이미 보낸 뒤에 또 응답을 시도하는 코드가 뒤따라오지 않는 한 `res.send` 이후 `next()`를 불러도 당장 에러는 안 나지만, 습관적으로 응답 이후엔 `return`으로 끝내는 게 안전할 것 같다.

---

## 💡 정리하면

오늘 실습은 화려한 기능보다는 "요청이 들어와서 응답이 나가기까지 어떤 순서로 코드가 실행되는가"를 확인하는 시간이었다. 특히 미들웨어 순서랑 `next()` 호출 여부가 이후에 만들 인증 로직(로그인 체크 등)의 기반이 될 것 같아서, 지금 확실히 짚고 넘어간 게 다행이라는 생각이 든다.

**오늘의 핵심 4가지**

- 🚏 라우팅은 "경로 + 메서드" 조합으로 구분되고, `app.all()`은 메서드 무관하게 잡는다
- 📦 `express.Router()`로 라우트를 파일 단위로 분리하고, `app.use(마운트경로, router)`로 연결한다
- 🔍 params(경로) / query(? 뒤) / body(요청 본문) — body는 `express.json()` 없이는 `undefined`
- 🧱 미들웨어는 등록 순서대로 실행되고, `next()`를 호출해야만 다음 단계로 넘어간다
