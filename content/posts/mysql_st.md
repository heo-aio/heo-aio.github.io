+++
title = "[DB] MySQL 공유 킥보드 서비스 데이터베이스 설계 및 SQL 실습"
date = 2026-06-27T16:30:00+09:00
draft = false
tags = ["Database", "MySQL", "SQL"]
categories = ["db"]
+++

데이터베이스를 공부하면서 SQL 문법을 익히는 것도 중요하지만, 직접 데이터베이스를 설계하고 데이터를 관리해보는 경험도 있으면 좋을 것 같았다.

이번 실습에서는 **공유 킥보드 서비스(shared_kickboard)** 를 가정하여 데이터베이스를 설계하고, 테이블 생성부터 데이터 삽입, JOIN 조회, 권한 관리(DCL), 인덱스(Index) 생성까지 전체 과정을 실습해보았다.

---

# 🏗️ 데이터베이스 생성

먼저 프로젝트에서 사용할 데이터베이스를 생성한 후 작업할 데이터베이스를 선택한다.

```sql
CREATE DATABASE shared_kickboard;

USE shared_kickboard;
```
![테이블생성 결과](images/db/create_db.png)
---

# 📋 테이블 설계

공유 킥보드 서비스를 구성하기 위해 다음과 같이 4개의 테이블을 설계하였다.

|테이블|설명|
|---|---|
|Customer|회원 정보|
|Brand|킥보드 브랜드 정보|
|Kickboard|킥보드 정보|
|Borrow|대여 이력|

각 테이블은 **PRIMARY KEY**, **FOREIGN KEY**, **UNIQUE** 등의 제약조건을 이용하여 데이터의 무결성을 유지하도록 설계하였다.

---

## 👤 Customer 테이블

회원 정보를 저장하는 테이블이다.

```sql
CREATE TABLE customer(
    customer_number VARCHAR(10) PRIMARY KEY,
    name VARCHAR(10) NOT NULL,
    id VARCHAR(15) NOT NULL UNIQUE,
    pw VARCHAR(20) NOT NULL,
    phone_number VARCHAR(11),
    birth_date DATE
);
```

### 주요 컬럼

- customer_number : 회원번호(기본키)
- id : 로그인 아이디(중복 불가)
- phone_number : 연락처
- birth_date : 생년월일

---

## 🏢 Brand 테이블

킥보드 브랜드 정보를 저장한다.

```sql
CREATE TABLE brand(
    brand_number INT PRIMARY KEY,
    name VARCHAR(20) NOT NULL UNIQUE,
    company VARCHAR(20) NOT NULL
);
```

---

## 🛴 Kickboard 테이블

실제 킥보드 정보를 저장하는 테이블이다.

```sql
CREATE TABLE kickboard(
    id VARCHAR(4) PRIMARY KEY,
    brand_number INT NOT NULL,
    model_year INT NOT NULL,
    basic_price INT NOT NULL,
    price_per_minute INT NOT NULL,
    FOREIGN KEY (brand_number)
        REFERENCES brand(brand_number)
);
```

### 💡 설계 포인트

브랜드 번호를 **FOREIGN KEY** 로 지정하여 존재하지 않는 브랜드의 킥보드가 등록되지 않도록 하였다.

---

## 📑 Borrow 테이블

회원의 킥보드 대여 이력을 저장한다.

```sql
CREATE TABLE borrow(
    customer_number VARCHAR(10),
    rental_time DATETIME,
    lat_location FLOAT NOT NULL,
    lon_location FLOAT NOT NULL,
    rental_status ENUM('대여','반납') NOT NULL,
    kickboard_id VARCHAR(4) NOT NULL,

    CONSTRAINT borrow_pk
        PRIMARY KEY(customer_number, rental_time),

    FOREIGN KEY(customer_number)
        REFERENCES customer(customer_number),

    FOREIGN KEY(kickboard_id)
        REFERENCES kickboard(id)
);
```

### 💡 설계 포인트

- 회원번호와 대여시간을 **복합 기본키(Composite Primary Key)** 로 사용하였다.
- 회원과 킥보드 정보를 각각 참조하도록 외래키를 설정하였다.
- 대여 상태는 ENUM을 이용하여 `대여`, `반납` 두 값만 저장하도록 제한하였다.

---

# ➕ 데이터 삽입(INSERT)

테이블 생성 후 샘플 데이터를 입력하였다.

## Customer

```sql
INSERT INTO customer
VALUES('0012616925','이서연','flykite','HASH-u73ylz5jao','21865059766','1995-07-12');

INSERT INTO customer
VALUES('0187642351','김민준','kmax6','HASH-lui235dfi2','08786173448','1989-03-09');
```

---

## Brand

```sql
INSERT INTO brand
VALUES(100,'boardkick','elice');

INSERT INTO brand
VALUES(200,'willgo','everythere');

INSERT INTO brand
VALUES(201,'fastgoing','everythere');
```

