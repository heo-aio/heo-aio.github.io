+++
title = "SQL 집합 연산자와 계층형 질의 정리"
date = 2026-06-19T00:00:00+09:00
draft = false
tags = ["SQL", "Database"]
categories = ["SQL"]
+++

# 📚 SQL 집합 연산자와 계층형 질의 정리

SQL을 공부하다 보면 JOIN은 정말 자주 접하게 된다.

반면 집합 연산자(UNION, INTERSECT, EXCEPT)나 계층형 질의는 학습 과정에서 한 번 보고 넘어가는 경우가 많다. 나 역시 SQL 문제를 풀거나 프로젝트를 진행하면서 JOIN은 여러 번 사용해봤지만, 집합 연산자나 계층형 질의는 거의 사용해본 경험이 없었다.

그래서 처음에는 단순히 문법만 외우고 넘어갔는데, 관계형 데이터베이스의 기반이 되는 **관계형 대수(Relational Algebra)** 와 연결해서 이해해보니 각각의 연산이 왜 존재하는지 조금 더 명확하게 이해할 수 있었다.

특히 취업을 준비하면서 느끼는 것은 단순히 문법을 암기하는 것보다 **"왜 사용하는가?"** 를 이해하는 것이 면접에서도 훨씬 도움이 된다는 점이다.

이번 글에서는 SQL 집합 연산자와 계층형 질의에 대해 공부한 내용을 정리해보려고 한다.

---

# 📌 Standard SQL과 관계형 대수

관계형 데이터베이스는 수학의 집합 이론을 기반으로 만들어졌다.

Standard SQL 역시 관계형 대수(Relational Algebra)를 바탕으로 동작하며, 데이터를 조회하고 가공하기 위한 여러 연산을 제공한다.

관계형 대수는 크게 다음과 같은 연산들로 구성된다.

* 합집합(UNION)
* 교집합(INTERSECT)
* 차집합(EXCEPT)
* 선택(SELECT)
* 투영(PROJECT)
* 조인(JOIN)

SQL을 배우면서 JOIN만 자주 사용하다 보니 JOIN이 전부인 것처럼 느껴질 수 있지만, 실제로는 집합 연산 역시 관계형 데이터베이스의 중요한 개념 중 하나이다.

---

# 📌 집합 연산자(Set Operator)

집합 연산자는 두 개 이상의 SELECT 결과를 하나의 결과 집합으로 만들어준다.

사용하기 전에 반드시 알아야 할 조건이 있다.

## ✅ 사용 조건

두 SELECT 문의 결과는 다음 조건을 만족해야 한다.

* 컬럼 개수가 같아야 한다.
* 대응되는 컬럼의 데이터 타입이 같아야 한다.

예를 들어 다음과 같은 테이블이 있다고 가정해보자.

### ALPHA

| A | B |
| - | - |
| 1 | A |
| 2 | B |
| 3 | C |

### BETA

| A | B |
| - | - |
| 2 | B |
| 3 | C |
| 4 | D |

---

# 🔹 UNION

UNION은 두 결과 집합을 합친 뒤 중복 데이터를 제거한다.

```sql
SELECT *
FROM ALPHA

UNION

SELECT *
FROM BETA;
```

결과

| A | B |
| - | - |
| 1 | A |
| 2 | B |
| 3 | C |
| 4 | D |

집합으로 표현하면 다음과 같다.

```text
{1,2,3} ∪ {2,3,4}
=
{1,2,3,4}
```

### ✅ 특징

* 중복 제거
* 내부적으로 정렬 발생
* 합집합 역할 수행

처음에는 UNION과 UNION ALL의 차이를 단순히 "중복 제거 여부" 정도로만 외웠는데, 공부하다 보니 중복 제거를 위해 정렬 과정이 필요하다는 점도 알게 되었다.

---

# 🔹 UNION ALL

UNION ALL은 중복 제거 없이 결과를 그대로 연결한다.

```sql
SELECT *
FROM ALPHA

UNION ALL

SELECT *
FROM BETA;
```

결과

| A | B |
| - | - |
| 1 | A |
| 2 | B |
| 3 | C |
| 2 | B |
| 3 | C |
| 4 | D |

### ✅ 특징

* 중복 제거 없음
* 정렬 없음
* UNION보다 성능상 유리

실무에서는 중복 제거가 필요하지 않은 경우 UNION보다 UNION ALL을 더 많이 사용한다고 한다.

면접에서도 종종 나오는 질문 중 하나가

> UNION과 UNION ALL의 차이는 무엇인가?

인데 단순히 중복 제거 여부뿐만 아니라 정렬 과정 때문에 성능 차이가 발생한다는 점도 함께 설명할 수 있어야 할 것 같다.

---

# 🔹 INTERSECT

INTERSECT는 두 집합의 공통된 부분만 추출한다.

```sql
SELECT A, B
FROM ALPHA

INTERSECT

SELECT A, B
FROM BETA;
```

결과

| A | B |
| - | - |
| 2 | B |
| 3 | C |

집합으로 표현하면

```text
{1,2,3} ∩ {2,3,4}
=
{2,3}
```

### ✅ 특징

* 교집합 역할
* 중복 제거

### ⚠️ 참고

MySQL에서는 INTERSECT를 지원하지 않는다.

그래서 실제로는 JOIN이나 EXISTS를 활용해서 동일한 결과를 구현하는 경우가 많다.

---

# 🔹 EXCEPT

EXCEPT는 앞쪽 결과 집합에서 뒤쪽 결과 집합과 겹치는 데이터를 제거한다.

