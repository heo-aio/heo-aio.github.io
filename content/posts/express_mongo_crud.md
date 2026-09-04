+++
title = "[Express + Mongoose] 회원 CRUD API 만들면서 정리한 것들"
date = 2026-08-28
draft = false
tags = ["Express", "Node.js", "MongoDB", "Mongoose", "백엔드"]
categories = ["dev"]
math = false
+++

지난번엔 메모리 배열로만 게시판을 만들었는데, 이번엔 진짜 DB(MongoDB)를 붙여서 회원 CRUD를 만들어봤다. `mongoose`라는 라이브러리가 처음이라 스키마 정의부터 낯설었고, 만들다 보니 예전에 게시판 프로젝트에서 겪었던 버그가 여기서도 똑같이 튀어나와서 반가우면서도 뜨끔했다. 그 과정을 정리한다.

**목차**

1. [🗂️ 전체 구조 - Router와 Model이 나뉘는 이유](#-전체-구조---router와-model이-나뉘는-이유)
2. [📐 mongoose 스키마 - DB한테 규칙을 미리 알려주는 것](#-mongoose-스키마---db한테-규칙을-미리-알려주는-것)
3. [🔁 저번에 배운 버그가 여기서도 나왔다 - return 누락](#-저번에-배운-버그가-여기서도-나왔다---return-누락)
4. [🚨 몽구스 에러코드로 중복 아이디 잡아내기](#-몽구스-에러코드로-중복-아이디-잡아내기)
5. [🧹 비밀번호는 응답에서 왜 빼야 하나](#-비밀번호는-응답에서-왜-빼야-하나)
6. [💡 정리하면](#-정리하면)

---

## 🗂️ 전체 구조 - Router와 Model이 나뉘는 이유

파일이 `app.js`, `db.js`, `model.js`, `member_router.js` 이렇게 나뉘어 있는데, 처음엔 이걸 왜 이렇게까지 쪼개나 싶었다. 근데 각자 역할이 명확히 갈려 있었다.

- `db.js` : MongoDB에 실제로 접속하는 것만 담당 (`connectDB()`)
- `model.js` : 데이터가 어떤 모양이어야 하는지(스키마) 정의만 담당
- `member_router.js` : 요청을 받아서 model을 통해 DB를 조작하는 실제 로직
- `app.js` : 이 모든 걸 연결하고 서버를 켜는 역할

```js
// app.js
app.use(cors());
app.use(express.json());
app.use('/member', require('./member_router'));
connectDB();
```

`app.use('/member', require('./member_router'))`로 라우터를 연결하는 방식은 저번 게시판 프로젝트 때 했던 것과 똑같았다. 다만 이번엔 라우터 안에서 배열 대신 `Member` 모델을 불러와서 DB를 직접 조작한다는 게 차이였다.

![Client에서 요청이 들어오면 Express(app.js) → 라우터(member_router.js) → 모델(model.js)을 거쳐 MongoDB까지 도달하는 흐름. db.js는 앱 시작 시 미리 연결을 맺어둔다](/images/express/express_mongo_arch.svg)

> ✅ **핵심**: DB 연결(`db.js`), 데이터 형태 정의(`model.js`), 실제 요청 처리(`router.js`)를 파일로 나누면, 나중에 "이 데이터 규칙이 뭐였지"는 model만, "이 요청 어떻게 처리하지"는 router만 보면 돼서 헷갈릴 일이 줄어든다.

---

## 📐 mongoose 스키마 - DB한테 규칙을 미리 알려주는 것

`model.js`를 보면서 제일 신기했던 건 mongoDB 자체는 원래 자유로운 형식(스키마리스)인데, `mongoose`를 쓰면 테이블처럼 규칙을 미리 정해둘 수 있다는 거였다.

```js
let schema = new mongoose.Schema({
    id: {
        type: String,
        required: [true, '아이디는 필수 사항입니다.'],
        unique: true,
        trim: true,
        minlength: [4, '아이디는 4자 이상입니다.'],
        maxlength: [25, '아이디는 25자 이하입니다.']
    },
    pw: {
        type: String,
        required: [true, '비밀번호는 필수 입니다.'],
        select: false // 조회할 때 기본적으로 빼고 가져온다.
    },
    ...
    grade: {
        type: String,
        default: 'user',
        enum: ['user', 'admin']
    }
}, {
    collection: 'member',
    timestamps: true,
    id: false
});
```

`required`, `unique`, `minlength/maxlength`, `enum` 같은 옵션들이 그냥 문서용 주석이 아니라 실제로 저장 시점에 검증까지 해준다는 게 인상적이었다. `grade`처럼 `enum`으로 값을 제한해두면 `'user'`, `'admin'` 말고 다른 값이 들어오려고 하면 애초에 저장이 안 된다.

`select: false`도 처음 보는 옵션이었는데, 회원 조회할 때 `pw`가 기본적으로 결과에 안 딸려온다는 뜻이었다. 근데 `join`(가입) 시점에는 어차피 `Member.create()`로 막 만든 결과라 `pw`가 그대로 들어있어서, 라우터 코드에서 `delete object.pw`로 한 번 더 명시적으로 지워주고 있었다. `select:false`는 "조회(find류)" 시점에만 막아주는 옵션이라 이 둘이 별개로 필요하다는 걸 알게 됐다.

> ✅ **핵심**: mongoose 스키마는 몽고DB에 "이 컬렉션엔 이런 규칙의 데이터만 들어올 수 있다"고 미리 선언하는 역할을 한다. `required`/`unique`/`enum` 같은 옵션은 실제 저장 시점에 검증까지 관여한다.

---

## 🔁 저번에 배운 버그가 여기서도 나왔다 - return 누락

`update`, `delete`, `get/:id` 라우터를 보다가 낯익은 패턴을 발견했다.

```js
router.put('/update/:id', async function(req, res){
    ...
    const member = await Member.findOneAndUpdate({id}, update, {new: true, runValidators: true}).lean();

    if (member == null){
        res.json({'success': false, 'msg': '없는 회원'});
    }
    res.json({'success': true, 'msg': '수정에 성공했습니다.', data: member});
});
```

지난 게시판 프로젝트 때 겪었던 "응답을 두 번 보내는 버그"랑 완전히 같은 패턴이었다. `member`가 `null`이면 `if` 블록에서 `res.json()`을 한 번 보내는데, `return`이 없어서 그 아래에 있는 `res.json()`이 이어서 또 실행되려고 한다. 존재하는 회원을 수정할 땐 안 걸리지만, 없는 `id`로 수정 요청을 보내면 그 순간 `Cannot set headers after they are sent` 에러가 날 거다. `delete`, `get/:id` 라우터에도 똑같은 패턴이 그대로 있었다.

![return 없이 조건부 응답을 처리하면 member가 null일 때 두 res.json이 모두 실행을 시도해 에러가 난다. if 블록의 응답 앞에 return을 붙이면 해결된다](/images/express/express_return_bug_recap.svg)

```js
// 수정
if (member == null){
    return res.json({'success': false, 'msg': '없는 회원'});
}
return res.json({'success': true, 'msg': '수정에 성공했습니다.', data: member});
```

한 번 겪어본 버그라서 그런지 이번엔 "왜 이런 에러가 나지" 하고 헤매지 않고, 코드를 보자마자 "아 이거 return 빠졌네"가 바로 보였다. 예전에 시간 들여서 원인 찾았던 게 그냥 날린 시간이 아니었다는 게 체감됐다.

> ✅ **핵심**: 조건 분기마다 응답을 보내는 코드에서는 `return`을 꼭 붙여야 한다. 이번엔 정상 케이스에서는 안 터지고 "존재하지 않는 id로 요청했을 때"만 터지는 조건부 버그라서, 테스트할 때 정상 케이스만 확인하면 놓치기 쉽다는 것도 새로 느꼈다.

---

## 🚨 몽구스 에러코드로 중복 아이디 잡아내기

`join` 라우터의 `catch` 블록도 눈에 띄었다.

```js
router.post('/join', async (req, res)=>{
    const {id, pw, name, phone} = req.body;
    try{
        let result = await Member.create({id, pw, name, phone});
        let object = result.toObject();
        delete object.pw;
        res.json({'join success':true, 'data': object});
    }catch(e) {
        let msg = "";
        switch (e.code) {
            case 11000:
                msg = "이미 사용중인 아이디 입니다.";
                break;
            default:
                msg = "필수 값을 확인해 주세요";
        }
        res.json({'success': false, message:msg});
    }
});
```

스키마에서 `unique: true`로 걸어둔 `id`가 중복되면, `Member.create()`가 그냥 조용히 실패하는 게 아니라 에러를 던진다는 걸 여기서 처음 알았다. 그 에러 객체의 `e.code`가 `11000`이면 "중복 키 에러"라는 몽고DB 고유의 에러 코드였다. 이걸 `switch`로 분기해서 사람이 이해할 수 있는 메시지로 바꿔주는 구조였다.

처음엔 `try/catch`가 "에러 나면 그냥 죽지 않게 막는 것" 정도로만 생각했는데, 여기서는 **에러의 종류(코드)까지 구분해서 각기 다른 메시지를 보여주는 용도**로 쓰인다는 걸 새로 배웠다.

> ✅ **핵심**: mongoose에서 `unique` 제약을 어기면 에러 코드 `11000`이 발생한다. `catch(e)`에서 `e.code`를 분기하면 "왜 실패했는지"에 따라 다른 메시지를 사용자에게 보여줄 수 있다.

---

## 🧹 비밀번호는 응답에서 왜 빼야 하나

작은 부분이지만 계속 눈에 밟혔던 코드다.

```js
let result = await Member.create({id, pw, name, phone});
let object = result.toObject();
delete object.pw; // pw는 결과값에서 제거하고 보여준다.
res.json({'join success':true, 'data': object});
```

`Member.create()`로 만든 결과는 mongoose 문서(document) 객체라서 그냥 JSON처럼 다루기 애매한 부분이 있고, `.toObject()`로 순수 자바스크립트 객체로 바꾼 다음에 `delete`로 `pw` 필드를 지워서 응답에 담고 있었다. DB에는 비밀번호가 저장되어야 하지만, 그 값을 그대로 클라이언트 응답에 실어 보내면 네트워크 상에 비밀번호가 그대로 노출되니까 굳이 한 번 더 걸러주는 거였다. `list`, `get/:id` 조회 라우터는 스키마의 `select: false` 덕분에 애초에 `pw`가 안 딸려오지만, `join`은 방금 만든 결과라 이 옵션이 안 먹혀서 수동으로 지워줘야 한다는 차이도 같이 이해됐다.

> ✅ **핵심**: 민감한 필드는 스키마의 `select: false`로 조회 시 기본 차단하거나, 생성 직후처럼 그 옵션이 안 통하는 시점에는 응답 직전에 명시적으로 지워서 내보내야 한다.

---

## 💡 정리하면

이번 실습은 DB가 실제로 붙는다는 것 자체도 새로웠지만, 그보다 예전에 겪었던 버그(return 누락)를 이번엔 스스로 발견했다는 게 제일 의미 있었다. 같은 실수를 계속 반복하는구나 싶으면서도, 한 번 이해하고 넘어간 개념은 다른 프로젝트에서도 똑같이 써먹을 수 있다는 걸 체감한 날이었다.

**오늘의 핵심 4가지**

- 🗂️ DB 연결(`db.js`) / 스키마 정의(`model.js`) / 요청 처리(`router.js`)로 역할을 나누면 관리가 쉬워진다
- 📐 mongoose 스키마의 `required`/`unique`/`enum`은 실제 저장 시점에 검증까지 관여한다
- 🔁 조건부로 응답을 나눠 보내는 코드는 매번 `return` 여부를 의심해야 한다 - 정상 케이스만 테스트하면 숨어있는 버그를 놓치기 쉽다
- 🚨 mongoose의 `unique` 제약 위반은 에러 코드 `11000`으로 잡아낼 수 있고, `try/catch`에서 에러 코드별로 다른 메시지를 줄 수 있다