---

## Kickboard

```sql
INSERT INTO kickboard
VALUES('54JP',100,2015,1000,100);

INSERT INTO kickboard
VALUES('672Z',200,2018,950,110);

INSERT INTO kickboard
VALUES('7L3D',100,2018,1050,100);

INSERT INTO kickboard
VALUES('7YWC',100,2015,1000,100);
```

---

## Borrow

```sql
INSERT INTO borrow
VALUES
('0012616925',NOW(),37.5665,126.9780,'대여','54JP');

INSERT INTO borrow
VALUES
('0187642351','2020-08-20 13:01:02',37.514194,127.065038,'대여','7YWC');

INSERT INTO borrow
VALUES
('0187642351','2020-08-20 13:12:32',37.5141,127.0600,'반납','7YWC');
```

### 💡 실습하면서 경험한 오류

데이터를 삽입하는 과정에서 다음과 같은 오류를 경험하였다.

- FOREIGN KEY 제약조건 위반
- DATETIME 형식 오류

외래키는 참조하는 데이터가 먼저 존재해야 하므로 **Brand → Kickboard**, **Customer → Borrow** 순서로 데이터를 입력해야 정상적으로 저장된다.

---

# 🔍 JOIN 조회

관계형 데이터베이스의 가장 큰 특징은 여러 테이블을 연결하여 조회할 수 있다는 점이다.

## 회원과 대여내역 조회

```sql
SELECT *
FROM customer c
JOIN borrow b
ON c.customer_number = b.customer_number;
```

### 조회 결과 예시

|회원|대여상태|
|---|---|
|이서연|대여|
|김민준|대여|
|김민준|반납|

---

## 특정 회사의 킥보드 조회

```sql
SELECT *
FROM kickboard k
JOIN brand b
ON k.brand_number = b.brand_number
WHERE b.company = 'elice';
```

### 💡 JOIN을 사용하는 이유

- 고객 정보와 주문 정보를 함께 조회
- 브랜드별 킥보드 조회
- 회원별 대여 이력 조회

처럼 여러 테이블의 데이터를 하나의 결과로 확인할 수 있다.

---

# 🔐 사용자 권한 관리(DCL)

실무에서는 모든 사용자가 관리자(root) 권한을 가지지 않는다.

필요한 권한만 부여하는 것이 보안 측면에서 매우 중요하다.

## 사용자 생성

```sql
CREATE USER jm@localhost
IDENTIFIED BY '0514';
```

---

## SELECT 권한 부여

```sql
GRANT SELECT
ON shared_kickboard.*
TO jm@localhost;
```

---

## 권한 확인

```sql
SHOW GRANTS FOR jm@localhost;
```

---

## 권한 회수

```sql
REVOKE SELECT
ON shared_kickboard.*
FROM jm@localhost;
```

---

## 변경사항 적용

```sql
FLUSH PRIVILEGES;
```

### 💡 실습 결과

SELECT 권한만 부여된 사용자로 로그인한 후 DELETE를 실행하면 다음과 같이 권한 오류가 발생한다.

```
Access denied
```

이를 통해 사용자별 권한을 분리하는 이유를 확인할 수 있었다.

---

# ⚡ INDEX 생성

검색이 자주 발생하는 컬럼에는 인덱스를 생성하여 조회 성능을 향상시킬 수 있다.

## 인덱스 생성

```sql
CREATE INDEX customer_index
ON customer(phone_number);
```

---

## 인덱스 확인

```sql
SHOW INDEX
FROM customer;
```

---

## 인덱스 삭제

```sql
ALTER TABLE customer
DROP INDEX customer_index;
```

### 💡 언제 인덱스를 사용할까?

예를 들어 회원이 수백만 명 존재한다고 가정해보자.

```sql
SELECT *
FROM customer
WHERE phone_number='01012345678';
```

이처럼 자주 검색되는 컬럼에 인덱스를 생성하면 전체 테이블을 탐색하지 않아도 되어 검색 속도를 향상시킬 수 있다.

---

# 📝 정리

이번 실습에서는 공유 킥보드 서비스를 예시로 데이터베이스를 직접 설계하고 관리하는 과정을 경험하였다.

실습을 통해 다음 내용을 익힐 수 있었다.

- 데이터베이스 및 테이블 생성
- PRIMARY KEY와 FOREIGN KEY를 활용한 관계 설정
- INSERT를 이용한 데이터 입력
- JOIN을 이용한 관계형 데이터 조회
- DCL을 활용한 사용자 권한 관리
- INDEX를 이용한 조회 성능 향상

SQL 문법을 익히는 것만큼 중요한 것은 **실제 서비스를 가정하여 데이터 모델을 설계하고 관계를 이해하는 것**이다.

이러한 실습을 반복하면서 관계형 데이터베이스의 구조와 설계 방식에 대한 이해를 더욱 높일 수 있었다.
