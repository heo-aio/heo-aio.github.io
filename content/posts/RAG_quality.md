+++
title = "[RAG] RAG를 쓸 때와 안 쓸 때, 답변은 얼마나 다를까? (LangChain 복습)"
date = 2026-10-15
draft = false
categories = ["dev"]
tags = ["RAG", "LangChain", "Ollama", "Chroma", "RunnablePassthrough", "RunnableParallel", "환각", "Python"]
summary = "같은 질문을 RAG 없이, 그리고 RAG와 함께 던져서 답변을 비교했다. RAG 없이는 그럴듯한 숫자를 지어내고, RAG를 붙이니 PDF 근거로 답했다. 이 차이를 만드는 qa_chain의 딕셔너리 구조(RunnableParallel, RunnablePassthrough)까지 정리했다."
+++

RAG가 좋다는 말은 많이 들었는데, **정말 답변이 달라지는지** 직접 확인해 본 적은 없었다. 이번 실습에서는 키오스크 이용실태 조사 PDF를 놓고, 같은 질문을 RAG 없이/RAG와 함께 던져서 비교했다. 결과는 꽤 극적이었고, 그 차이를 만든 `qa_chain`의 구조도 함께 정리했다.

## 목차
- 🧭 실습 개요
- ⚙️ 준비: 문서 로드부터 벡터DB까지
- 🔗 두 개의 체인 만들기
- 📊 답변 비교: RAG 없이 vs RAG와 함께
- 🔍 qa_chain의 딕셔너리, 뭘 하는 걸까?
- 🛠️ 헷갈렸던 부분
- 💡 정리하면

---

## 🧭 실습 개요

```
PDF 로드 → 쪼개기(chunk) → 임베딩 → 벡터DB(Chroma) 저장 → retriever
```

이 준비물을 만든 뒤, 체인을 **두 개** 만들어서 같은 질문을 던졌다.

| 체인 | 구성 | 근거 문서 |
|---|---|---|
| `chain` | 프롬프트 → LLM → 파서 | ❌ 없음 |
| `qa_chain` | 검색 → 프롬프트 → LLM → 파서 | ✅ PDF 검색 결과 |

질문은 둘 다 `"키오스크 관련 설문조사 결과를 알려줘"`이고, 모델은 로컬에서 돌리는 Ollama의 `llama3.1`을 썼다.

## ⚙️ 준비: 문서 로드부터 벡터DB까지

```python
llm = ChatOllama(model="llama3.1")
embeddings = OllamaEmbeddings(model="llama3.1")

loader = PyPDFLoader("키오스크(무인정보단말기) 이용실태 조사.pdf")
docs = loader.load()   # 페이지 단위 Document 리스트
```

문서를 그대로 LLM에 넣으면 길어서 비효율적이라, 먼저 **chunk로 쪼갠다.**

```python
text_splitter = RecursiveCharacterTextSplitter(chunk_size=1000, chunk_overlap=200)
splited_docs = text_splitter.split_documents(docs)
```

| 옵션 | 의미 |
|---|---|
| `chunk_size=1000` | 조각 하나의 최대 길이 (글자 수 기준) |
| `chunk_overlap=200` | 인접한 조각끼리 200자씩 겹치게 해서 문맥이 끊기지 않도록 함 |

쪼갠 뒤에는 `profile_docs()`로 **페이지 수와 길이를 찍어서** 확인했다. 쪼개기 전후의 개수를 비교해 보면 쪼개기가 제대로 됐는지 눈으로 볼 수 있다.

```python
vectorstore = Chroma.from_documents(documents=splited_docs, embedding=embeddings)
db_retriever = vectorstore.as_retriever()
```

- `from_documents`: 조각들을 임베딩(숫자 벡터)으로 바꿔서 Chroma에 저장
- `as_retriever()`: 질문과 비슷한 조각을 찾아주는 **검색기**로 변환

## 🔗 두 개의 체인 만들기

### RAG 없는 체인

```python
prompt = ChatPromptTemplate.from_messages([
    ("system", "당신은 마케터를 위한 친절한 지원 챗봇입니다."),
    ("human", "{question}."),
])
chain = prompt | llm | StrOutputParser()
```

### RAG 체인

```python
prompt_with_context = ChatPromptTemplate.from_messages([
    ("system", "당신은 마케터를 위한 친절한 지원 챗봇입니다. 사용자가 입력하는 정보를 바탕으로 질문에 답하세요."),
    ("human", "정보: {context}.\n{question}."),
])

qa_chain = (
    {"context": db_retriever | get_retrieved_text, "question": RunnablePassthrough()}
    | prompt_with_context
    | llm
    | StrOutputParser()
)
```

