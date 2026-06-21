+++
title = "SQL 윈도우 함수와 그룹 함수 정리"
date = 2026-06-21T00:00:00+09:00
draft = false
tags = ["SQL", "Database", "면접준비"]
categories = ["SQL"]
+++

# 📚 SQL 윈도우 함수(Window Function)와 그룹 함수(Group Function) 정리

SQL을 공부하면서 `GROUP BY`는 자주 사용했지만, 윈도우 함수(Window Function)는 처음 봤을 때 상당히 낯설었다.

특히 OVER 구문이 등장하면서 기존에 알고 있던 집계 함수와는 무엇이 다른지 이해하기 어려웠다.

하지만 여러 예제를 직접 확인해보니 윈도우 함수는 데이터를 그룹화해서 행을 줄이는 것이 아니라, **원본 데이터를 유지하면서 추가적인 통계 정보를 제공하는 기능**이라는 점을 알게 되었다.

이번 글에서는 윈도우 함수와 그룹 함수에 대해 정리해보려고 한다.

---

# 📌 윈도우 함수(Window Function)

윈도우 함수(Window Function)는 순위, 집계 등 행과 행 사이의 관계를 정의하는 함수이다.

윈도우 함수는 반드시 `OVER()` 구문과 함께 사용한다.

```sql
SELECT WINDOW_FUNCTION(ARGUMENTS)
OVER (
    [PARTITION BY 컬럼]
    [ORDER BY 절]
    [WINDOWING 절]
)
FROM 테이블명;
```

## 🔹 구성 요소

| 구성 요소        | 설명               |
| ------------ | ---------------- |
| ARGUMENTS    | 함수에 필요한 인수       |
| PARTITION BY | 데이터를 그룹으로 나누는 기준 |
| ORDER BY     | 그룹 내 정렬 기준       |
| WINDOWING    | 계산 범위 지정         |

---

## 🤔 GROUP BY와 무엇이 다를까?

처음에는 집계 함수와 비슷해 보여서 헷갈렸다.

예를 들어 부서 평균 급여를 구한다고 가정해보자.

### GROUP BY

```sql
SELECT DEPARTMENT_ID,
       AVG(SALARY)
FROM EMPLOYEE
GROUP BY DEPARTMENT_ID;
```

결과

| 부서  | 평균 급여 |
| --- | ----- |
| 개발팀 | 5000  |
| 영업팀 | 4500  |

행 수가 줄어든다.

---

### WINDOW FUNCTION

```sql
SELECT ID,
       NAME,
       SALARY,
       AVG(SALARY)
       OVER(PARTITION BY DEPARTMENT_ID)
FROM EMPLOYEE;
```

결과

| 사원 | 급여   | 부서 평균 |
| -- | ---- | ----- |
| 철수 | 6000 | 5000  |
| 영희 | 4000 | 5000  |
| 민수 | 4500 | 4500  |

원본 데이터는 유지되고 추가 정보만 붙는다.

---

# 🏆 순위 함수

순위를 구할 때 사용하는 함수들이다.

```sql
SELECT ID,
       NAME,
       SALARY,
       RANK() OVER(ORDER BY SALARY DESC) RANK,
       DENSE_RANK() OVER(ORDER BY SALARY DESC) DENSE_RANK,
       ROW_NUMBER() OVER(ORDER BY SALARY DESC) ROW_NUMBER
FROM EMPLOYEE;
```

---

## 🔹 RANK

동일한 값은 동일한 순위를 부여한다.

| 급여    | 순위 |
| ----- | -- |
| 10000 | 1  |
| 9000  | 2  |
| 9000  | 2  |
| 8000  | 4  |

중간 순위가 비어있다.

---

## 🔹 DENSE_RANK

동일한 값은 같은 순위를 부여하지만 건너뛰지 않는다.

| 급여    | 순위 |
| ----- | -- |
| 10000 | 1  |
| 9000  | 2  |
| 9000  | 2  |
| 8000  | 3  |

---

## 🔹 ROW_NUMBER

동일한 값이어도 고유 번호를 부여한다.

| 급여    | 순위 |
| ----- | -- |
| 10000 | 1  |
| 9000  | 2  |
| 9000  | 3  |
| 8000  | 4  |

---

## 🎯 RANK / DENSE_RANK / ROW_NUMBER 차이 한 눈에 보기

