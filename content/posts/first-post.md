+++
title = 'Hugo 블로그 구축기'
date = 2026-06-13
draft = false

categories = ['Git']

tags = [
  'Hugo',
  'Git',
  'GitHub Pages'
]
+++

# 🚀 우여곡절 끝에 완성한 나만의 블로그!

새로운 마음으로 블로그를 개설하려다 예상치 못한 Git 에러들과 마주쳤다. 명령어 오타와 주소 꼬임 현상 등 여러 시행착오가 있었지만, 끝내 에러를 돌파하고 배포에 성공한 기록을 타임라인 형식으로 남겨본다.

---

## 🕒 파워쉘 작업 타임라인 복기

### 1단계: 새로운 사이트 생성 및 위치 확인

새로운 블로그 폴더를 생성하기 위해 아래 명령어를 입력했다.

```powershell
hugo new site jmo_o
```

> 📌 작업 확인
> `ls` 명령어로 로컬 디렉터리를 확인해 보니 이전에 만들어두었던 `heo_blog` 폴더가 존재하는 것을 발견했다.

새 폴더를 사용하는 대신 기존 폴더를 이어서 활용하기로 결정하고 아래 명령어로 이동했다.

```powershell
cd .\heo_blog\
```

---

### 2단계: Git 개발자 정보 등록

GitHub에 코드를 업로드하기 위해 로컬 환경에 사용자 정보를 등록했다.

```powershell
git config --global user.email "yourEmail@gmail.com"
git config --global user.name "Honggilldong"
```

> ⚠️ 삽질 포인트
> 중간에 사소한 오타로 `invalid key` 에러가 발생했지만 빠르게 수정하고 정상 등록을 완료했다.

---

### 3단계: 로컬 저장소 초기화 및 파일 추가

프로젝트 폴더를 Git 저장소로 관리하기 위해 초기화를 진행하고 전체 파일을 스테이징했다.

```powershell
git init
git add .
```

---

### 4단계: 💥 최대 난관, public 폴더 에러 해결

커밋을 진행하려는 순간 Git이 아래와 같은 메시지를 출력하며 진행을 거부했다.

```text
modified: public (untracked content)
```

> 📌 원인 분석
> Hugo 빌드 결과물이 저장되는 `public` 폴더 내부에 Git 설정이 꼬여 있는 상태였다.

처음에는 아래 명령어로 폴더를 삭제하려고 시도했다.

```powershell
Remove-Item -Recurse -Force public
```

> ⚠️ 삽질 포인트
> 옵션을 잘못 입력하는 바람에 또 다른 에러를 만났다.

다행히 문제 원인이 `public` 폴더에 있다는 것을 파악하고 있었기 때문에 다시 파일을 추가하고 커밋을 진행했다.

```powershell
git add .
git commit -m "Initial commit"
```

이 과정에서 다음과 같은 메시지가 나타났다.

```text
delete mode 160000 public
```

> 🚀 해결 완료
> 꼬여 있던 서브모듈 문제를 제거하며 정상적으로 커밋에 성공했다.

---

### 5단계: GitHub 아이디 변경 및 원격 저장소 연결

GitHub 아이디를 `heo-aio`로 변경했기 때문에 로컬 설정도 함께 수정했다.

```powershell
git config --global user.name "heo-aio"
```

이후 새로운 GitHub 저장소를 연결하려고 했지만 아래 에러가 발생했다.

```text
error: remote origin already exists
```

> 📌 원인 분석
> 이미 등록된 원격 저장소 주소가 있었기 때문이다.

기존 주소를 삭제하는 대신 아래 명령어로 URL만 변경했다.

```powershell
git remote set-url origin https://github.com/heo-aio/heo-aio.github.io.git
```

> 🚀 해결 완료
> 기존 원격 저장소 설정을 유지한 채 새로운 저장소 주소로 정상 변경할 수 있었다.

---

### 6단계: 🚀 대망의 첫 배포

마지막으로 GitHub 원격 저장소에 코드를 업로드했다.

```powershell
git push -u origin main
```

> 🚀 해결 완료
> 화면에 성공 메시지가 출력되며 첫 배포를 성공적으로 마무리했다.

성공 메시지가 터미널에 출력되는 순간, 그동안의 삽질이 보상받는 기분이었다.

---

## 💡 한 줄 요약

> 과거에 만들어 둔 Hugo 프로젝트를 재활용하고, 꼬여 있던 `public` 폴더 문제를 해결한 뒤 GitHub 저장소 주소까지 수정하여 성공적으로 첫 배포를 완료했다.

---

## ✍️ 느낀 점

처음 하는 과정이라 터미널에 빨간 에러 메시지가 나타날 때마다 당황스러웠다.

하지만 하나씩 원인을 찾아가며 해결하는 과정에서 단순히 명령어를 따라 입력하는 것이 아니라 Git과 GitHub가 어떻게 동작하는지 조금씩 이해할 수 있었다.

이번 경험을 통해 개발에서 에러는 실패가 아니라 문제를 해결하기 위한 힌트라는 사실을 다시 한번 느꼈다.

> 💡 배운 점
> 에러 메시지를 무조건 두려워하기보다 원인을 분석하고 하나씩 해결해 나가는 과정이 개발 실력을 키우는 가장 빠른 방법이라는 것을 체감했다.

아직 부족한 점이 많지만, 나만의 기술 블로그를 만들고 직접 배포까지 완료했다는 점에서 의미 있는 첫걸음이었다.

앞으로는 이 공간에 AI, 데이터 분석, Python, 취업 준비 과정에서 배우는 내용들을 꾸준히 기록해 나갈 예정이다.