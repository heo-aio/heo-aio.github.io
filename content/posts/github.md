+++
title = "[Git] Git과 GitHub 기초 - 인증, 브랜치, 병합까지 한 번에 정리"
date = 2026-10-10
draft = false
categories = ["dev"]
tags = ["Git", "GitHub", "협업"]
summary = "GitHub은 알고 있다고 생각했는데, 강의 커리큘럼을 따라가다 보니 헷갈리는 게 많았다. Git과 GitHub의 차이부터 SSH 인증, 브랜치, 병합 전략까지 공부하면서 궁금했던 점 위주로 정리했다."
+++

GitHub은 이미 써본 적이 있어서 "알고 있다"고 생각했는데, 강의 커리큘럼에 들어 있어서 처음부터 다시 공부하기로 했다. 막상 정리해보니 **어떤 인증 방식으로 push가 되고 있었는지**, **브랜치가 정확히 뭔지** 같은 기본적인 부분을 제대로 모르고 있었다. 공부하면서 헷갈렸던 질문과 답을 중심으로 정리했다.

## 목차
- 🧭 Git과 GitHub는 뭐가 다를까
- 🔐 CLI에서 GitHub 인증하기
- 🚀 첫 저장소 만들고 push하기
- 🙈 .gitignore
- 📥 Repository 생성하고 로컬로 가져오기
- 🔄 작업 시작 전에는 pull부터
- 🌿 브랜치(Branch)
- 🔀 병합(Merge)과 병합 전략
- 🛠️ 헷갈렸던 부분
- 💡 정리하면

---

## 🧭 Git과 GitHub는 뭐가 다를까

| 구분 | Git | GitHub |
|---|---|---|
| 종류 | 분산 버전 관리 **시스템(소프트웨어)** | Git 저장소를 호스팅하는 **클라우드 플랫폼(서비스)** |
| 위치 | 내 컴퓨터(로컬)에 설치 | 웹(원격 서버) |
| 역할 | 코드 변경 이력 기록, 브랜치 관리, 커밋 | 원격 저장소 제공, Pull Request, 코드 리뷰, 이슈 관리, 협업 |
| 개발사 | Linus Torvalds (2005) | Microsoft (2008년 출시, 2018년 인수) |
| 사용 방식 | `git commit`, `git branch` 등 명령어 | 웹 UI + Git 명령어로 push/pull |

- **Git** = 내 컴퓨터에서 코드 변경 이력을 관리하는 *도구*
- **GitHub** = 그 Git 저장소를 인터넷에 올려 공유하고 협업하는 *서비스*

비유하면 Git은 **워드의 변경 내용 추적 기능**이고, GitHub는 **그 파일을 올려두는 구글 드라이브**다.

GitHub 말고도 GitLab, Bitbucket 같은 Git 호스팅 서비스가 있다. 그래서 "Git ≠ GitHub"라는 점이 중요하다.

## 🔐 CLI에서 GitHub 인증하기

CLI에서 GitHub과 통신하려면 "내가 누구인지" 증명해야 한다. 이 단계의 명령어는 모두 VSCode 통합 터미널에서 실행했다. (터미널 토글: `Ctrl + `` / Mac: `Cmd + ``)

### 공통: Git 사용자 정보 설정 (1회만)

```bash
git config --global user.name "본인 이름"
git config --global user.email "GitHub 가입 이메일"
git config --global --list | grep user
```

> ⚠️ 여기 입력하는 이메일은 **GitHub 가입 이메일과 같아야** 한다. 다르면 커밋이 내 것으로 인식되지 않고, GitHub 잔디(contribution graph)에도 안 찍힌다.

### 인증 방식은 두 가지

| | SSH 키 | HTTPS + Token |
|---|---|---|
| 특징 | 한 번 설정하면 매번 비밀번호 없이 push/pull | Personal Access Token 발급 필요 |
| 추천 상황 | 개인 개발 환경 | 회사 보안 정책 등으로 SSH가 막힌 환경 |

### 방법 1. SSH 키 생성과 GitHub 등록

