+++
title = "[Git] GitHub 협업 실습 - 브랜치 생성부터 Pull Request, Merge까지"
date = 2026-10-10
draft = false
categories = ["dev"]
tags = ["Git", "GitHub", "협업"]
summary = "그동안 아무 생각 없이 add, commit, push만 했는데, 협업의 정석 흐름을 직접 따라 해봤다. feature 브랜치를 만들고, Pull Request를 올리고, merge한 뒤 브랜치를 정리하기까지의 과정을 정리했다."
+++

[앞 글](#)에서 Git과 GitHub의 차이, SSH 인증, 브랜치와 병합 전략 같은 **개념**을 정리했다. 이번 글은 그 개념을 실제로 따라 해본 **실습 편**이다. 블로그에 `add .`, `commit`, `push`만 습관처럼 쳐왔는데, 각 단계가 왜 있는지 알고 나니 협업의 흐름이 한 줄로 이어졌다.

## 목차
- 🧭 협업의 정석 흐름
- 📝 첫 커밋과 push (status로 상태 확인하기)
- 🌿 feature 브랜치 만들고 작업하기
- 📊 git log로 브랜치 구조 보기
- 🔁 Pull Request 생성하고 Merge하기
- 🔄 merge 후 로컬 동기화와 브랜치 정리
- 🛠️ 헷갈렸던 부분
- 💡 정리하면

---

## 🧭 협업의 정석 흐름

협업의 원칙은 **main에서 직접 작업하지 않고, 작업 목적이 이름에 드러나는 브랜치를 만들어 그 안에서만 작업하는 것**이다.

```
1. git pull                       # main 최신화
2. git switch -c feature/login    # 목적 있는 브랜치 생성 + 이동
3. 작업 → add → commit            # 그 브랜치에서만 작업
4. git push -u origin feature/login
5. Pull Request → 리뷰 → main에 merge
6. git switch main → git pull     # 로컬 동기화
7. 작업 끝난 브랜치 삭제
```

브랜치 이름만 봐도 팀원이 "이 사람이 지금 뭘 하고 있구나"를 알 수 있다.

| 브랜치 이름 | 팀원이 읽는 의미 |
|---|---|
| `feature/login` | 로그인 기능 만드는 중 |
| `fix/header-error` | 헤더 버그 고치는 중 |
| `docs/readme-update` | README 수정 중 |

이렇게 하면 `main`이 항상 배포 가능한 상태로 유지되고, 작업이 서로 섞이지 않아서 충돌도 줄고 리뷰도 쉬워진다. 단, **브랜치 하나에는 작업 하나(한 가지 목적)** 만 담는다. 작업이 달라지면 새 브랜치를 딴다.

## 📝 첫 커밋과 push (status로 상태 확인하기)

VSCode에서 `README.md`를 만들고 저장(`Ctrl+S`)한 뒤 터미널에서 실행했다.

```bash
git status                       # Untracked files: README.md (빨간색)
git add README.md
git status                       # Changes to be committed (초록색)
git commit -m "docs: add README"
git push -u origin main
```

`git status`를 두 번 치는 이유는 파일이 **어느 단계에 있는지** 눈으로 확인하기 위해서다.

| 시점 | status 표시 | 의미 |
|---|---|---|
| add 전 | `Untracked files` (빨간색) | Git이 아직 추적하지 않는 새 파일 |
| add 후 | `Changes to be committed` (초록색) | 스테이징 완료, 커밋 대기 중 |

```
작업 폴더 ──add──▶ 스테이징 ──commit──▶ 로컬 저장소 ──push──▶ GitHub
```

### 커밋 메시지 앞의 `docs:`는?

커밋 메시지 앞에 붙이는 **종류 표시(접두사)** 다. 커밋 목록만 봐도 어떤 작업인지 알 수 있다.

| 접두사 | 용도 |
|---|---|
| `feat:` | 새 기능 추가 |
| `fix:` | 버그 수정 |
| `docs:` | 문서 수정 (README 등) |

> ⚠️ 저장(`Ctrl+S`)을 안 하고 add하면 변경이 반영되지 않는다. add 없이 commit하면 `nothing added to commit` 메시지가 나온다.

<!-- TODO: git status 빨간색/초록색 화면 캡처 붙이기 -->

## 🌿 feature 브랜치 만들고 작업하기

```bash
git switch -c feature/login      # 브랜치 생성 + 즉시 전환
```

VSCode 좌측 하단 상태바에 현재 브랜치 이름이 표시된다. `feature/login`으로 바뀐 걸 확인하고 작업하는 습관을 들이자.

`login.py`를 만들어서 아래 코드를 작성했다.

```python
def login(username, password):
    """사용자 로그인 처리"""
    if not username or not password:
        return False
    return True
```

그다음 커밋하고 **내 브랜치를** push한다.

```bash
git add login.py
git commit -m "feat: add login function"
git push -u origin feature/login
```

- 이번엔 `origin main`이 아니라 **`origin feature/login`** 이다. 내 브랜치를 GitHub에 올리는 것이기 때문이다.
- `-u`는 `--set-upstream`의 줄임말로, 로컬 브랜치와 원격 브랜치를 연결한다. **브랜치가 처음 push될 때 한 번만** 붙이면 되고, 이후엔 `git push`, `git pull`만 쳐도 된다.

## 📊 git log로 브랜치 구조 보기

```bash
git log --oneline --graph --all
```

| 옵션 | 의미 |
|---|---|
| `--oneline` | 커밋을 한 줄로 요약 |
| `--graph` | 브랜치가 갈라지고 합쳐지는 모습을 선으로 표시 |
| `--all` | 현재 브랜치만이 아니라 모든 브랜치 표시 |

```
* a1b2c3d (HEAD -> feature/login, origin/feature/login) feat: add login function
* 9f8e7d6 (origin/main, main) docs: add README
```

- `HEAD -> feature/login`: **지금 내가 서 있는 브랜치**
- `origin/...`: GitHub(원격)에 있는 브랜치의 위치

협업하면서 가장 자주 쓰게 될 명령어라고 한다. 머릿속에 브랜치 구조 그림이 그려져야 충돌이나 merge를 다룰 때 안 헷갈린다.

<!-- TODO: git log --graph 결과 캡처 붙이기 -->

## 🔁 Pull Request 생성하고 Merge하기

**Pull Request(PR)** 는 "내 브랜치를 main에 **가져가(pull) 주세요**"라고 팀에 보내는 요청서다. 바로 merge하지 않고 PR을 거치는 이유는 **리뷰**를 받기 위해서다.

### 1. PR 생성 (브라우저)

1. GitHub 저장소 페이지 새로고침
2. 노란색 `Compare & pull request` 배너 클릭 (안 보이면 `Pull requests` 탭 → `New pull request`)
3. PR 제목 작성: `feat: add login function`
4. 본문 작성
5. `Create pull request` 클릭

### 좋은 PR 본문 템플릿

리뷰어가 5분 안에 맥락을 파악할 수 있도록 **Why · What · Test** 를 담는다.

```markdown
## 📌 작업 배경
사용자가 서비스를 이용하려면 로그인이 필요합니다.
기본 로그인 함수의 뼈대를 먼저 추가합니다.

## ✨ 변경 내용
- `login.py` 파일 추가
- 사용자명/비밀번호 검증 `login()` 함수 구현
- 빈 입력값 유효성 검사 포함

## 🧪 테스트 방법
- `login('user', 'pw')` → True 반환 확인
- `login('', '')` → False 반환 확인
```

### 2. 리뷰 후 Merge

1. `Files changed` 탭에서 바뀐 줄만 초록(추가)/빨강(삭제)으로 확인
2. (선택) 코드 줄의 `+` 아이콘으로 리뷰 코멘트 작성
3. `Conversation` 탭으로 돌아와 `Merge pull request` → `Confirm merge`

`Merge pull request` 버튼 옆 ▾ 에서 앞 글에서 정리한 **병합 전략 3가지**(Merge Commit / Squash / Rebase)를 고를 수 있다. 기본은 Merge Commit이다.

<!-- TODO: PR 생성 화면, Files changed 화면 캡처 붙이기 -->

## 🔄 merge 후 로컬 동기화와 브랜치 정리

```bash
git switch main
git pull
git log --oneline --graph --all

# 머지된 feature 브랜치 정리
git branch -d feature/login
git push origin --delete feature/login
```

merge는 **GitHub(원격)에서** 일어났다. 내 컴퓨터의 `main`은 아직 옛날 상태이기 때문에, `main`으로 돌아가 `git pull`로 최신화해야 한다. 자주 놓치는 부분이다.

### 끝난 브랜치는 삭제하는 게 좋다

- merge된 커밋은 이미 `main`에 들어가 있어서, 브랜치를 지워도 **작업 내용과 기록은 그대로** 남는다.
- GitHub PR 페이지에 기록이 남고, 필요하면 `Restore branch`로 복구도 된다.
- 지우지 않으면 `feature/...`가 쌓여서 어떤 게 진행 중인지 알 수 없다.
- 삭제 전에 **PR이 merge 완료**됐는지 꼭 확인한다. merge 안 된 작업이 남아 있다면 삭제 금지.
- `main`이나 `develop` 같은 **장기 브랜치**는 삭제하지 않는다.

`-d`는 **이미 merge된 브랜치만** 지워주는 안전한 옵션이다. 다만 Squash나 Rebase로 merge했다면 커밋이 압축되거나 새로 만들어져서 Git이 "merge 안 됐다"며 거부할 수 있다. 이때는 PR이 merge된 걸 확인한 뒤에만 강제 삭제(`-D`)를 쓴다.

```bash
git branch -D feature/login   # 강제 삭제 (merge 확인 후에만!)
```

## 🛠️ 헷갈렸던 부분

**1) 원격에 올리려면 `git init`이 필수인가?**
→ 필수다. 저장소가 아닌 폴더에서는 `add`, `commit`, `push`가 안 된다. 다만 상황에 따라 직접 칠 때와 자동으로 될 때가 있다.

