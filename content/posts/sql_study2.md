+++
title = "SQL JOIN(심화) 공부일지"
date = 2026-06-19
draft = false
tags = ["SQL", "Database", "면접준비"]
categories = ["SQL"]
+++

## 📚 SQL JOIN 정리

데이터베이스를 공부하다 보면 가장 많이 만나게 되는 개념 중 하나가 JOIN이다.

처음에는 단순히 테이블을 연결하는 기능이라고 생각했는데, 실제로 SQL 문제를 풀어보니 JOIN을 제대로 이해하지 못하면 원하는 데이터를 가져오기 어려웠다.

이번 글에서는 JOIN의 종류와 특징을 정리해보려고 한다.

---

## 📌 JOIN이란?

JOIN은 두 개 이상의 테이블을 연결해서 원하는 데이터를 조회하는 명령어이다.

예를 들어,

* USER 테이블에는 회원 정보
* CLASS 테이블에는 수강 정보

가 있다고 할 때, 각 테이블을 따로 조회하는 것이 아니라 연결해서 한 번에 볼 수 있다.

```text
USER                    CLASS
+----+-------+          +----+--------+
| ID | NAME  |          | ID | CLASS  |
+----+-------+          +----+--------+
| 1  | 철수  |          | 1  | SQL    |
| 2  | 영희  |          | 2  | Python |
+----+-------+          +----+--------+
```

JOIN을 사용하면 필요한 데이터를 하나의 결과로 조회할 수 있다.

---

## 🔹 EQUI JOIN

가장 일반적인 JOIN 방식이다.

두 테이블의 값이 정확히 일치할 때 연결한다.

```sql
SELECT *
FROM USER u
JOIN CLASS c
ON u.CLASS_ID = c.ID;
```

✅ 특징

* `=` 연산자를 사용
* 대부분 기본키(PK)와 외래키(FK)를 기준으로 연결
* 실무에서 가장 많이 사용

```text
USER.CLASS_ID = CLASS.ID
```

처럼 서로 같은 값일 때만 연결된다.

---

## 🔹 Non EQUI JOIN

값이 정확히 같지 않아도 범위를 기준으로 연결하는 JOIN이다.

```sql
SELECT *
FROM EMPLOYEE e
JOIN GRADE g
ON e.SALARY BETWEEN g.MIN_SALARY AND g.MAX_SALARY;
```

✅ 사용 연산자

* >
* <
* > =
* <=
* BETWEEN

처음 봤을 때는 잘 사용되지 않을 것 같았는데,

급여 구간별 등급 분류 같은 경우에 많이 사용된다.

---

## 🔹 INNER JOIN

가장 기본적인 JOIN이다.

두 테이블에 공통으로 존재하는 데이터만 조회한다.

```sql
SELECT *
FROM USER a
INNER JOIN CLASS b
ON a.CLASS_ID = b.ID;
```

`INNER`는 생략 가능하다.

```sql
SELECT *
FROM USER a
JOIN CLASS b
ON a.CLASS_ID = b.ID;
```

결과는 동일하다.

### 👀 시각적으로 이해하기

```text
USER      CLASS

   (A)
  ◯◯◯◯◯
    ◯◯◯
  ◯◯◯◯◯
   (B)
```

💡 결과

```text
교집합만 조회
```

즉, 양쪽 모두 존재하는 데이터만 가져온다.

---

## 🔹 USING

두 테이블에 동일한 이름의 컬럼이 있을 때 사용할 수 있다.

```sql
SELECT *
FROM USER
JOIN CLASS
USING (CLASS_ID);
```

장점은 코드가 간결하다는 점이다.

다만 실무에서는 ON 절을 더 많이 보는 것 같다.

⚠️ 주의

* SQL Server 지원 X
* 별칭 사용 제한

---

## 🔹 NATURAL JOIN

동일한 이름을 가진 모든 컬럼을 자동으로 연결한다.

```sql
SELECT *
FROM USER
NATURAL JOIN CLASS;
```

처음 보면 편해 보이지만,

어떤 컬럼이 자동으로 연결되는지 명확하지 않아서 유지보수 측면에서는 위험할 수 있다.

그래서 실무에서는 ON 절을 사용하는 경우가 더 많다고 한다.

---

## 🔹 CROSS JOIN

JOIN 조건 없이 가능한 모든 조합을 생성한다.

```sql
SELECT *
FROM PERSON
CROSS JOIN PUBLIC_TRANSPORT;
```