두 체인의 차이는 딱 두 가지다. 프롬프트에 `{context}` 빈칸이 있고, 그 앞에 **빈칸을 채울 재료를 만드는 딕셔너리**가 붙어 있다.

<!-- TODO: 코드 실행 화면 캡처 붙이기 -->

## 📊 답변 비교: RAG 없이 vs RAG와 함께

### ❌ RAG 없이 (`chain`)

```
키오스크 관련 설문조사는 다음과 같은 결과가 나왔습니다.

1. 고객 만족도: 7/10 (...)
2. 사용률: 60% (키오스크는 매출의 30% 이상을 차지하는 중간 정도의 사용률을 보였다.)
3. 고객 인식: 4/5 (...)
```

### ✅ RAG와 함께 (`qa_chain`)

```
키오스크(무인정보단말기) 이용 실태 조사 결과는 다음과 같습니다.

* 233명의 사람 중, 206명이(88.4%) 키오스크 이용을 도중에 중단한 경험이 있다고 응답했습니다.
* 60대 이상 고령자는 52명(25.2%)으로, ...
* 이용 중단 유형은 다음과 같습니다. (상품 없음 / 안내 부족 / 기기 오류 / 이용법 복잡 / 뒷사람 눈치 / ...)
* 키오스크의 화면 높이는 기준선(1,220mm) 아래로 설치되어야 하며, ...
```

### 차이점

| 비교 | RAG 없이 | RAG와 함께 |
|---|---|---|
| 근거 | 없음 (모델의 일반 지식) | 내가 넣은 PDF |
| 수치 | `7/10`, `60%`, `4/5` 같은 **출처 없는 숫자** | `233명`, `206명(88.4%)` 등 **문서에 있는 구체적 수치** |
| 설문 대상 | 어떤 조사인지 불명확 | 이용 중단 경험, 연령대별 분포 등 **실제 조사 내용** |
| 신뢰도 | 그럴듯하지만 확인할 길이 없음 | 원문과 대조 가능 |

핵심은 이 부분이다.

> RAG 없이 나온 답변은 **형식은 설문 결과처럼 보이지만 내용은 지어낸 것**이다. 문서를 본 적이 없으니, 질문에서 "설문조사 결과"라는 말만 보고 그럴듯한 숫자를 만들어냈다. 이게 바로 **환각(hallucination)** 이다.

더 무서운 점은, 답변이 **자신감 있게 구조화돼서 나온다**는 것이다. 이 PDF를 모르는 사람이 보면 진짜 결과로 착각할 수 있다. RAG를 쓰는 이유가 "답변이 조금 나아져서"가 아니라 **"근거 없는 답변을 막기 위해서"**라는 걸 체감했다.

참고로 두 번째 답변의 숫자가 실제로 맞는지는 원문과 한 번씩 대조해 볼 필요가 있다. RAG를 써도 LLM이 검색된 내용을 잘못 요약할 수 있기 때문이다.

<!-- TODO: 두 답변 비교 캡처 붙이기 -->

## 🔍 qa_chain의 딕셔너리, 뭘 하는 걸까?

```python
{"context": db_retriever | get_retrieved_text, "question": RunnablePassthrough()}
```

이 부분이 가장 헷갈렸다. 한 줄로 요약하면 **프롬프트의 빈칸 두 개를 채울 재료를 준비하는 단계**다.

프롬프트에는 빈칸이 두 개인데, 사용자가 넣는 입력은 질문 하나다.

```python
("human", "정보: {context}.\n{question}.")
```

그래서 질문 하나가 **복사되어 두 갈래로** 간다.

```
"키오스크 설문 결과 알려줘"
        ├──▶ context 갈래: [검색] → [문서 조각들을 하나의 글로 합치기]
        └──▶ question 갈래: [그대로 통과]
        ▼
{"context": "찾은 근거 글...", "question": "키오스크 설문 결과 알려줘"}
        ▼
프롬프트 빈칸에 들어감
```

### 부품별 정리

| 부품 | 하는 일 |
|---|---|
| `db_retriever` | 질문과 비슷한 문서 조각 검색 → `Document` 리스트 |
| `get_retrieved_text` | 리스트에서 `page_content`만 꺼내 문자열 하나로 합침 |
| `RunnablePassthrough()` | 받은 입력을 **그대로 통과** (`lambda x: x`와 같음) |
| `{ ... }` 딕셔너리 | LangChain이 자동으로 `RunnableParallel`로 바꿔서 갈래를 **동시에** 실행하고, 결과를 같은 key의 딕셔너리로 모음 |