| 상황 | init 직접 필요? |
|---|---|
| 내 컴퓨터에 만든 폴더를 GitHub에 올릴 때 | ✅ 필요 (`init` → `remote add origin` → `push`) |
| GitHub 저장소를 `git clone`으로 받았을 때 | ❌ 불필요 (clone이 자동으로 해줌) |

저장소인지 모를 땐 `git status`를 쳐본다. 저장소가 아니면 `fatal: not a git repository`가 나온다. `init`은 폴더당 한 번만 하면 된다.

**2) `git add .`는 괜찮은 걸까?**
→ `git add 파일명`은 특정 파일만, `git add .`은 **바뀐 파일 전부**를 스테이징한다. 편한 만큼 `.env` 같은 올리면 안 되는 파일까지 같이 올라갈 수 있다. `.gitignore`를 먼저 챙기고, add 전에 `git status`로 **뭐가 올라가는지 확인**하는 습관을 들이면 실수가 줄어든다.

**3) 브랜치는 merge 후 삭제하는 게 좋은가?**
→ 좋다. 위의 "끝난 브랜치는 삭제하는 게 좋다" 참고.

## 💡 정리하면

| 단계 | 명령어 / 행동 |
|---|---|
| 최신화 | `git pull` |
| 브랜치 생성 + 이동 | `git switch -c feature/기능명` |
| 상태 확인 | `git status` |
| 커밋 | `git add 파일` → `git commit -m "feat: ..."` |
| 내 브랜치 올리기 | `git push -u origin feature/기능명` |
| 브랜치 구조 보기 | `git log --oneline --graph --all` |
| PR 생성 → 리뷰 → merge | GitHub 웹에서 |
| 로컬 동기화 | `git switch main` → `git pull` |
| 브랜치 삭제 | `git branch -d` / `git push origin --delete` |

- 협업의 정석은 **목적이 드러나는 브랜치 생성 → 그 안에서만 작업 → PR → merge → 브랜치 삭제**의 한 사이클이다.
- 커밋 메시지와 브랜치 이름의 접두사(`feat:`, `fix:`, `docs:`)로 작업 종류를 알린다.
- merge는 원격에서 일어나므로, 끝나면 **로컬 `main`을 pull**로 동기화한다.
- "되니까 쓴다"에서 **"왜 되는지 안다"** 로 넘어가는 중이다.
- 다음엔 충돌(conflict)을 직접 만들어서 해결해볼 예정이다.