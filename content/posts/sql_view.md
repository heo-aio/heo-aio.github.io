+++
title = "SQL VIEW(뷰) 정리"
date = 2026-06-20
draft = false
tags = ["SQL", "Database", "CS", "면접준비"]
categories = ["CS"]
+++

# SQL VIEW(뷰) 정리

데이터베이스를 공부하다 보면 VIEW(뷰)라는 개념을 자주 만나게 된다.

처음에는 단순히 "가상 테이블" 정도로만 알고 있었는데, 공부해보니 복잡한 쿼리를 단순화하고 보안성을 높이는 데 활용되는 중요한 기능이었다.

이번 글에서는 VIEW의 개념과 특징, 그리고 장단점을 정리해보려고 한다.

---

## 📌 VIEW란?

VIEW는 하나 이상의 테이블을 기반으로 생성되는 가상의 테이블이다.

실제 데이터를 저장하는 것이 아니라, SQL 문만 저장해두고 필요할 때마다 원본 테이블을 조회하여 결과를 보여준다.

즉,

```text
테이블 → 실제 데이터 저장

뷰(View) → 조회 결과만 보여주는 가상 테이블
```

이라고 이해하면 편하다.

---

## 👀 VIEW 동작 방식

예를 들어 직원 테이블과 부서 테이블이 있다고 가정하자.

```text
EMPLOYEE
+
DEPARTMENT
```

두 테이블을 JOIN하여 자주 조회해야 한다면,

매번 JOIN 문을 작성하는 대신 VIEW를 만들어 사용할 수 있다.

```text
EMPLOYEE
      \
       → VIEW
      /
DEPARTMENT
```

사용자는 VIEW만 조회하면 된다.

---

# 1️⃣ VIEW의 장점

## 🔹 독립성

원본 테이블 구조가 일부 변경되더라도,

VIEW를 사용하는 응용 프로그램은 영향을 덜 받을 수 있다.

### 💡 예시

```text
프로그램
   ↓
 VIEW
   ↓
테이블
```

응용 프로그램이 직접 테이블을 참조하지 않고 VIEW를 참조하기 때문이다.

---

## 🔹 편리성

복잡한 SQL을 매번 작성할 필요가 없다.

자주 사용하는 JOIN 문이나 집계 쿼리를 VIEW로 만들어두면 편리하게 사용할 수 있다.

### 예시

원래는

```sql
SELECT E.NAME,
       D.NAME
FROM EMPLOYEE E
LEFT JOIN DEPARTMENT D
ON E.DEPARTMENT_ID = D.ID;
```

처럼 작성해야 하지만,

VIEW를 만들어 두면

```sql
SELECT *
FROM EMPLOYEE_FULL;
```

만으로 동일한 결과를 조회할 수 있다.

---

## 🔹 보안성

사용자에게 필요한 데이터만 제공할 수 있다.

예를 들어

* 이름
* 부서명

만 보여주고

* 주민등록번호
* 급여

는 숨길 수 있다.

### 💡 활용

```text
관리자 → 전체 정보 조회

일반 사용자 → 제한된 정보 조회
```

---

# 2️⃣ VIEW의 특징

## 🔹 VIEW로 VIEW 생성 가능

생성된 VIEW를 기반으로 또 다른 VIEW를 만들 수 있다.

```text
원본 테이블
     ↓
 VIEW A
     ↓
 VIEW B
```

---

## 🔹 VIEW 정의 변경 제한

VIEW의 정의를 직접 수정하는 것이 제한되는 경우가 있다.

따라서 기존 VIEW를 삭제하고 다시 생성하는 방식으로 작업하는 경우가 많다.

---

## 🔹 갱신 시 기본키 필요

VIEW를 통해 데이터를 수정하려면

원본 테이블의 기본키(PK)가 포함되어 있어야 한다.

### ⚠️ 이유

어떤 데이터를 수정해야 하는지 DBMS가 식별해야 하기 때문이다.

---

## 🔹 원본 객체 삭제 시 영향 발생

원본 테이블이나 VIEW가 삭제되면

이를 기반으로 만들어진 VIEW도 정상적으로 사용할 수 없게 된다.

```text
EMPLOYEE 삭제
      ↓
EMPLOYEE_VIEW 사용 불가
```

---

# 3️⃣ VIEW 생성 예시

실제로는 CREATE VIEW 문을 사용한다.

```sql
CREATE VIEW EMPLOYEE_FULL AS
(
    SELECT
        E.ID AS EMPLOYEE_ID,
        E.NAME AS EMPLOYEE_NAME,
        E.SALARY,
        E.DEPARTMENT_ID,
        D.NAME AS DEPARTMENT_NAME
    FROM EMPLOYEE E
    LEFT OUTER JOIN DEPARTMENT D
    ON E.DEPARTMENT_ID = D.ID
);
```

---

## 💡 생성 후 조회

VIEW가 생성되면 일반 테이블처럼 사용할 수 있다.

```sql
SELECT *
FROM EMPLOYEE_FULL;
```

복잡한 JOIN 문을 다시 작성하지 않아도 된다.

---

# 🎯 면접 포인트

### Q. VIEW는 실제 데이터를 저장하는가?

❌ 아니다.

VIEW는 SQL 정의만 저장하고,

실제 데이터는 원본 테이블에 존재한다.

---

### Q. VIEW를 사용하는 이유는?

✅ 독립성

✅ 편리성

✅ 보안성

---

### Q. VIEW를 통해 UPDATE가 가능한가?

가능하지만 조건이 있다.

원본 테이블의 기본키가 포함되어 있어야 하며,

복잡한 JOIN VIEW는 수정이 제한될 수 있다.

---

# 📝 공부하면서 느낀 점

처음에는 VIEW를 단순히 "가상 테이블"이라고만 외웠다.

그런데 실제로는 복잡한 JOIN 문을 숨겨주고, 사용자별로 접근 가능한 데이터를 제한할 수 있다는 점에서 생각보다 활용도가 높다는 것을 알게 되었다.

특히 실무에서는 단순 조회뿐 아니라 보안과 유지보수 측면에서도 VIEW를 많이 활용한다고 하니 개념만 외우기보다 왜 사용하는지 이해하는 것이 중요한 것 같다.

---

## 🚀 한 줄 정리

📌 VIEW는 실제 데이터를 저장하지 않는 가상 테이블이며, 복잡한 쿼리 단순화와 보안성 향상을 위해 사용된다.