| 함수         | 특징                 |
| ---------- | ------------------ |
| RANK       | 공동 순위 허용, 순위 건너뜀   |
| DENSE_RANK | 공동 순위 허용, 순위 안 건너뜀 |
| ROW_NUMBER | 무조건 고유 순위          |

---

# 📊 일반 집계 함수

윈도우 함수에서는 일반 집계 함수도 사용할 수 있다.

```sql
SELECT ID,
       NAME,
       SALARY,
       AVG(SALARY)
       OVER(PARTITION BY DEPARTMENT_ID)
       AS DEPARTMENT_AVG
FROM EMPLOYEE;
```

결과적으로 각 직원 행마다 부서 평균 급여를 함께 출력할 수 있다.

---

# 🔄 그룹 내 행 순서 함수

특정 그룹 내에서 앞 또는 뒤의 값을 참조할 수 있다.

---

## 🔹 FIRST_VALUE

가장 먼저 나온 값을 반환한다.

---

## 🔹 LAST_VALUE

가장 마지막 값을 반환한다.

```sql
SELECT ID,
       DEPARTMENT_ID,
       NAME,
       SALARY,

       FIRST_VALUE(SALARY)
       OVER (
            PARTITION BY DEPARTMENT_ID
            ORDER BY SALARY
            ROWS BETWEEN UNBOUNDED PRECEDING
            AND UNBOUNDED FOLLOWING
       ) AS DEPARTMENT_MIN_SALARY,

       LAST_VALUE(SALARY)
       OVER (
            PARTITION BY DEPARTMENT_ID
            ORDER BY SALARY
            ROWS BETWEEN UNBOUNDED PRECEDING
            AND UNBOUNDED FOLLOWING
       ) AS DEPARTMENT_MAX_SALARY

FROM EMPLOYEE;
```

💡 각 부서의 최저 급여와 최고 급여를 함께 조회할 수 있다.

---

## 🔹 LAG

이전 행을 조회한다.

```sql
SELECT ID,
       NAME,
       SALARY,
       LAG(NAME,1)
       OVER(ORDER BY ID)
       AS PREV_EMPLOYEE_NAME
FROM EMPLOYEE;
```

---

## 🔹 LEAD

다음 행을 조회한다.

```sql
SELECT ID,
       NAME,
       SALARY,
       LEAD(NAME,1)
       OVER(ORDER BY ID)
       AS AFTER_EMPLOYEE_NAME
FROM EMPLOYEE;
```

---

### 👀 예시

| ID | 이름 | 이전 직원 | 다음 직원 |
| -- | -- | ----- | ----- |
| 1  | 철수 | NULL  | 영희    |
| 2  | 영희 | 철수    | 민수    |
| 3  | 민수 | 영희    | NULL  |

실무에서는 전월 매출, 전일 방문자 수 등 이전 데이터와 비교할 때 자주 사용된다고 한다.

---

# 📈 그룹 내 비율 함수

---

## 🔹 RATIO_TO_REPORT

전체 합계에서 현재 행이 차지하는 비율을 계산한다.

```sql
SELECT ID,
       NAME,
       SALARY,
       SUM(SALARY) OVER() TOTAL_SALARY,
       RATIO_TO_REPORT(SALARY) OVER()
FROM EMPLOYEE;
```

예를 들어 전체 급여 합계가 10000이고 현재 직원 급여가 2000이라면

```text
2000 / 10000 = 0.2
```

즉 20%가 된다.

---

## 🔹 PERCENT_RANK
```sql
SELECT
    name,
    score,
    PERCENT_RANK() OVER (ORDER BY score) AS percent_rank
FROM student;
```
순위를 백분율로 표현한다.

### 예시

학생들의 점수를 기준으로 `PERCENT_RANK()`를 계산해보자.

| name | score |
|------|------:|
| 김철수 | 70 |
| 이영희 | 80 |
| 박민수 | 90 |
| 최지훈 | 100 |

```sql
SELECT
    name,
    score,
    PERCENT_RANK() OVER (ORDER BY score) AS percent_rank
FROM student;
```

### 실행 결과

| name | score | percent_rank |
|------|------:|-------------:|
| 김철수 | 70 | 0.000 |
| 이영희 | 80 | 0.333 |
| 박민수 | 90 | 0.667 |
| 최지훈 | 100 | 1.000 |

최하위 순위는 0, 최상위 순위는 1로 표현되며, 현재 행의 순위가 전체 데이터에서 어느 정도 위치에 있는지 백분율로 확인할 수 있다.