SSH는 **열쇠 한 쌍**으로 동작한다.
- **공개키(`.pub`)** = 자물쇠 → GitHub에 등록
- **비밀키** = 열쇠 → 내 컴퓨터에만 보관 (절대 공유 금지)

```bash
# 1) SSH 키 생성
ssh-keygen -t ed25519 -C "you@example.com"
# → 저장 위치: Enter로 기본값
# → 비밀번호: 선택사항 (Enter로 skip 가능)

# 2) 공개키 내용 출력
cat ~/.ssh/id_ed25519.pub
# → ssh-ed25519로 시작하는 한 줄 전체를 복사
```

> 이미 키가 있으면 `id_ed25519 already exists. Overwrite (y/n)?` 가 뜬다. 보통 `n`을 눌러 기존 키를 그대로 쓰는 게 안전하다.

**GitHub에 공개키 등록 (브라우저)**
1. 우측 상단 프로필 아이콘 → `Settings`
2. 왼쪽 사이드바 `SSH and GPG keys` → `New SSH key`
3. `Title`에 식별용 이름(예: "My MacBook"), `Key`에 복사한 공개키 붙여넣기
4. `Add SSH key` 클릭

**연결 테스트**

```bash
ssh -T git@github.com
```

`Hi {username}! You've successfully authenticated, but GitHub does not provide shell access.` 가 나오면 성공이다. 핵심은 앞부분의 `authenticated`이고, 뒷부분은 GitHub의 정책 안내일 뿐이다.

<!-- TODO: ssh -T 성공 화면 캡처 붙이기 -->

### 방법 2. Personal Access Token (HTTPS)

1. `Settings` → 좌측 맨 아래 `Developer settings`
2. `Personal access tokens` → `Tokens (classic)` → `Generate new token (classic)`
3. `Note`, `Expiration` 설정
4. Scope에서 `repo` 체크
5. `Generate token` 후 표시된 토큰을 **즉시 복사**

> ⚠️ 토큰은 **한 번만** 보인다. 새로고침하거나 닫으면 다시 못 보니 비밀번호 매니저 등 안전한 곳에 저장해둔다.

토큰을 자동 저장하려면 OS에 맞는 Credential Helper를 설정한다.

```bash
# macOS
git config --global credential.helper osxkeychain
# Windows
git config --global credential.helper manager
# Linux
git config --global credential.helper "cache --timeout=3600"
```

## 🚀 첫 저장소 만들고 push하기

GitHub에서 새 저장소를 만들면 보여주는 기본 명령어 세트다.

```bash
echo "# test" >> README.md
git init
git add README.md
git commit -m "first commit"
git branch -M main
git remote add origin https://github.com/elice/test.git
git push -u origin main
```

| 명령어 | 의미 |
|---|---|
| `echo "# test" >> README.md` | `README.md` 파일 생성 (내용: `# test`) |
| `git init` | 이 폴더를 Git 저장소로 **초기화** (`.git` 폴더 생성) |
| `git add README.md` | 커밋할 파일을 **스테이징** (택배 상자에 담기) |
| `git commit -m "..."` | 변경 내용을 **기록으로 저장** (내 컴퓨터 안에서만) |
| `git branch -M main` | 현재 브랜치 이름을 `main`으로 **강제 변경** |
| `git remote add origin URL` | 원격 저장소 주소에 `origin`이라는 **별명** 붙이기 |
| `git push -u origin main` | `main` 브랜치를 `origin`(GitHub)에 **업로드** |

- `origin`은 관례적인 별명이다. 매번 긴 URL을 치지 않으려고 붙인다.
- `-M`: 브랜치 이름을 강제로 바꾸는 옵션. 기본 브랜치가 `master`인 환경에서 `main`으로 맞출 때 쓴다.
- `-u`: "앞으로 `main`은 `origin/main`과 연결해둬"라는 뜻. 한 번만 쓰면 이후엔 `git push`만 쳐도 된다.
- 위 예시는 `https://` 주소(HTTPS 방식)다. SSH를 설정했다면 `git@github.com:elice/test.git` 형태로 써야 한다.

흐름 한 줄 요약: `init → add → commit → 원격 연결 → push`

