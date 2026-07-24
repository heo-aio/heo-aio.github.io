+++
title = "[강의 정리] 코드 컨벤션이랑 패키지 관리, uv까지"
date = 2026-07-24T10:00:00+09:00
draft = false
tags = ["Python", "코드컨벤션", "패키지관리", "uv"]
categories = ["dev"]
math = false
+++

오늘은 코드 컨벤션이랑 패키지 관리 수업이었다. 사실 둘 다 "그냥 대충 짜도 돌아가긴 하는데" 싶은 주제라 흘려듣기 쉬운데, 막상 정리하면서 왜 이게 취업 준비하는 입장에서 중요한지 좀 와닿았다. 특히 uv는 이름만 들어봤지 실제로 왜 쓰는지는 오늘 처음 제대로 이해했다.

---

## ✍️ 코드 컨벤션, 결국 남 보라고 짜는 코드

컨벤션은 파이썬 코드를 일관되고 읽기 쉽게 쓰기 위한 규칙이다. 들여쓰기, 변수명, 함수명, 공백, 줄 길이, import 순서 같은 것들이 다 포함된다.

```python
# Bad
def f(x, y):
    return x + y

# Good
def add_numbers(first_number, second_number):
    return first_number + second_number
```

당연한 얘기 같지만 막상 급하게 짤 때는 `f(x, y)` 스타일로 짜고 넘어가는 경우가 많다. 근데 이런 게 쌓이면 나중에 나조차 내가 짠 코드를 못 알아본다.

표준 / 써드파티 / 로컬 라이브러리 구분도 나왔는데, import 순서는 이 순서로 정리하는 게 컨벤션이라고 한다. 표준 → 써드파티 → 로컬.

```python
import os                  # 표준
import numpy as np         # 써드파티

from my_module import util # 로컬
```

---

## 🐫 이름 짓는 방식 세 가지

| 표기법 | 예시 | 주로 쓰는 곳 |
|---|---|---|
| 파스칼 (PascalCase) | TextClassifier | 클래스명 |
| 카멜 (camelCase) | textClassifier | (파이썬에선 잘 안 씀, JS 쪽) |
| 스네이크 (snake_case) | text_classifier | 변수명, 함수명 |

스네이크 표기법은 원래 대소문자 구분을 못 하는 언어 환경에서 주로 썼다는 얘기가 재밌었다. 지금은 그냥 파이썬 표준 스타일로 굳어진 느낌.

```python
model, tokenizer              # 변수명
load_data(), train_model()    # 함수명 (동사 + 목적어)
TextClassifier, DataModule    # 클래스명
MAX_LEN, LEARNING_RATE        # 상수명
is_training, has_padding      # 불리언 변수명 (is/has/can으로 시작)
```

상수는 프로그램 동작 중에 안 변하는 값(비밀번호, 인증키 등)이고 전체 대문자로 쓴다.

```python
API_KEY = ""
```

---

## 📏 PEP 8, 자주 까먹는 것들 위주로

- 들여쓰기 4칸
- 줄 길이 79자 이내 권장 (사실 이거 지키는 사람 거의 못 봄...)
- 연산자 주변, 콤마 뒤에 공백
- import는 표준 → 써드파티 → 로컬 순서
- 빈 줄은 가독성 위해 적절히

---

## 🔧 함수는 한 가지 일만

1. 단일 책임 원칙: 강사님이 든 비유가 기억에 남는다. 짐싸기() → 이동하기() → 수화물적재() → 발권하기() → 비행기타기() → 자리에앉기() 처럼 쪼개는 게, 짐싸기() → 이동하기() → 공항에서처리하기() → 자리에앉기() 처럼 뭉치는 것보다 낫다는 거다. 뭉치면 재사용도 안 되고 테스트도 어려워짐.
2. 인자 수 줄이기: plus(a, oper, b)처럼. 너무 많으면 plus(*list) 고려.
3. 반환값 명확화: 예측 가능한 결과를 줘야 한다.
4. 중복 제거: 같은 코드가 여기저기 보이면 함수로 빼기.
5. 테스트 가능성: 함수가 커질수록 어디서 문제 생겼는지 찾기 힘들어진다. 지갑 잃어버리면 그 안에 뭐가 들어있었는지 하나하나 기억 안 나는 것처럼.

```
load_data → preprocess_data → train_model → evaluate_model
```

---

## 📝 주석은 '왜', docstring은 '어떻게'

주석은 무엇을 하는지보다 왜 이렇게 짰는지에 집중하는 게 맞다고 한다. 코드 자체가 이해되게 짜는 게 (self-explanatory code) 우선이고, docstring은 함수/모듈 목적과 사용법을 담는다.

```python
def tokenize(text: str) -> list[str]:
    """텍스트를 토큰 리스트로 변환하다.

    Args:
        text (str): 입력 텍스트
    Returns:
        list[str]: 토큰 리스트
    """
    ...
```

타입힌트도 같이 정리:

```python
from typing import Optional

def predict(texts: list[str]) -> list[str]:
    ...

def get_threshold() -> Optional[float]:
    ...

def is_valid(score: float) -> bool:
    ...
```

없어도 돌아가긴 하는데, IDE 자동완성 잘 되고 버그도 줄어든다니까 습관 들여야겠다.

---

## 🚨 예외 처리랑 프로젝트 구조는 간단히

- 구체적인 예외 타입 쓰기 (FileNotFoundError, requests.RequestException 등), bare except: 지양
- 예외 메시지는 맥락이 드러나게
- 복구 가능한 경우만 처리하고, 무작정 예외 삼키지 않기

