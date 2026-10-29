+++
title = "[RAG] DSPy 정리: 프롬프트를 쓰지 않고 컴파일하는 프레임워크"
date = 2026-10-28
draft = false
categories = ["dev"]
tags = ["RAG", "DSPy", "Python"]
summary = "RAG 프레임워크 비교 수업의 네 번째, DSPy를 정리했다. 프롬프트 문장을 직접 쓰지 않고 Signature와 Module로 구조만 선언하는 방식, Optimizer가 few-shot 예시를 자동으로 넣어 정확도를 0.60에서 0.80~1.00으로 올린 실험, 그리고 4개 프레임워크 종합 비교까지 정리했다."
+++

RAG 프레임워크 비교 시리즈의 마지막, **DSPy**다. 네 개 중 가장 낯설었다. 다른 프레임워크는 "검색을 어떻게 구성하나"였는데, DSPy는 **"프롬프트를 어떻게 얻나"**가 주제다. 그래서 두 교시에 걸쳐 배웠고, 마지막에 4개 프레임워크의 종합 비교도 이 글에 같이 정리했다.

## 목차
- 🧭 DSPy는 어떤 프레임워크?
- ✍️ Signature: 입력/출력 명세
- 🧱 Module: Predict · ChainOfThought · Retrieve
- ⚙️ RAG를 모듈로 조립하기
- 🚀 Optimizer로 자동 튜닝
- 📈 컴파일 전후 비교 실험
- 🆚 4개 프레임워크 종합 비교
- 🛠️ 헷갈렸던 부분
- 💡 정리하면

---

## 🧭 DSPy는 어떤 프레임워크?

슬로건은 **"Programming, not prompting."**

> 프롬프트 문자열을 손으로 쓰지 마라. **구조만 선언**하고, 프롬프트는 **컴파일**한다.

| 구성 | 역할 |
|---|---|
| Signature | **무엇을** 할지 (입력/출력 명세) |
| Module | **어떤 구조로** 할지 |
| Optimizer | 프롬프트를 **데이터로 생성** |

수업에서 든 비유가 이해에 도움이 됐다. 프롬프트 엔지니어링이 **어셈블리 손코딩**이라면, DSPy는 **고수준 언어 + 컴파일러**다. 프롬프트를 사람이 "작성한 글"이 아니라 **"학습의 결과물"**로 본다.

## ✍️ Signature: 입력/출력 명세

```python
class GenerateAnswer(dspy.Signature):
    '''문맥에 근거해 한국어로 간결히 답한다.'''      # 작업 설명(고수준)
    context  = dspy.InputField(desc='검색된 관련 문서')
    question = dspy.InputField()
    answer   = dspy.OutputField(desc='문맥에 근거한 답변')
```

프롬프트 문구는 한 줄도 없고, **"이런 입력을 받아서 이런 출력을 낸다"는 계약만 선언**했다. Signature는 "무엇을(what)"만 정한다. "어떻게(how)", 즉 실제 프롬프트 문구와 예시는 Optimizer의 몫이다.

## 🧱 Module: Predict · ChainOfThought · Retrieve

| 모듈 | 하는 일 |
|---|---|
| `dspy.Predict` | Signature대로 한 번 추론 (가장 기본) |
| `dspy.ChainOfThought` | 답을 내기 전에 추론 과정을 거치게 함 → 복잡한 질문에서 정확도↑ |
| `dspy.Retrieve(k)` | 검색기(RM)에서 top-k 문서 가져오기 |

모듈은 **중첩·조합**할 수 있다. 여러 모듈을 묶어서 하나의 `dspy.Module`(RAG)을 만들고, **전체를 한꺼번에 최적화**할 수 있다.

## ⚙️ RAG를 모듈로 조립하기

```python
class RAG(dspy.Module):
    def __init__(self, k=3):
        self.k = k
        self.generate = dspy.ChainOfThought(GenerateAnswer)

    def forward(self, question):
        ctx = retrieve(question, self.k)          # e5 코사인 직접 구현
        return self.generate(context=ctx, question=question)
```

실행 결과는 다른 프레임워크와 같은 출처를 찾았다.

```
출처(검색): ['파인튜닝', 'RAG', '평가']   답변: RAG는 검색해 답, 파인튜닝은 추가학습…
```

그런데 이 답변을 만든 **프롬프트는 내가 쓴 적이 없다.** DSPy가 Signature를 보고 자동으로 만든 것이다. 실제로 LLM에 들어간 프롬프트를 열어 보면 이런 형태였다.

```
Your input fields are:  1. `context` ...  2. `question` ...
Your output fields are: 1. `reasoning` ...  2. `answer` ...
...
In adhering to this structure, your objective is:
        문맥에 근거해 한국어로 간결히 답한다.
```

Signature의 설명 문장은 목표(objective)로, 필드 이름과 `desc`는 입출력 형식 안내로 들어갔다. `ChainOfThought`라서 `reasoning`이라는 출력 필드도 자동으로 추가됐다.

### dspy.Retrieve(k) vs 직접 만든 _retrieve