## 🙈 .gitignore

`.gitignore`는 **"Git아, 이 파일들은 못 본 척해줘"** 라고 알려주는 목록 파일이다. 등록된 항목은 `git status`에 안 뜨고, 커밋에도 안 들어가고, GitHub에도 안 올라간다.

| 종류 | 예시 | 이유 |
|---|---|---|
| 설치된 라이브러리 | `node_modules/`, `venv/` | 용량이 크고 다시 설치하면 됨 |
| 빌드 결과물 | `dist/` | 코드로 재생성 가능 |
| **환경 변수 파일** | `.env` | **API 키, 비밀번호 유출 방지 (가장 중요)** |
| IDE 설정 | `.vscode/` | 개인 환경 설정이라 공유 불필요 |
| OS 생성 파일 | `.DS_Store` | Mac이 자동으로 만드는 파일 |

AI 서비스를 만들 때는 `.env`에 OpenAI, Claude 같은 **API 키**를 보관하게 된다. 이걸 GitHub에 올리면 키가 그대로 노출되고, 누군가 가져다 써서 요금 폭탄을 맞을 수 있다. Python 프로젝트라면 아래처럼 시작하면 된다.

```
.env
venv/
__pycache__/
.DS_Store
```

> ⚠️ 이미 커밋된 파일은 `.gitignore`에 추가해도 계속 추적된다. **처음 `git add` 하기 전에** 만들어두자. GitHub에서 저장소를 만들 때 Python 템플릿으로 자동 생성할 수도 있다.

## 📥 Repository 생성하고 로컬로 가져오기

**GitHub에서 저장소 만들기**
1. 우측 상단 `+` → `New repository`
2. Repository name 입력 (예: `git-collab-demo`)
3. `Public` 선택
4. `Add a README file` **체크 해제** (로컬에서 만들 거라서)
5. `Create repository`

**로컬로 가져오기 (clone)**

```bash
git clone git@github.com:USERNAME/git-collab-demo.git
code git-collab-demo     # VSCode 새 창으로 열기
git remote -v            # origin이 자동 연결되어 있는지 확인
```

- `clone`은 원격 저장소를 **통째로 복사**해서 내 컴퓨터에 폴더로 만든다. 앞에서 한 `init` + `remote add origin`과 비슷한 효과를 한 번에 해주는 셈이라, `git remote -v`를 쳐보면 origin이 이미 연결돼 있다.
- 방향이 반대다. 처음 방식은 **로컬 → 원격**(`push`), clone은 **원격 → 로컬**이다.
- SSH 키를 등록하지 않고 `git@github.com:...` 주소로 clone하면 `Permission denied`가 난다.
- `code` 명령어가 안 먹으면 `Ctrl+Shift+P` → `Shell Command: Install 'code' command in PATH`를 한 번 실행한다.

## 🔄 작업 시작 전에는 pull부터

협업 환경에서는 동료의 변경 사항이 이미 원격 저장소에 반영돼 있을 수 있다. 내 컴퓨터가 옛날 버전인 채로 작업하면 나중에 **충돌(conflict)** 이 난다. 그래서 작업 전에 먼저 최신 상태로 맞춘다.

```bash
git pull
# = git fetch + git merge
```

| 명령어 | 하는 일 |
|---|---|
| `git fetch` | 원격의 최신 내용을 **가져오기만** 함 (내 코드는 안 바뀜) |
| `git merge` | 가져온 내용을 **내 코드에 합침** |
| `git pull` | 위 두 개를 **한 번에** |

루틴: `pull (최신화) → 작업 → add → commit → push`

## 🌿 브랜치(Branch)

**브랜치는 원래 코드는 그대로 두고, 안전하게 작업하는 "작업 줄기"** 다. 워드 문서를 고칠 때 원본이 불안해서 "복사본_수정중.docx"를 만들어 작업하는 것과 비슷하다. 다만 파일을 통째로 복사하는 게 아니라 Git이 변경 이력만 가볍게 나눠서 관리한다.

```
main    ●──●──────────●──●
             \        /
feature       ●──●──●
```