프로젝트 구조는 역할별로 폴더 나누는 게 기본이다.

```
my_project/
├─ src/
├─ config/
├─ scripts/
├─ tests/
└─ README.md
```

포매터/린터는 Black(포매팅), Ruff(린팅+자동수정), isort(import 정렬) 조합을 많이 쓴다고. 저장할 때 자동 실행되게 세팅해두면 편함.

AI로 코드 짤 때 조심할 것도 슬라이드에 있었는데, 나한테 좀 뜨끔했다. 하드코딩된 경로/하이퍼파라미터, 전역 변수 남용, 한 함수가 너무 많은 일 하는 것, 중복 코드, 의미 없는 변수명 — 다 AI한테 반복 수정 요청하다 보면 슬금슬금 생기는 것들이라고 한다. 결국 AI가 짠 코드도 내가 컨벤션 알고 검토할 수 있어야 한다는 게 요지.

---

## 📦 패키지 관리 — pip 복습

PyPI는 파이썬 패키지 공식 저장소, pip는 거기서 검색·설치하는 도구.

```
pip install <패키지명>
pip install <패키지명>==버전
pip list
pip list | grep <패키지명>
pip install --upgrade <패키지명>
pip uninstall <패키지명>
```

패키지 하나만 설치해도 의존성 관리 때문에 명시 안 한 하위 패키지들이 같이 딸려온다. 여러 개 설치해야 할 땐 requirements.txt.

```
pip install -r requirements.txt
pip freeze > requirements.txt
```

근데 pip freeze는 그 순간 스냅샷일 뿐이고, 내가 직접 설치한 것과 딸려온 것(전이 의존성)이 뒤섞여서 나중에 보면 뭐가 뭔지 헷갈린다는 게 오늘 배운 포인트.

---

## ⚡ uv, 왜 다들 쓰라는지 이제 알겠다

uv는 Rust로 만든 파이썬 패키지·프로젝트 관리 도구다. pip-sync 대비 체감상 압도적으로 빠르고, pip랑 venv 여러 개를 하나로 대체하면서도 pip 명령어 그대로 호환된다. 요즘 문서들 보면 환경 세팅할 때 uv를 기본으로 쓰라고 안내하는 곳이 많아진 걸 보니 이쪽에서는 거의 표준처럼 자리잡은 느낌.

### 방식 1 — 그냥 pip 대신 쓰기

명령어만 uv로 바꾸는 방식.

```
uv venv                      # .venv 작업방 생성
uv venv --python 3.12        # 파이썬 버전 지정
uv venv --seed                # 작업방에 pip도 같이 넣기
```

```
$ uv venv
Using CPython 3.12.9 interpreter at: /usr/local/bin/python3
Creating virtual environment at: .venv
Activate with: source .venv/bin/activate
```

VS Code에서 Ctrl(Cmd)+Shift+P → Select Interpreter → .venv 선택해서 연결.

```
uv pip install pandas
uv pip install "pandas==2.2.0"
uv pip uninstall pandas
uv pip list
```

```
$ uv pip install pandas
Resolved 4 packages in 653ms
Installed 4 packages in 808ms
 + numpy==2.5.0
 + pandas==3.0.4
 + python-dateutil==2.9.0.post0
 + six==1.17.0
```

설치하고 나면 uv run 파일명으로 같은 폴더 .venv를 자동으로 찾아서 실행해준다.

### 방식 2 — 프로젝트 방식 (이게 진짜 편함)

```
uv init my-project   # pyproject.toml, main.py, README.md, .venv까지 자동 생성
uv add pandas        # 설치 + pyproject.toml 기록 + uv.lock 갱신, 한 번에
uv remove pandas
uv tree               # 의존성 트리 확인
uv lock               # 버전 고정
uv sync               # 가상환경을 uv.lock이랑 100% 똑같이 맞춤
```

uv add가 하는 일이 세 가지라는 걸 알고 나니 왜 편하다는 건지 이해됐다.

1. pyproject.toml에 패키지 기록
2. 의존성 해석해서 uv.lock 갱신
3. 가상환경에 실제 설치

uv sync는 특히 좋았는데, uv.lock에는 있는데 설치 안 된 건 설치하고 프로젝트 의존성에서 빠진 건 삭제해서 환경을 그대로 재현해준다. 팀원끼리 "나는 되는데 왜 너는 안 되냐" 하는 상황이 이걸로 줄어드는 거였다.

### pip 방식 vs uv 프로젝트 방식

| | pip / requirements.txt | uv 프로젝트 방식 |
|---|---|---|
| 의존성 기록 | requirements.txt (수동) | pyproject.toml (자동) |
| 버전 고정 | 약함 | uv.lock으로 고정 |
| 환경 복제 | pip install -r | uv sync |
| 적합한 상황 | 간단한 작업, pip 이전 환경 | 협업, 배포 |

---

## 💡 정리하면

requirements.txt를 손으로 계속 관리하던 걸 uv가 설치랑 동시에 알아서 해준다는 게 오늘 배운 핵심이다. 그리고 컨벤션 쪽은 사실 특별히 새로운 내용은 아니었는데, "AI가 짠 코드도 결국 내가 검토할 줄 알아야 한다"는 말이 계속 머리에 남는다. 다음 미니 프로젝트 할 때는 uv init → uv add → uv lock → uv sync 순서로 한번 직접 세팅해봐야겠다.
