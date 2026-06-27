````markdown
+++
title = "[DB] 데이터베이스 기초와 테이블 구성하기 (개념부터 제약조건까지)"
date = 2026-06-27T21:49:00+09:00
draft = false
tags = ["Database", "SQL"]
categories = ["db"]
+++

데이터를 효율적으로 저장하고 관리하기 위해서는 데이터베이스(Database)에 대한 이해가 필수적이다.

단순히 SQL 문법을 사용하는 것뿐 아니라, 왜 데이터베이스가 필요한지와 데이터를 안전하게 관리하기 위한 **제약조건(Constraint)** 의 역할을 이해하는 것이 중요하다.

이번 글에서는 데이터베이스의 기본 개념부터 관계형 데이터베이스(RDB)의 테이블 구성, 그리고 주요 제약조건까지 정리한다.

---

# 📦 데이터베이스(Database)란?

데이터베이스(Database)는 여러 사용자와 응용 프로그램이 **공동으로 사용할 데이터를 통합하여 저장하고 관리하는 시스템**이다.

과거에는 프로그램마다 개별 파일을 생성하여 데이터를 관리하는 **파일 처리 시스템(File Processing System)** 을 많이 사용했지만 여러 문제가 존재했다.

### 💡 예시

쇼핑몰을 만든다고 가정해 보자.

- 회원 정보
- 상품 정보
- 주문 정보

이 모든 데이터를 엑셀 파일 여러 개로 관리한다면 검색과 수정이 매우 번거롭다.

반면 데이터베이스를 사용하면 하나의 시스템에서 데이터를 체계적으로 관리하고 여러 사용자가 동시에 사용할 수 있다.

---

# ⚠️ 파일 처리 시스템의 문제점

## 1️⃣ 데이터 종속성(Data Dependency)

데이터 구조가 변경되면 이를 사용하는 프로그램도 함께 수정해야 한다.

예를 들어 회원 테이블에 새로운 컬럼(전화번호)이 추가되면 해당 데이터를 사용하는 프로그램도 수정해야 한다.

---

## 2️⃣ 데이터 중복(Data Redundancy)

동일한 데이터가 여러 파일에 저장되어 저장 공간이 낭비되고 관리가 어려워진다.

### 💡 예시

회원 이름을 변경했다고 가정해 보자.

- 회원관리.xlsx
- 주문관리.xlsx
- 배송관리.xlsx

세 파일 모두 수정해야 한다.

만약 주문관리 파일만 수정하지 않았다면 동일한 회원인데 이름이 서로 달라지는 문제가 발생한다.

---

## 3️⃣ 데이터 무결성(Integrity) 저하

중복 데이터 중 일부만 수정되는 경우 데이터의 일관성이 깨질 수 있다.

예를 들어 고객 주소를 수정했는데 일부 파일만 변경되었다면 시스템마다 서로 다른 주소를 가지게 된다.

이러한 문제를 해결하기 위해 데이터베이스가 등장하였다.

---

# 📊 데이터와 정보의 차이

|구분|설명|
|---|---|
|Data|현실 세계에서 수집한 사실이나 값 자체|
|Information|데이터를 목적에 맞게 가공하여 의미를 부여한 결과|

### 💡 예시

```
데이터(Data)

90
85
100

↓

가공

↓

정보(Information)

평균 점수 : 91.7점
최고 점수 : 100점
합격 여부 : 합격
```

---

# ⭐ 데이터베이스의 특징

## 🚀 1. 실시간 접근성 (Real-Time Accessibility)

필요한 순간 언제든 데이터를 조회할 수 있어야 한다.

**예시**

ATM에서 계좌 잔액을 조회하면 즉시 결과가 나타난다.

---

## 🔄 2. 지속적인 변화 (Continuous Evolution)

삽입(INSERT), 수정(UPDATE), 삭제(DELETE)를 통해 항상 최신 상태를 유지한다.

**예시**

쇼핑몰에서 주문이 발생하면 주문 정보가 즉시 저장된다.

---

## 👥 3. 동시 공유 (Concurrent Sharing)

여러 사용자가 동시에 데이터를 사용할 수 있다.

**예시**

수백 명의 사용자가 동시에 같은 쇼핑몰에서 주문을 진행할 수 있다.

---

## 🔍 4. 내용 기반 참조 (Content Reference)

데이터의 저장 위치가 아니라 **데이터의 값**을 기준으로 검색한다.

**예시**

회원번호뿐 아니라 이름이나 이메일을 이용해서도 회원을 검색할 수 있다.

---

# 🗄️ 관계형 데이터베이스(RDB)와 NoSQL

|구분|RDB|NoSQL|
|---|---|---|
|구조|테이블 기반|문서, Key-Value, Graph 등|
|스키마|고정|유연|
|관계|지원|필요에 따라 관리|
|대표 제품|MySQL, PostgreSQL, Oracle|MongoDB, Redis, Cassandra|

### 💡 언제 사용할까?

