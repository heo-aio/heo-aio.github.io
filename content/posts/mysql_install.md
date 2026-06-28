+++
title = "[DB] Windows 환경에서 MySQL 설치 및 터미널 접속"
date = 2026-06-28T17:03:00+09:00
draft = false
tags = ["Database", "MySQL", "Windows"]
categories = ["db"]
+++

데이터베이스를 학습하기 위해서는 SQL 문법뿐 아니라 로컬 환경에서 직접 데이터베이스를 설치하고 실행해보는 과정도 중요하다.

이번에는 Windows 환경에서 **MySQL**을 설치한 뒤, PowerShell을 이용하여 MySQL 서버를 실행하고 터미널에서 접속하는 과정을 실습해보았다.

---

# 📥 MySQL 설치

Windows 환경에서 MySQL을 설치하는 방법은 크게 두 가지가 있다.

- **MySQL Installer**를 이용하여 설치
MySQL 공식문서 링크 : https://dev.mysql.com/downloads/installer/

- **Chocolatey**와 같은 패키지 매니저를 이용하여 설치
chocolatey 링크 : https://chocolatey.org/install

일반적으로는 공식 홈페이지에서 제공하는 **MySQL Installer**를 이용하는 방법을 많이 사용한다.

### 💡 설치 시 주의사항

설치 과정에서 **root 계정의 비밀번호**를 설정하게 된다.

이 비밀번호는 이후 MySQL에 접속하거나 데이터베이스를 관리할 때 계속 사용되므로 반드시 기억해두는 것이 좋다.

### 💻 설치 화면

![MySQL 설치](/images/db/install_Mysql.png)

> Chcolayey를 이용하여 설치를 진행한 화면

---

# 🖥️ PowerShell 실행

MySQL 설치가 완료되면 cmd를 **관리자 권한**으로 실행한다.

Windows 검색창에서 **cmd**을 검색한 뒤 **관리자 권한으로 실행**을 선택하면 된다.

---

# 🔍 MySQL 버전 확인

설치가 정상적으로 완료되었는지 확인하기 위해 버전을 조회한다.

```bash
mysql - mysql --version
```

### 💻 실행 결과

![MySQL Version](/images/db/version_sql.png)

> `mysql - mysql --version` 명령을 실행하여 설치된 MySQL 버전을 확인한 결과

---

# ▶️ MySQL 서비스 시작

MySQL 서버가 실행 중인지 확인하고 필요하다면 서비스를 시작한다.

```bash
net start mysql
```

### 💡 실행 결과

이미 실행 중이라면 다음과 같은 메시지가 출력된다.

```
요청한 서비스가 이미 시작되었습니다.
```

이는 MySQL 서버가 정상적으로 실행 중이라는 의미이다.

### 💻 실행 결과

![MySQL Service](/images/db/start_sql.png)

> MySQL 서비스를 시작한 결과

---

# 🔐 MySQL 접속

MySQL에 접속할 때는 다음 명령어를 사용한다.

```bash
mysql -u root -p
```

명령을 실행하면 비밀번호 입력창이 나타난다.

설치 과정에서 설정한 root 비밀번호를 입력하면 MySQL Monitor에 접속할 수 있다.

### 💻 실행 결과

![MySQL Login](/images/db/root_sql.png)

> root 계정으로 MySQL에 정상적으로 접속한 화면

접속이 완료되면 다음과 같은 메시지가 출력된다.

```
Welcome to the MySQL monitor.
```

---

# 🚪 MySQL 종료

실습을 마친 후에는 MySQL Monitor를 종료하면 된다.

```bash
\q
```

또는

```bash
exit
```

명령어를 입력해도 동일하게 종료된다.

### 💻 실행 결과

![\q](/images/db/bye_sql.png)

> `\q` 명령어를 입력하여 MySQL을 종료한 화면

---

# 📌 정리

이번 실습에서는 Windows 환경에서 MySQL을 설치하고 터미널을 통해 직접 접속하는 과정을 수행하였다.

실습을 통해 다음 내용을 익힐 수 있었다.

- MySQL 설치
- PowerShell 실행
- MySQL 버전 확인
- MySQL 서비스 실행
- root 계정 접속
- MySQL 종료

SQL을 작성하는 것도 중요하지만, 실제 개발 환경에서는 데이터베이스를 직접 설치하고 실행하는 과정도 자주 경험하게 된다.

기본적인 환경 설정 방법을 익혀두면 이후 데이터베이스 실습이나 프로젝트를 진행할 때 훨씬 수월하게 개발 환경을 구축할 수 있다.

---

# ✍️ 회고

지금까지는 강의에서 제공하는 데이터베이스 환경만 사용해왔는데, 이번에는 직접 MySQL을 설치하고 터미널에서 접속해보면서 개발 환경을 구축하는 과정도 중요한 경험이라는 것을 느꼈다.

처음에는 터미널 명령어가 낯설었지만 직접 하나씩 실행해보니 데이터베이스가 어떻게 실행되고 관리되는지 조금 더 이해할 수 있었다.

앞으로는 단순히 SQL 문법을 공부하는 것에 그치지 않고, 실제 개발 환경에서도 자연스럽게 사용할 수 있도록 다양한 명령어와 데이터베이스 관리 방법도 꾸준히 익혀나갈 계획이다.