수업에서는 검색 부분을 `dspy.Retrieve`가 아니라 **직접 구현한 함수**로 썼다. DSPy 3.x에서 RM 설정이 레거시가 되어, 비교 목적상 직접 구현을 택했다고 한다.

| 관점 | (A) `dspy.Retrieve(k)` | (B) 직접 `_retrieve` |
|---|---|---|
| 검색 백엔드 | `configure(rm=…)`로 등록한 RM 호출, 설정 교체로 변경 | 임베딩·코사인을 코드로 직접 통제 |
| DSPy 통합 | `dspy.Module`이라 트레이싱·검사 단위가 됨 | 블랙박스(그냥 파이썬) |
| 출력 | `.passages`(문자열 리스트) | dict(topic+text) → 출처 추적이 쉬움 |

여기서 **오해하기 쉬운 부분**이 있다고 짚어 줬다. `Retrieve`를 `__init__`에 넣는다고 검색이 똑똑해지지(최적화되지) 않는다. 순수 검색에는 프롬프트가 없기 때문이다. DSPy의 최적화는 멀티홉에서 **"검색 질의를 LLM이 생성"**할 때, 그 질의 생성 프롬프트가 학습되는 것이다.

## 🚀 Optimizer로 자동 튜닝

여기가 DSPy의 진짜 가치다. 수업의 ML 비유로 정리하면 이렇다.

| ML 용어 | DSPy에서 대응 |
|---|---|
| 학습 데이터 | `trainset` |
| 손실 함수 | `metric` |
| 학습 알고리즘 | Optimizer |
| 학습된 가중치 | 컴파일된 프롬프트 |

즉 **"프롬프트를 학습한다."** 손으로 프롬프트를 고치는 대신, `metric`과 `trainset`을 주면 few-shot 예시를 **자동으로 탐색해서 삽입**해 준다.

### Metric: 무엇이 좋은 답인가

| 종류 | 특징 |
|---|---|
| 단순 metric | 정답 키워드 포함 여부. 빠르지만 거칠다 |
| 의미 metric | 정답과 예측의 임베딩 유사도. 표현 차이를 흡수 |
| LLM-as-judge | 다른 LLM이 충실성·답변성을 채점. 정교하지만 비용↑ |

한국어에서는 단순 포함 검사가 조사·어미에 약하다('벡터DB는' ≠ '벡터DB'). 그래서 의미 유사도나 정규화를 권장한다. **metric이 곧 최적화의 방향**이 된다.

### Optimizer 종류

```python
# ① BootstrapFewShot: 성공한 실행을 few-shot 예시로 (입문)
opt = BootstrapFewShot(metric=metric, max_bootstrapped_demos=4)
compiled = opt.compile(RAG(), trainset=trainset)

# ② MIPROv2: 지시문 + 예시를 베이지안 탐색으로 최적화
from dspy.teleprompt import MIPROv2
opt = MIPROv2(metric=metric, auto='light')      # light / medium / heavy
compiled = opt.compile(RAG(), trainset=trainset, max_bootstrapped_demos=4)
```

| | BootstrapFewShot | MIPROv2 |
|---|---|---|
| 최적화 대상 | few-shot 예시 | 예시 + **지시문(instruction)** |
| 품질 | 입문용 | 더 높음 (특히 지시문이 중요한 과제) |
| 비용 | 낮음 | LLM 호출이 많아 비용·시간↑ → `auto='light'`로 시작 |

## 📈 컴파일 전후 비교 실험

정확한 한국어 용어를 맞혀야 하는 질문(예: "관련 문서를 놓치지 않는 정도를 보는 한국어 지표는?" → "재현율")으로 train 5개, dev 5개를 만들어 실험했다.

```
[before] 기본 프롬프트 RAG      → dev 정확도 0.60
[after ] BootstrapFewShot       → dev 정확도 0.80~1.00   (Δ+0.2~0.4)
```

개선이 눈에 보이려면 **개선할 여지(헤드룸)**가 있어야 한다. 모델이 기본 프롬프트에서는 "재현율"을 "recall"로 바꿔 말해서 틀렸기 때문에, 이런 정확 용어 메트릭으로 헤드룸을 만들었다고 한다. 강한 모델에 쉬운 과제라면 변화가 작다.

### Optimizer가 프롬프트를 어떻게 바꿨나

같은 질문을 base와 compiled로 비교해 봤다.

| | 답변 | 통과 | 프롬프트 |
|---|---|---|---|
| base | 'recall입니다' | False | 메시지 2개 |
| compiled | '재현율' | True | 메시지 8개 |

compiled 쪽 프롬프트에는 Optimizer가 **few-shot 예시(demos) 4개**를 넣어 놨다. 예를 들면 "…유사도 척도 이름은? → 코사인 유사도", "…빠뜨리지 않았는지 보는 지표? → 재현율" 같은 예시다. 내가 손으로 쓴 게 아니라 **정답 용어를 쓰는 예시가 자동으로 삽입**되어서, 모델이 'recall'이 아니라 '재현율'로 답하게 된 것이다. 이게 "프롬프트를 데이터로 학습한다"의 실제 모습이었다.

### 비용 주의