- `main`은 **항상 정상 동작하는 코드**로 유지한다.
- 새 기능이나 실험은 **별도 브랜치**에서 한다.
- 완성되면 `main`에 합친다(merge). 망하면 그 브랜치만 버리면 된다.

| 명령어 | 의미 |
|---|---|
| `git branch` | 브랜치 목록 보기 (현재 브랜치에 `*`) |
| `git branch 이름` | 브랜치 **만들기만** (이동은 안 함) |
| `git switch 이름` | 이미 있는 브랜치로 **이동만** |
| `git switch -c 이름` | **만들면서 바로 이동** (`-c` = create) |

### 목적이 있는 브랜치를 만들고 → 이동해서 작업

```bash
git switch -c feature/login         # 로그인 기능용 브랜치 생성 + 이동
# (코드 수정)
git add .
git commit -m "로그인 기능 추가"
git push -u origin feature/login    # GitHub에 올리기
```

브랜치 이름은 목적이 드러나게 짓는 게 관례다.

| 접두사 | 용도 | 예시 |
|---|---|---|
| `feature/` | 새 기능 | `feature/login` |
| `fix/` | 버그 수정 | `fix/header-error` |
| `docs/` | 문서 작업 | `docs/readme-update` |

전체 협업 흐름: `브랜치 생성·이동 → 작업·커밋 → push → Pull Request → 리뷰 → main에 merge`

> 작업 전에 **지금 어느 브랜치인지** 꼭 확인하자 (`git branch`). 실수로 `main`에서 바로 작업하는 게 초보 때 가장 흔한 실수라고 한다.

## 🔀 병합(Merge)과 병합 전략

### Merge

한 브랜치의 변경 사항을 다른 브랜치로 통합하는 과정이다. 이 과정을 통해 모든 구성원이 같은 메인 코드베이스에서 작업할 수 있다.

| 용어 | 의미 |
|---|---|
| `Common base` | 두 브랜치가 갈라지기 직전의 공통 지점 |
| `Main tip` | `main` 브랜치의 가장 최신 커밋 |
| `Feature tip` | 기능 브랜치의 가장 최신 커밋 |

```bash
git switch main        # 합침을 "받는" 브랜치로 먼저 이동
git merge feature      # feature를 현재 브랜치(main)에 합치기
```

핵심은 **받는 쪽 브랜치로 먼저 이동한 뒤 merge** 하는 것이다. 두 브랜치가 같은 파일의 같은 줄을 고쳤다면 Git이 판단할 수 없어서 **충돌(conflict)** 이 나고, 직접 수정해서 해결해야 한다.

<!-- TODO: 충돌 해결 실습하면 내용 추가 -->

### 브랜치 병합 전략

GitHub의 Pull Request에서 병합할 때, **커밋 기록을 어떻게 남길지** 고르는 3가지 방법이다. (feature 브랜치에서 커밋 A → B → C를 했다고 가정)

| 전략 | 결과 | 비유 |
|---|---|---|
| **Merge Commit** (기본) | A, B, C 모두 보존 + 합침 커밋 1개 추가 | 작업 일기를 통째로 붙이기 |
| **Squash and Merge** | A+B+C를 커밋 1개로 압축 | 일기를 한 줄 요약해서 붙이기 |
| **Rebase and Merge** | A, B, C를 main 끝에 일렬로 이어 붙임 | 내 일기를 main 맨 뒤로 옮겨 적기 |

```
Merge Commit    ●──●──●────────M     (갈라졌다 합쳐진 모양)
                     \        /
                      A──B──C

Squash          ●──●──●──[ABC]       (한 줄, 커밋 1개)

Rebase          ●──●──●──A'─B'─C'    (한 줄, 커밋 3개)
```

| 전략 | 장점 | 단점 |
|---|---|---|
| Merge Commit | 기록이 전부 남음 (가장 안전) | 히스토리가 복잡해짐 |
| Squash and Merge | 히스토리가 깔끔 | 작업 세부 기록이 사라짐 |
| Rebase and Merge | 히스토리가 일직선 | 충돌 시 복잡, 커밋 해시가 바뀜 |