```sql
SELECT A, B
FROM ALPHA

EXCEPT

SELECT A, B
FROM BETA;
```

결과

| A | B |
| - | - |
| 1 | A |

집합으로 표현하면

```text
{1,2,3} - {2,3,4}
=
{1}
```

### ✅ 특징

* 차집합 역할
* 중복 제거

### ⚠️ 참고

MySQL은 EXCEPT 역시 지원하지 않는다.

실제로는 LEFT JOIN + NULL 조건이나 NOT EXISTS를 활용하여 구현한다.

---

# 📊 집합 연산자 한눈에 보기

```text
ALPHA = {1,2,3}
BETA  = {2,3,4}

UNION       = {1,2,3,4}
UNION ALL   = {1,2,3,2,3,4}
INTERSECT   = {2,3}
EXCEPT      = {1}
```

관계형 대수 관점에서 보니 왜 이름이 UNION, INTERSECT, EXCEPT인지 이해가 훨씬 쉬웠다.

---

# 🌳 계층형 질의(Hierarchical Query)

집합 연산자를 공부한 뒤 계층형 질의를 보니 조금 색다르게 느껴졌다.

지금까지 SQL은 단순히 행(Row)을 조회하는 언어라고 생각했는데, 조직도나 댓글 구조처럼 부모-자식 관계를 가진 데이터도 표현할 수 있기 때문이다.

예를 들어 직원 테이블이 다음과 같다고 가정해보자.

| 사원번호 | 관리자  |
| ---- | ---- |
| 100  | NULL |
| 101  | 100  |
| 102  | 100  |
| 103  | 101  |
| 104  | 101  |

이를 구조로 표현하면 다음과 같다.

```text
100
├─101
│  ├─103
│  └─104
└─102
```

---

# 🌳 Oracle 계층형 질의

Oracle은 CONNECT BY 문법을 제공한다.

```sql
SELECT LEVEL,
       사원번호,
       관리자
FROM 직원
START WITH 관리자 IS NULL
CONNECT BY PRIOR 사원번호 = 관리자;
```

## 🔹 START WITH

```sql
START WITH 관리자 IS NULL
```

가장 상위 노드(Root)를 찾는다.

```text
100
```

## 🔹 CONNECT BY

```sql
CONNECT BY PRIOR 사원번호 = 관리자
```

부모와 자식을 연결하면서 아래 방향으로 탐색한다.

```text
100
├─101
│ ├─103
│ └─104
└─102
```

---

# 📌 Oracle 주요 키워드

| 키워드                 | 설명       |
| ------------------- | -------- |
| LEVEL               | 현재 깊이    |
| CONNECT_BY_ROOT     | 루트 노드    |
| CONNECT_BY_ISLEAF   | 리프 노드 여부 |
| SYS_CONNECT_BY_PATH | 전체 경로    |

예시

```sql
SELECT
    사원번호,
    SYS_CONNECT_BY_PATH(사원번호, ' > ')
FROM 직원
START WITH 관리자 IS NULL
CONNECT BY PRIOR 사원번호 = 관리자;
```

결과

```text
100
100 > 101
100 > 101 > 103
100 > 101 > 104
100 > 102
```

---

# 🌳 MySQL의 계층형 질의

MySQL은 Oracle의 CONNECT BY를 지원하지 않는다.

대신 재귀 CTE(Common Table Expression)를 사용한다.

```sql
WITH RECURSIVE CTE AS (

    SELECT
        member_id,
        manager_id,
        0 AS lvl
    FROM MEMBER
    WHERE manager_id IS NULL

    UNION ALL

    SELECT
        a.member_id,
        a.manager_id,
        b.lvl + 1
    FROM MEMBER a
    JOIN CTE b
      ON a.manager_id = b.member_id

)

SELECT *
FROM CTE;
```

처음 봤을 때는 문법이 상당히 복잡해 보였는데, 결국 루트 노드를 찾고 계속 자기 자신을 호출하면서 하위 노드를 탐색하는 구조라는 점을 이해하니 조금 수월하게 읽혔다.

---

# 📝 공부하면서 느낀 점

SQL을 공부할 때는 SELECT, GROUP BY, JOIN에 집중하게 되는데, 집합 연산자와 계층형 질의도 데이터베이스의 중요한 개념이라는 것을 다시 느꼈다.

특히 취업 준비를 하면서 CS 과목을 다시 공부하다 보니 단순히 문법을 외우는 것보다 개념의 배경을 이해하는 것이 훨씬 기억에 오래 남는 것 같다.

아직 실무 프로젝트에서 직접 사용해본 경험은 없지만, 조직도 데이터나 댓글 구조 같은 계층형 데이터를 다룰 때 왜 이런 문법이 필요한지 이해할 수 있었다.

앞으로 SQL을 공부할 때도 단순히 쿼리를 작성하는 데서 끝나는 것이 아니라, 데이터베이스가 어떤 원리로 동작하는지 함께 이해하는 방향으로 학습해보려고 한다.

---

# 🚀 주기적으로 꼭 다시보자!!!

* UNION → 합집합 (중복 제거)
* UNION ALL → 단순 연결 (중복 제거 없음)
* INTERSECT → 교집합
* EXCEPT → 차집합
* Oracle → CONNECT BY 사용
* MySQL → WITH RECURSIVE 사용
* UNION ALL이 UNION보다 일반적으로 빠름
* 계층형 질의는 조직도, 댓글 구조, 카테고리 구조 등에 활용 가능