Optimizer는 LLM을 여러 번 호출한다. 하지만 **컴파일은 오프라인에서 한 번**만 하고, 서빙할 때는 컴파일된 결과를 재사용한다.

## 🆚 4개 프레임워크 종합 비교

수업의 마지막 교시에서 같은 질문, 같은 코퍼스로 세 프레임워크를 돌려 봤다.

**질문: "RAG와 파인튜닝의 차이는?"**

| 프레임워크 | 검색된 출처 | 답변 길이 | 시간 |
|---|---|---|---|
| LlamaIndex | 파인튜닝, RAG, 평가 | 300자 | 16.6s |
| Haystack | 파인튜닝, RAG, 평가 | 549자 | 12.8s |
| DSPy | 파인튜닝, RAG, 평가 | 79자 | 13.2s |

가장 중요한 관찰은 **셋 다 같은 출처를 검색했다**는 것이다. 즉 **검색 품질은 프레임워크가 아니라 임베딩 같은 구성요소가 결정**한다. 프레임워크에 따라 달라진 건 답변 스타일, 지연 시간, 코드 구조다.

한눈에 보는 비교표는 이렇다.

| 프레임워크 | 결 | 개발 편의 | 운영 | 학습곡선 |
|---|---|---|---|---|
| LangChain | 범용 오케스트레이션 | 중 | LangServe | 중 |
| LlamaIndex | 인덱싱·검색 특화 | 상 | 중 | 하~중 |
| Haystack | 컴포넌트 파이프라인 | 중 | 상(YAML) | 중 |
| RAGFlow | 엔드투엔드 엔진 | 상(웹UI) | 상(완제품) | 하 |
| DSPy | 프롬프트 최적화 | 하 | 중 | 상 |

상황별 선택 가이드는 다음과 같다.

| 상황·요구 | 1순위 |
|---|---|
| 문서 넣고 바로 질의, 다양한 인덱스 | LlamaIndex |
| 흐름을 명시적으로 관리, 서빙, 검색 제품 | Haystack |
| 복잡한 문서(스캔, 표), 비개발 운영 | RAGFlow |
| 복합 분기, 에이전트, 툴 결합 | LangChain(+LangGraph) |
| 품질을 데이터로 최적화 | DSPy |

그리고 한국어 RAG에서는 프레임워크와 별개로 **구성요소도 한국어용이어야** 한다.

| 구성요소 | 주의점 |
|---|---|
| 임베딩 | e5, bge-m3 같은 다국어/한국어 모델. 영어 모델은 점수가 무너짐 |
| 리랭커 | bge-reranker-v2-m3, ko-reranker. 영어 ms-marco는 한국어를 밀어냄 |
| BM25 토크나이저 | 형태소 분석(Kiwi/Mecab). 공백 토큰화는 'RAG와' ≠ 'RAG' |

> 프레임워크를 잘 골라도 구성요소가 영어용이면 한국어 RAG는 실패한다. 둘은 독립 변수다.

## 🛠️ 헷갈렸던 부분

**1) Signature와 Module의 차이는?**
→ Signature는 **무엇을**(입력과 출력의 계약), Module은 **어떤 구조로**(Predict, ChainOfThought 등) 처리할지다.

**2) 최적화 전에도 DSPy가 답변을 만들었는데, 그건 뭔가?**
→ 최적화 전에는 **기본 프롬프트**로 동작한다. DSPy의 진짜 가치는 Optimizer로 컴파일한 뒤에 나온다.

**3) DSPy가 다른 프레임워크의 대체재인가?**
→ 아니다. 비교의 전제는 **우열이 아니라 결**이다. 실무에서는 인덱싱은 LlamaIndex, 에이전트는 LangGraph, 튜닝은 DSPy처럼 **섞어 쓰는 경우가 흔하다.** "하나만 골라야 한다"는 전제부터 버려야 한다고 했다.

**4) 어떤 경우에 DSPy가 맞나?**
→ **평가 기준(metric)이 명확**하고 품질을 데이터로 끌어올려야 하는 실험·연구·고품질 RAG. 반대로 metric을 설계하기 어렵거나 빠르게 프로토타입만 만들 때는 부담이 크다.

## 💡 정리하면

| 개념 | 한 줄 요약 |
|---|---|
| Signature | 입력/출력 계약 (무엇을) |
| Module | Predict, ChainOfThought, Retrieve 등 구조 (어떻게 처리) |
| Optimizer | metric과 trainset으로 프롬프트(예시, 지시문)를 자동 탐색 |
| 컴파일 | 오프라인에서 한 번 최적화하고, 서빙은 재사용 |
| BootstrapFewShot / MIPROv2 | 예시 중심(입문) / 예시 + 지시문(고품질, 비용↑) |

- DSPy는 **"프롬프트를 작성하지 않고 학습한다"**는 관점의 프레임워크다.
- 강점은 **선언적, 재현 가능, 프롬프트 자동 최적화**이고, 약점은 **학습곡선, 메트릭 설계, 최적화 비용, 작은 생태계**다.
- 4개 프레임워크는 우열이 아니라 **결이 다르고, 검색 품질은 프레임워크보다 구성요소(임베딩 등)가 좌우**한다.