예를 들어

```text
사람 3명
교통수단 4개
```

라면 결과는

```text
3 × 4 = 12개
```

가 된다.

### 👀 시각화

```text
PERSON

철수
영희
민수

TRANSPORT

버스
지하철

결과

철수-버스
철수-지하철

영희-버스
영희-지하철

민수-버스
민수-지하철
```

모든 경우의 수가 생성된다.

---

## 🔹 OUTER JOIN

한쪽 테이블에만 존재하는 데이터도 포함해서 조회한다.

INNER JOIN이 교집합이라면,

OUTER JOIN은 교집합 + 한쪽 데이터까지 포함한다고 이해하면 편하다.

---

## 🔹 LEFT JOIN

왼쪽 테이블을 기준으로 조회한다.

```sql
SELECT *
FROM USER
LEFT JOIN CLASS
ON USER.CLASS_ID = CLASS.CLASS_ID;
```

### 👀 시각화

```text
USER 기준

USER ○○○○○
     ○○○
CLASS ○○○○
```

💡 결과

```text
USER의 모든 데이터 출력
매칭 안되면 NULL
```

---

## 🔹 RIGHT JOIN

오른쪽 테이블을 기준으로 조회한다.

```sql
SELECT *
FROM USER
RIGHT JOIN CLASS
ON USER.CLASS_ID = CLASS.CLASS_ID;
```

LEFT JOIN과 반대 개념이다.

---

## 🔹 FULL OUTER JOIN

양쪽 테이블의 모든 데이터를 가져온다.

```text
LEFT JOIN 결과
+
RIGHT JOIN 결과
```

라고 생각하면 이해하기 쉽다.

```sql
SELECT *
FROM USER
FULL OUTER JOIN CLASS
ON USER.CLASS_ID = CLASS.CLASS_ID;
```

### ⚠️ MySQL에서는?

MySQL은 FULL OUTER JOIN을 지원하지 않는다.

그래서 보통 UNION으로 구현한다.

```sql
SELECT *
FROM CLASS
LEFT JOIN USER
ON USER.CLASS_ID = CLASS.CLASS_ID

UNION

SELECT *
FROM CLASS
RIGHT JOIN USER
ON USER.CLASS_ID = CLASS.CLASS_ID;
```

이 부분은 면접에서도 가끔 물어보는 내용이라 기억해둘 필요가 있다.

🎯 면접 포인트

* MySQL은 FULL OUTER JOIN을 지원하지 않는다.
* UNION을 사용해 동일한 결과를 만들 수 있다.

---

## 🔹 SELF JOIN

같은 테이블을 자기 자신과 JOIN하는 방식이다.

조직도 구조를 조회할 때 자주 사용된다.

예를 들어,

```text
사원번호
관리자번호
```

가 같은 테이블 안에 존재한다고 가정하자.

```sql
SELECT
    A.사원번호,
    A.관리자,
    B.관리자 AS 차상위관리자
FROM 직원 A,
     직원 B
WHERE A.관리자 = B.사원번호;
```

### 🤔 왜 별칭이 필요할까?

SELF JOIN은 같은 테이블을 두 번 사용하기 때문에 SQL이 구분할 수 없다.

그래서 반드시 별칭(A, B)을 사용해야 한다.

```text
직원 테이블

사원 → 관리자

A → 현재 직원
B → 관리자 정보
```

이렇게 서로 다른 테이블처럼 취급한다.

---

## 📝 공부하면서 느낀 점

JOIN은 처음에는 종류가 많아서 복잡하게 느껴졌다.

특히 LEFT JOIN, RIGHT JOIN 에서 그냥 LEFT 나 RIGHT 둘 중에 하나만 쓰면 되는거 아닌가? 왜 굳이 나누어 놓은 걸까? 라는 생각도 들었다.

하지만 직접 그림을 그려보면서 정리하니 결국 핵심은 하나였다.

> 어떤 데이터를 기준으로 가져올 것인가?

실제로 SQL 문제를 풀 때도 JOIN을 이해하고 있느냐에 따라 난이도가 크게 달라지는 것 같다.

당분간은 단순히 문법을 외우기보다 직접 테이블을 만들어보고 결과를 확인하면서 익숙해지는 연습을 해야겠다.

---

## 🚀 한 줄 정리

📌 JOIN은 여러 테이블의 데이터를 연결하는 기능이며, SQL 실무와 코딩테스트에서 필요하니까 자주 자주 복습하자!!!