### RunnablePassthrough vs RunnableParallel

| | `RunnablePassthrough()` | `RunnableParallel({...})` |
|---|---|---|
| 정체 | 부품 하나 | 부품 여러 개를 묶는 틀 |
| 하는 일 | 입력을 그대로 통과 | 같은 입력으로 여러 갈래를 동시에 실행 |
| 출력 | 입력 그대로 | key별 결과가 담긴 딕셔너리 |

`RunnablePassthrough`는 `RunnableParallel`의 **갈래 하나로 들어가는 부품**이다.

### question 갈래가 왜 필요할까?

프롬프트에는 **근거 자료(`context`)와 질문(`question`)이 둘 다** 보여야 한다. context 갈래는 검색과 가공이 필요하지만, question 갈래는 **원문을 그대로 빈칸에 넣으면 끝**이라 건드릴 게 없다. 그래서 "그대로 통과"를 뜻하는 `RunnablePassthrough()`를 쓴다.

직접 확인하려면 체인에서 이 부분만 떼어서 돌려보면 된다.

```python
from langchain_core.runnables import RunnableParallel

step = {"context": db_retriever | get_retrieved_text, "question": RunnablePassthrough()}
out = RunnableParallel(step).invoke("키오스크 설문 결과 알려줘")
print(out.keys())             # dict_keys(['context', 'question'])
print(out["context"][:300])   # 검색된 PDF 내용
```

프롬프트에 **실제로 뭐가 들어가는지** 볼 수 있어서, 답변이 이상할 때 "검색 문제인지, LLM 문제인지"를 가르는 디버깅 방법이 된다.

## 🛠️ 헷갈렸던 부분

**1) 입력이 context로 갈지 question으로 갈지 모르는 건가?**
→ 아니다. 입력은 **모든 갈래에 복사**돼서 들어간다. 각 갈래가 받은 질문으로 무엇을 할지 스스로 정해야 하는데, question 갈래는 "그대로 써라"를 알려 줄 부품(`RunnablePassthrough`)이 필요한 것이다.

**2) `|`는 뭔가?**
→ 파이프다. 앞의 출력을 뒤의 입력으로 넘긴다. `프롬프트 | llm | 파서`처럼 이어 붙여서 체인을 만든다.

**3) 지난번에 배운 `create_retrieval_chain`과는 뭐가 다른가?**
→ 같은 일(검색 → 빈칸 채우기 → LLM)을 **기성품으로 묶은 것**이 `create_retrieval_chain`이고, 이번 코드는 `|`로 직접 조립한 버전이다. 직접 조립하면 안에서 무슨 일이 일어나는지 더 잘 보인다.

## 💡 정리하면

| 개념 | 한 줄 요약 |
|---|---|
| 환각 | 근거 없이 그럴듯한 내용을 지어내는 현상 |
| RAG | 문서를 먼저 찾아 프롬프트에 넣고, 거기서 답하게 하는 방식 |
| chunk | 문서를 쪼갠 조각 (`chunk_overlap`으로 문맥 유지) |
| retriever | 질문과 비슷한 조각을 찾아주는 검색기 |
| `RunnablePassthrough()` | 입력을 그대로 통과시키는 부품 |
| `RunnableParallel` | 같은 입력으로 여러 갈래를 동시에 실행해 딕셔너리로 모음 |
| `\|` (파이프) | 앞의 결과를 뒤로 넘겨 체인으로 연결 |

- RAG의 가치는 "답변이 조금 좋아진다"가 아니라 **"근거 없는 답변을 막고, 확인 가능한 답변을 만든다"**에 있다.
- RAG를 써도 답변이 틀릴 수 있어서, **원문과 대조하는 습관**이 필요하다.
- 다음엔 더 해볼 것들이다.
  - [ ] 질문 몇 개로 평가셋을 만들어서 RAG 유무를 비교하기 (감이 아니라 숫자로)
  - [ ] chunk 크기를 바꿔서 답변이 어떻게 달라지는지 확인하기
  - [ ] 문서에 없는 질문을 했을 때 "모른다"고 답하는지 확인하기
  - [ ] 임베딩 모델을 전용 모델로 바꿔서 검색 품질 비교하기