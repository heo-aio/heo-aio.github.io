+++
title = "SQL 서브쿼리(SubQuery) 정리 공부일지"
date = 2026-06-20
draft = false
tags = ["SQL", "Database", "CS", "면접준비"]
categories = ["SQL"]
+++

# SQL 서브쿼리(SubQuery) 정리

SQL을 공부하면서 JOIN 다음으로 자주 보게 되는 것이 바로 서브쿼리(SubQuery)였다.

처음에는 괄호 안에 또 SELECT 문이 들어가는 형태라 복잡해 보였지만, 실제로는 "쿼리 안에 또 다른 쿼리"라고 생각하니 이해하기 쉬웠다.

이번 글에서는 서브쿼리의 종류와 특징을 정리해보려고 한다.

---

## 📌 서브쿼리란?

서브쿼리(SubQuery)는 하나의 SQL 문 안에 포함된 또 다른 SQL 문이다.

```sql
SELECT *
FROM EMPLOYEE
WHERE DEPARTMENT_ID = (
    SELECT DEPARTMENT_ID
    FROM EMPLOYEE
    WHERE NAME = 'ELICE'
);
```

위 예시에서 괄호 안의 SELECT 문이 서브쿼리이다.

---

# 1️⃣ 동작 방식에 따른 서브쿼리

## 🔹 연관 서브쿼리 (Correlated SubQuery)

메인 쿼리의 컬럼이 서브쿼리에 포함되는 형태이다.

```sql
SELECT ID, DEPARTMENT_ID, NAME, SALARY
FROM EMPLOYEE A
WHERE SALARY > (
    SELECT AVG(SALARY)
    FROM EMPLOYEE B
    WHERE B.DEPARTMENT_ID = A.DEPARTMENT_ID
);
```

### 💡 결과

본인이 속한 부서의 평균 급여보다 높은 급여를 받는 직원 출력

### 👀 핵심

```text
A.DEPARTMENT_ID
```

가 서브쿼리 내부에서 사용된다.

즉,

메인 쿼리와 서브쿼리가 서로 연결되어 있다.

---

## 🔹 비연관 서브쿼리 (Non-Correlated SubQuery)

메인 쿼리와 독립적으로 실행되는 서브쿼리이다.

```sql
SELECT AVG(SALARY)
FROM EMPLOYEE
WHERE DEPARTMENT_ID = (
    SELECT DEPARTMENT_ID
    FROM EMPLOYEE
    WHERE NAME = 'ELICE'
);
```

### 💡 결과

ELICE가 속한 부서의 평균 급여 출력

### 👀 핵심

메인 쿼리 컬럼이 서브쿼리 안에 등장하지 않는다.

---

# 2️⃣ 반환되는 데이터 형태에 따른 서브쿼리

## 🔹 단일 행 서브쿼리

서브쿼리가 한 개의 행만 반환하는 경우이다.

```sql
SELECT ID, NAME, SALARY
FROM EMPLOYEE
WHERE DEPARTMENT_ID = (
    SELECT DEPARTMENT_ID
    FROM EMPLOYEE
    WHERE NAME = 'ELICE'
);
```

### ✅ 사용 연산자

* =
* >
* <
* > =
* <=

### 💡 결과

ELICE와 같은 부서에 속한 직원 조회

---

## 🔹 다중 행 서브쿼리

서브쿼리가 여러 개의 행을 반환하는 경우이다.

### 1. IN

```sql
SELECT NAME
FROM EMPLOYEE
WHERE DEPARTMENT_ID IN (
    SELECT ID
    FROM DEPARTMENT
    WHERE NAME = '품질'
       OR NAME = '영업'
);
```

💡 품질 또는 영업 부서 직원 조회

---

### 2. EXISTS

```sql
SELECT NAME
FROM EMPLOYEE A
WHERE EXISTS (
    SELECT ID
    FROM EMPLOYEE B
    WHERE B.SALARY >= 10000
      AND A.DEPARTMENT_ID = B.DEPARTMENT_ID
);
```

💡 고액 연봉자가 존재하는 부서의 모든 직원 조회

---

### 3. ALL

```sql
SELECT NAME
FROM EMPLOYEE
WHERE SALARY >= ALL (
    SELECT SALARY
    FROM EMPLOYEE
    WHERE DEPARTMENT_ID = 1
);
```

💡 개발팀 모든 직원보다 급여가 높은 직원 조회

---

### 4. ANY

```sql
SELECT NAME
FROM EMPLOYEE
WHERE SALARY >= ANY (
    SELECT SALARY
    FROM EMPLOYEE
    WHERE DEPARTMENT_ID = 1
);
```

💡 개발팀 직원 중 한 명 이상보다 급여가 높은 직원 조회

---

🎯 면접 포인트

| 연산자    | 의미        |
| ------ | --------- |
| IN     | 여러 값 중 하나 |
| EXISTS | 존재 여부     |
| ALL    | 모두 만족     |
| ANY    | 하나 이상 만족  |

---

## 🔹 다중 컬럼 서브쿼리

서브쿼리가 여러 개의 컬럼을 반환하는 경우이다.

```sql
SELECT NAME, SALARY
FROM EMPLOYEE
WHERE (DEPARTMENT_ID, SALARY) IN (
    SELECT DEPARTMENT_ID, MAX(SALARY)
    FROM EMPLOYEE
    GROUP BY DEPARTMENT_ID
);
```

### 💡 결과

각 부서 최고 연봉자 조회

---

# 3️⃣ 스칼라 서브쿼리

스칼라 서브쿼리는

* 하나의 컬럼
* 하나의 행

만 반환하는 서브쿼리이다.

SELECT절에서도 사용할 수 있다.

```sql
SELECT NAME,
(
    SELECT COUNT(*)
    FROM EMPLOYEE E
    WHERE E.DEPARTMENT_ID = D.ID
)
FROM DEPARTMENT D;
```

### 💡 결과

부서명과 부서 인원 수 출력

---

## 🔹 DUAL 예시

```sql
SELECT
(
    SELECT COUNT(*)
    FROM EMPLOYEE
    WHERE DEPARTMENT_ID = 1
)
/
(
    SELECT COUNT(*)
    FROM EMPLOYEE
)
AS DEVELOPER_RATIO
FROM DUAL;
```

### 💡 결과

전체 직원 중 개발팀 비율 출력

### 📝 참고

DUAL은 Oracle에서 제공하는 가상 테이블이다.

실제 테이블 없이도 SELECT 문을 실행할 수 있다.

---

# 📝 공부하면서 느낀 점

처음에는 서브쿼리가 JOIN보다 어렵게 느껴졌다.

특히 ALL, ANY, EXISTS의 차이가 헷갈렸다.

하지만 직접 예제를 보면서 정리해보니 결국 핵심은

> 서브쿼리가 어떤 값을 반환하는가?

를 이해하는 것이었다.

앞으로 SQL 문제를 풀 때도 JOIN과 서브쿼리를 상황에 맞게 사용하는 연습이 필요할 것 같다.

---

## 🚀 한 줄 정리

📌 서브쿼리는 SQL 안에 포함된 또 다른 SQL이며, 반환 형태와 동작 방식을 이해하는 것이 핵심이다.
