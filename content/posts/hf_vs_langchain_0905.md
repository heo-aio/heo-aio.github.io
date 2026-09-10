+++
title = "[LLM] Hugging Face Transformers vs LangChain - 코드분석"
date = 2026-09-08
draft = false
tags = ["HuggingFace", "LangChain", "LLM", "AI"]
categories = ["dev"]
math = false
+++

지금까지는 `AutoModelForCausalLM`이랑 `generate()`로만 로컬 LLM을 돌려봤는데, 강의에서 같은 걸 LangChain으로 짠 코드를 보여주셨다. 근데 코드 스타일이 너무 달라서 처음엔 "이거 완전 다른 걸 하는 건가" 싶었다. 알고 보니 목적 자체가 다른 도구였고, 왜 둘 다 배우는지도 같이 정리해봤다.

**목차**

1. [🔍 같은 LLM 호출인데 코드가 왜 이렇게 다를까](#-같은-llm-호출인데-코드가-왜-이렇게-다를까)
2. [🧩 Transformers - 과정을 다 내가 직접 쓴다](#-transformers---과정을-다-내가-직접-쓴다)
3. [🔗 LangChain - 조립만 하면 나머지는 알아서](#-langchain---조립만-하면-나머지는-알아서)
4. [🤔 그래서 뭐가 다른 건가 - 추상화 수준의 차이](#-그래서-뭐가-다른-건가---추상화-수준의-차이)
5. [🛠️ 그럼 언제 뭘 써야 하나](#-그럼-언제-뭘-써야-하나)
6. [🚀 실무에서는 이 둘이 어떻게 이어지는가](#-실무에서는-이-둘이-어떻게-이어지는가)
7. [💡 정리하면](#-정리하면)

---

## 🔍 같은 LLM 호출인데 코드가 왜 이렇게 다를까

강의에서 같은 목표(로컬 LLM에 질문하고 답 받기)를 두 가지 방식으로 보여주셨다.

**Transformers 방식**
```python
from transformers import AutoModelForCausalLM, AutoTokenizer

tokenizer = AutoTokenizer.from_pretrained('mistralai/Mistral-7B-Instruct-v0.2')
model = AutoModelForCausalLM.from_pretrained('mistralai/Mistral-7B-Instruct-v0.2', device_map='auto')

prompt = "Explain baseball rules simply."
inputs = tokenizer(prompt, return_tensors='pt').to('cuda')
outputs = model.generate(**inputs, max_new_tokens=200)
result = tokenizer.decode(outputs[0], skip_special_tokens=True)
print(result)
```

**LangChain 방식**
```python
from langchain_ollama import ChatOllama
from langchain_core.prompts import ChatPromptTemplate
from langchain_core.output_parsers import StrOutputParser

model = ChatOllama(model="exaone3.5:2.4b")
prompt = ChatPromptTemplate.from_messages([
    ("system", "Easily explain information, like a baseball expert with examples."),
    ("user", "{topic}")
])
chain = prompt | model | StrOutputParser()

result = chain.invoke({"topic": query})
print(result)
```

둘 다 "질문 넣고 답 받는다"는 결과는 똑같은데, LangChain 쪽엔 `tokenizer`도 없고 `return_tensors`, `max_new_tokens` 같은 옵션도 없다. 처음엔 "LangChain이 더 쉬운 버전인가?" 싶었는데, 정확히는 **추상화 수준이 다른 도구**라는 걸 하나씩 뜯어보면서 이해했다.

강의 자료로 받은 비교 이미지를 보면 이 차이가 한눈에 들어온다.

![Hugging Face Transformers는 토크나이저 로드, 수동 토큰화, 모델 추론, 수동 디코딩까지 단계를 직접 명시하는 저수준·명시적 구조인 반면, LangChain은 모델을 컴포넌트로 정의하고 프롬프트 템플릿과 체인으로 조립한 뒤 invoke만 호출하면 되는 고수준·추상화된 구조다](/images/HF/hf_vs_langchain_comparison.png)

---

## 🧩 Transformers - 과정을 다 내가 직접 쓴다

지금까지 배운 개념들을 그대로 대입해보면, Transformers 코드는 이 4단계를 전부 명시적으로 쓰고 있었다.

1. 토크나이저와 모델을 각각 따로 불러옴
2. 프롬프트를 직접 토큰화 (`tokenizer(prompt, return_tensors='pt')`)
3. 모델 추론 (`model.generate(**inputs, max_new_tokens=200)`)
4. 결과를 직접 디코딩 (`tokenizer.decode(...)`)

저번에 정리했던 "`generate()`는 `forward()`를 반복하는 자기회귀 루프"라는 개념이 여기 그대로 쓰이고 있었다. `return_tensors='pt'`(파이토치 텐서로 변환), `max_new_tokens`(몇 토큰까지 생성할지) 같은 옵션도 전부 내가 직접 지정해야 했다.

> ✅ **핵심**: Transformers는 "모델을 어떻게 동작시킬지"를 내가 한 단계씩 직접 제어하는 저수준(low-level) 방식이다. 대신 그만큼 모델 내부(헤드 구조, 생성 파라미터 등)를 자유롭게 다룰 수 있다.

---

## 🔗 LangChain - 조립만 하면 나머지는 알아서

반면 LangChain 코드에는 토큰화나 디코딩 코드가 아예 안 보였다.

```python
chain = prompt | model | StrOutputParser()
result = chain.invoke({"topic": query})
```

`prompt | model | StrOutputParser()` — 이 파이프(`|`) 문법이 처음엔 낯설었는데, "프롬프트를 만들고 → 모델에 넣고 → 결과를 문자열로 파싱한다"는 흐름을 **연결(체인)**해놓은 거였다. `chain.invoke()` 한 번 호출하면 이 파이프라인 전체가 내부에서 순서대로 실행되고, 그 안에서 토큰화/추론/디코딩이 다 알아서 처리된다.

`ChatPromptTemplate.from_messages([...])`로 system/user 역할을 나눠서 프롬프트를 템플릿화하는 것도 인상적이었다. 저번 `04_chat.py`에서 `apply_chat_template`으로 직접 만들었던 대화 형식(`{"role": "user", "content": ...}`)이, LangChain에서는 아예 템플릿 객체로 관리되는 느낌이었다.

> ✅ **핵심**: LangChain은 "모델을 어떻게 동작시킬지"의 디테일을 라이브러리 내부로 감추고, 대신 프롬프트·모델·후처리를 부품처럼 연결(`|`)해서 조립하는 데 집중하는 고수준(high-level) 방식이다.

---

## 🤔 그래서 뭐가 다른 건가 - 추상화 수준의 차이

강의 설명을 듣고 나서 정리가 된 건, 이 둘이 "쉬움 vs 어려움"이 아니라 **"뭘 하기 위한 도구인가"** 자체가 다르다는 거였다.

| | Transformers | LangChain |
|---|---|---|
| 추상화 수준 | 낮음 (직접 제어) | 높음 (조립 중심) |
| 잘하는 것 | 모델 구조 자체를 다루는 것 (파인튜닝, 헤드 교체) | 프롬프트 관리, 대화 기록 유지, RAG 등 외부 도구 연결 |
| 토큰화/디코딩 | 직접 작성 | 내부에서 자동 처리 |
| 비유 | 자동차 엔진을 직접 조립 | 이미 조립된 부품(엔진, 바퀴)을 가져다 차를 완성 |

즉 Transformers는 **모델 자체를 만지고 바꾸는 데** 최적화돼 있고, LangChain은 **이미 만들어진 모델을 가져다가 서비스로 엮는 데** 최적화돼 있는 거였다. 둘 다 "LLM을 쓴다"는 목적은 같지만, 어느 층위를 다루느냐가 다른 거다.

---

## 🛠️ 그럼 언제 뭘 써야 하나

강의 내용 기준으로 정리하면 이렇게 나뉘는 것 같다.

**Transformers가 필요한 상황**
- 베이스 모델을 내 데이터로 파인튜닝해야 할 때 (LoRA, 커스텀 헤드 등)
- 모델 내부 구조(hidden_states, logits 등)를 직접 들여다봐야 할 때
- 생성 파라미터(temperature, top_p 등)를 세밀하게 실험해야 할 때

**LangChain이 필요한 상황**
- 이미 만들어진 모델(또는 API)을 가져다 서비스 로직을 짤 때
- 대화 기록을 유지하거나, 외부 문서를 참고하게(RAG) 만들 때
- 프롬프트를 체계적으로 관리하고 재사용해야 할 때

지금까지 실습한 `AutoModelForCausalLM` 코드들은 전부 전자(모델 자체를 다루는 것)였고, 나중에 서비스를 만들 단계가 되면 후자(LangChain류)가 필요해진다는 걸 이제야 구분이 됐다.

---

## 🚀 실무에서는 이 둘이 어떻게 이어지는가

강의에서 제일 도움이 됐던 부분이 이거였다. 이 두 기술이 **경쟁 관계가 아니라 이어지는 파이프라인**이라는 설명이었다.

![Transformers로 베이스 모델을 파인튜닝하고, Ollama나 vLLM 같은 추론 엔진에 마운트해서 API로 서빙한 뒤, LangChain으로 그 API에 접근해 RAG나 대화 기록 같은 기능을 조립해 서비스 앱을 완성하는 3단계 흐름](/images/HF/hf_langchain_workflow.svg)

1. **Transformers**로 베이스 모델(EXAONE, Gemma 등)을 내 데이터로 파인튜닝
2. 그 결과물을 **Ollama, vLLM** 같은 추론 엔진에 마운트해서 API 형태로 배포
3. **LangChain**의 `ChatOllama` 같은 인터페이스로 그 API에 접근해서, RAG나 대화 맥락 유지 같은 기능을 엮어 서비스 앱 완성

이 흐름을 보고 나니 지금 배우고 있는 조각들(토크나이저, 헤드, generate, 파인튜닝)이 전체 그림에서 어디쯤 위치하는지 감이 잡혔다. 작업 성격에 따라 이 단계 중 일부는 생략되거나 다른 팀이 맡을 수도 있다고 하셨는데, 그래도 "이 기술이 왜 필요한지"를 전체 흐름 안에서 이해하고 나니 훨씬 덜 막막했다.

---

## 💡 정리하면

두 코드를 나란히 보고 "왜 이렇게 다르지"라고 궁금해하지 않았으면 그냥 "LangChain이 더 편한 버전이네" 정도로만 넘어갔을 것 같다. 근데 뜯어보니 Transformers는 모델 자체를 다루는 도구, LangChain은 모델을 가져다 서비스로 엮는 도구로 목적이 완전히 다르다는 걸 알게 됐다. 지금 배우고 있는 Transformers 기초가 나중에 LangChain 단계로 넘어갈 때도 "모델이 내부적으로 뭘 하고 있는지"를 이해하는 바탕이 될 것 같다.

**오늘의 핵심 4가지**

- 🧩 Transformers는 토큰화-추론-디코딩을 직접 제어하는 저수준 방식이다
- 🔗 LangChain은 프롬프트-모델-파서를 `|`로 조립하면 내부에서 알아서 처리해주는 고수준 방식이다
- 🛠️ Transformers는 모델 자체(파인튜닝, 구조 변경)에, LangChain은 서비스 조립(RAG, 대화기록)에 강하다
- 🚀 실무에서는 Transformers(파인튜닝) → Ollama/vLLM(배포) → LangChain(서비스 조립) 순으로 이어질 수 있다