---

## 🔹 CUME_DIST

누적 분포 비율을 구한다.

```sql
SELECT ID,
       NAME,
       SALARY,
       PERCENT_RANK()
       OVER(ORDER BY SALARY DESC),

       ROUND(
           CUME_DIST()
           OVER(ORDER BY SALARY DESC),
           4
       )
FROM EMPLOYEE;
```

---

## 🔹 NTILE

데이터를 N개의 그룹으로 나눈다.

```sql
SELECT ID,
       NAME,
       SALARY,
       NTILE(3)
       OVER(ORDER BY SALARY DESC)
FROM EMPLOYEE;
```

예를 들어 30명을 3개 그룹으로 나누면

| 그룹 | 인원  |
| -- | --- |
| 1  | 10명 |
| 2  | 10명 |
| 3  | 10명 |

처럼 나누어진다.

---

# 📌 그룹 함수(Group Function)

그룹 함수는 데이터를 그룹화하여 통계 정보를 생성하는 함수이다.

대표적으로

* SUM
* AVG
* MAX
* MIN
* COUNT

등이 있다.

이번에는 SQLD와 면접에서 자주 나오는 고급 그룹 함수들을 정리해보려고 한다.

---

# 🔹 ROLLUP

부분 합계를 생성한다.

```sql
GROUP BY ROLLUP(D.NAME, J.NAME)
```

예를 들어

| 부서 | 직무   | 평균 급여 |
| -- | ---- | ----- |
| 개발 | 백엔드  | 6000  |
| 개발 | 프론트  | 5000  |
| 개발 | NULL | 5500  |

마지막 행이 개발 부서 전체 평균이다.

---

### 💡 생성 순서

```text
(부서, 직무)
→ (부서)
→ 전체
```

---

# 🔹 CUBE

가능한 모든 조합의 집계를 생성한다.

```sql
GROUP BY CUBE(D.NAME, J.NAME)
```

---

### 👀 생성 결과

```text
(부서, 직무)
(부서)
(직무)
(전체)
```

ROLLUP보다 더 많은 집계 결과를 생성한다.

---

# 🔹 GROUPING SETS

원하는 그룹만 선택적으로 생성한다.

```sql
GROUP BY GROUPING SETS(
    D.NAME,
    J.NAME
)
```

---

### 💡 결과

```text
부서 기준 집계
+
직무 기준 집계
```

즉,

```sql
GROUP BY D.NAME

UNION ALL

GROUP BY J.NAME
```

과 동일한 결과를 만든다.

---

# 📊 ROLLUP vs CUBE vs GROUPING SETS

| 함수            | 생성 범위   |
| ------------- | ------- |
| ROLLUP        | 계층적 부분합 |
| CUBE          | 모든 조합   |
| GROUPING SETS | 지정한 그룹만 |

---

# 📝 공부하면서 느낀 점

처음에는 윈도우 함수가 GROUP BY와 크게 다르지 않은 기능이라고 생각했다.

하지만 실제로는 데이터를 그룹화해서 행을 줄이는 것이 아니라, 원본 데이터를 유지하면서 통계 정보를 제공한다는 점에서 완전히 다른 개념이었다.

특히 RANK, ROW_NUMBER, LAG, LEAD 같은 함수들은 데이터 분석이나 대시보드 구축 과정에서 상당히 유용하게 사용될 것 같았다.

또한 ROLLUP, CUBE, GROUPING SETS는 처음에는 복잡하게 느껴졌지만 결국 "어떤 기준으로 집계를 만들 것인가?"의 차이로 이해하니 훨씬 수월하게 정리할 수 있었다.

아직 직접 사용할 기회는 많지 않았지만 SQLD나 데이터 분석 면접에서도 자주 등장하는 개념인 만큼 주기적으로 복습할 필요가 있을 것 같다.

---

# 🚀 반복하자!!

* OVER()가 나오면 윈도우 함수
* PARTITION BY = 그룹 나누기
* RANK = 공동 순위, 건너뜀
* DENSE_RANK = 공동 순위, 안 건너뜀
* ROW_NUMBER = 무조건 고유 순위
* LAG = 이전 행
* LEAD = 다음 행
* ROLLUP = 부분합
* CUBE = 모든 조합 집계
* GROUPING SETS = 원하는 집계만 생성
* 윈도우 함수는 원본 행을 유지한다
* GROUP BY는 행 수가 줄어든다