Rebase는 커밋을 *새로 만들어 옮기는* 방식이라 A → A'처럼 커밋 ID(해시)가 바뀐다. 그래서 이미 공유된 브랜치에 쓰면 팀원과 꼬일 수 있다. 처음엔 **Merge Commit(기본)** 으로 시작하고, 팀마다 정한 규칙이 있다면 그걸 따르면 된다. 이 옵션은 명령어가 아니라 **Pull Request의 병합 버튼 옆 ▾** 에서 고른다.

## 🛠️ 헷갈렸던 부분

**1) SSH 키는 GitHub 계정에 한 번만 등록하면 되나?**
→ **기기(컴퓨터)마다** 한 번씩이다. 같은 컴퓨터에서는 한 번이면 끝이고, 저장소를 새로 만들 때마다 다시 할 필요는 없다. 새 노트북을 사거나 OS를 재설치하면 새 키를 만들어 **추가**하면 된다. GitHub 계정에는 키를 여러 개 등록할 수 있으니 Title을 "집 데스크탑", "노트북"처럼 구분해둔다. 공용 PC에서는 키를 만들지 않는 게 좋다.

**2) SSH 인증을 안 했는데 push, commit이 잘 됐었는데?**
→ 이미 **HTTPS 방식**으로 인증이 돼 있었을 가능성이 높다. VSCode의 GitHub 로그인이나 Windows의 Git Credential Manager가 토큰을 저장해두고 자동으로 쓰는 경우가 흔하다. 그리고 애초에 **commit은 인증과 무관**하다. commit은 내 컴퓨터 안에서만 일어나고, 인증은 GitHub과 통신하는 `push` / `pull` / `clone`(private)에서만 필요하다. 내 방식은 저장소 폴더에서 확인할 수 있다.

```bash
git remote -v
# https://github.com/...  → HTTPS 방식
# git@github.com:...      → SSH 방식
```

**3) 브랜치는 각자 로컬에서 작업하는 코드인가?**
→ 절반만 맞다. 브랜치는 "어디에 있느냐"가 아니라 **작업 줄기**를 뜻한다. **로컬 브랜치**는 내 컴퓨터에만 있고, **원격 브랜치**는 `git push`로 GitHub에 올린 것이다. 협업에서는 보통 로컬에서 만들고 원격에도 올린다. 동료가 리뷰할 수 있고, 백업도 되고, Pull Request를 만들려면 원격에 있어야 하기 때문이다.

**4) `switch`가 브랜치를 만드는 명령어인가?**
→ 아니다. `switch`는 **이동**이고, 만드는 건 **`-c` 옵션**이다. 예전 자료에는 `git checkout 이름`, `git checkout -b 이름`이 많이 나오는데 같은 기능이고, `switch`가 더 이해하기 쉽게 나온 최신 명령어다.

## 💡 정리하면

| 하고 싶은 일 | 명령어 |
|---|---|
| 사용자 정보 설정 | `git config --global user.name / user.email` |
| SSH 키 생성 / 확인 | `ssh-keygen -t ed25519` / `ssh -T git@github.com` |
| 저장소 초기화 | `git init` |
| 원격 저장소 연결 | `git remote add origin URL` |
| 원격 저장소 복사 | `git clone URL` |
| 연결 확인 | `git remote -v` |
| 최신화 | `git pull` |
| 브랜치 생성 + 이동 | `git switch -c 이름` |
| 브랜치 이동 | `git switch 이름` |
| 합치기 | `git merge 이름` (받는 브랜치에서 실행) |

- Git은 **도구**, GitHub는 **서비스**다.
- SSH 키는 **기기마다** 한 번 등록하고, 공개키(`.pub`)만 GitHub에 올린다.
- `.env`는 반드시 `.gitignore`에 넣는다. API 키 유출 방지!
- 작업 시작 전에는 **pull**, 작업은 **목적별 브랜치**에서, 합칠 때는 **Pull Request**로.
- 병합 전략은 처음엔 기본인 Merge Commit으로 시작한다.
- 다음엔 충돌(conflict)을 직접 만들어서 해결해보고, Pull Request 실습까지 해볼 예정이다.