✅ **RDB**

- 은행
- 쇼핑몰
- ERP
- 회원관리 시스템

→ 정확성과 데이터 무결성이 중요한 서비스

✅ **NoSQL**

- SNS
- 채팅 서비스
- 로그 데이터
- 실시간 추천 시스템

→ 대용량 데이터를 빠르게 처리하는 서비스

---

# 🔒 테이블과 제약조건(Constraint)

제약조건은 **잘못된 데이터가 저장되는 것을 방지하여 데이터의 무결성을 유지하기 위한 규칙**이다.

대표적으로 다음과 같은 제약조건이 사용된다.

---

## ✅ NOT NULL

NULL 값 입력을 허용하지 않는다.

```sql
CREATE TABLE customer(
    id VARCHAR(10),
    name VARCHAR(20) NOT NULL
);
```

### 💡 예시

```
가능
이름 = 홍길동

불가능
이름 = NULL
```

---

## ✅ UNIQUE

중복 값을 허용하지 않는다.

단, NULL은 서로 다른 값으로 간주하므로 여러 개 저장될 수 있다.

```sql
CREATE TABLE customer(
    id VARCHAR(10) UNIQUE,
    name VARCHAR(20) NOT NULL
);
```

### 💡 예시

```
가능

hong99
kim99

불가능

hong99
hong99
```

---

## ✅ DEFAULT

값을 입력하지 않았을 경우 기본값을 지정한다.

```sql
CREATE TABLE customer(
    id VARCHAR(10),
    name VARCHAR(20),
    address VARCHAR(50) DEFAULT 'No Address'
);
```

### 💡 예시

```
입력

이름 : 홍길동

↓

저장

주소 : No Address
```

---

## ✅ CHECK

특정 조건을 만족하는 값만 저장할 수 있도록 제한한다.

```sql
CREATE TABLE customer(
    id VARCHAR(10),
    age INT CHECK(age >= 19)
);
```

### 💡 예시

```
가능

나이 = 25

불가능

나이 = 17
```

---

## ✅ PRIMARY KEY

행(Row)을 유일하게 구분하는 기본키이다.

다음 두 가지 특징을 가진다.

- NULL 불가
- 중복 불가

```sql
CREATE TABLE customer(
    id VARCHAR(10) PRIMARY KEY,
    name VARCHAR(20)
);
```

### 💡 예시

|id|name|
|---|---|
|1001|홍길동|
|1002|김철수|

```
불가능

id = 1001 (중복)

id = NULL
```

---

## ✅ FOREIGN KEY

다른 테이블의 PRIMARY KEY를 참조하는 제약조건이다.

테이블 간 관계(Relationship)를 표현할 때 사용한다.

```sql
CREATE TABLE orders(
    order_id INT PRIMARY KEY,
    customer_id VARCHAR(10),
    FOREIGN KEY(customer_id)
        REFERENCES customer(id)
);
```

### 💡 관계 예시

```
customer

id
-------
1001
1002

↓

orders

order_id    customer_id

1           1001
2           1002
```

존재하지 않는 `customer_id = 9999`를 입력하면 오류가 발생한다.

---

# 🏷️ CONSTRAINT로 제약조건 이름 지정

제약조건에 이름을 지정하면 관리하기 쉬워진다.

- 어떤 제약조건에서 오류가 발생했는지 쉽게 확인할 수 있다.
- 수정 및 삭제 작업이 편리해진다.

```sql
CREATE TABLE customer(
    id VARCHAR(10),
    age INT,

    CONSTRAINT customer_id_unique
        UNIQUE(id),

    CONSTRAINT customer_age_chk
        CHECK(age >= 19)
);
```

---

# 🛠️ ALTER TABLE로 제약조건 추가

이미 생성된 테이블에도 제약조건을 추가하거나 수정할 수 있다.

## UNIQUE 추가

```sql
ALTER TABLE customer
ADD CONSTRAINT customer_address_unique
UNIQUE(address);
```

---

## CHECK 추가

```sql
ALTER TABLE customer
ADD CONSTRAINT customer_age_chk
CHECK(age >= 19);
```

---

## DEFAULT 수정

```sql
ALTER TABLE customer
ALTER COLUMN address
SET DEFAULT '주소 없음';
```

---

# 📝 정리

이번 글에서는 데이터베이스의 기본 개념과 관계형 데이터베이스에서 사용하는 주요 제약조건을 정리했다.

데이터베이스는 단순히 데이터를 저장하는 공간이 아니라 **데이터의 정확성과 일관성을 유지하기 위한 시스템**이다.

특히 테이블을 설계하는 단계에서 적절한 제약조건을 설정하면 잘못된 데이터 입력을 방지하고 유지보수가 쉬운 데이터베이스를 구축할 수 있다.

SQL 문법을 암기하는 것도 중요하지만, **왜 이러한 제약조건이 필요한지**를 이해하는 것이 더욱 중요하다.
````
