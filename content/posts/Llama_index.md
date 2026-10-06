+++
title: "[LlamaIndex] RAG 저장/질문 코드 분석 - from_documents vs from_vector_store"
date: 2026-09-26
draft: false
categories: ["AI"]
tags: ["LlamaIndex", "RAG", "ChromaDB", "Ollama", "Python", "LLM"]
summary: "예시 코드를 따라 치긴 했는데 context_str은 f-string도 아닌데 어떻게 채워지는지, index는 번호도 아닌데 왜 index인지 감이 안 왔다. 4단계 실습 파일을 하나씩 뜯어보면서 정리했다."
+++

강의에서 LlamaIndex + ChromaDB + Ollama로 소설 Q&A(RAG)를 만들어봤다. 코드는 돌아가는데 "이게 왜 되지?" 싶은 지점이 계속 나왔다. `{context_str}`은 f-string도 아닌데 누가 채워주는지, `index`는 번호가 아닌데 왜 index라고 부르는지 같은 것들이다. 실습 파일 4개를 순서대로 다시 열어보면서 정리했다.

## 목차
- 🧭 RAG 전체 흐름 - 오픈북 시험
- 🪜 4단계로 쪼개서 만든 실습 구조
- 🔌 ChromaVectorStore와 StorageContext
- ⚖️ from_documents vs from_vector_store
- 🧩 context_str / query_str - 빈칸 채우기
- 🌊 response_gen과 source_nodes
- 💬 query_engine vs chat_engine
- 🤔 감이 안 왔던 부분, 다시 짚어보기
- 💡 정리하면

---

## 🧭 RAG 전체 흐름 - 오픈북 시험

RAG는 AI가 소설을 외우고 있는 게 아니라, **시험을 볼 때 책에서 관련 부분을 찾아 펼쳐놓고 답을 쓰는 방식**이다.

```
[저장 코드]  PDF → 읽기 → 청킹 → 임베딩 → ChromaDB 저장   (처음 1번)
[질문 코드]  질문 → 검색(top_k) → 프롬프트에 끼우기 → LLM 답변 (매번)
```

LlamaIndex는 이 과정을 직접 다 짜지 않아도 되게 해주는 도구 상자다. ChromaDB는 벡터를 저장하고 유사도로 검색해주는 창고고, LlamaIndex는 문서 읽기, 청킹, 프롬프트 조립, LLM 호출까지 이어주는 주방장 같은 역할이다.

## 🪜 4단계로 쪼개서 만든 실습 구조

한 번에 다 만들지 않고, 파일을 4개로 나눠서 **하나 확인하고 다음 단계로** 넘어갔다.

| 파일 | 하는 일 | 확인한 것 |
|---|---|---|
| `01_basic.py` | PDF 읽기 → 인덱싱 → 노드 확인 | 청크가 어떻게 쪼개졌나 |
| `02_search.py` | 검색기(retriever) → 질문 엔진 | 유사도 점수, 근거 조각 |
| `03_store.py` | ChromaDB에 영구 저장 | `coll.count()` |
| `04_llm_engine.py` | 저장된 DB 불러와서 질문/답변 | 스트리밍, 대화 |

특히 02에서 **LLM 없이 검색만** 먼저 확인한 게 도움이 됐다. 답변이 이상할 때 검색이 틀린 건지 LLM이 틀린 건지 구분할 수 있기 때문이다.

```python
ret = index.as_retriever(similarity_top_k=3)
nodes = ret.retrieve("문서에서 말하는 핵심 내용이 뭐야?")

for i, node_with_score in enumerate(nodes):
    print(f"[검색결과 {i+1}] 유사도 : {node_with_score.score:.4f}")
    print(node_with_score.node.get_content())
```

<!-- TODO: 내가 직접 돌린 검색 결과(유사도 점수) 붙이기 -->

### 01에서 헷갈렸던 숫자

```python
documents = SimpleDirectoryReader("data", required_exts=[".pdf"]).load_data()
print(f"읽은 chunk 수 : {len(documents)}")   # ← 이름이 틀렸다
```

`SimpleDirectoryReader`는 PDF를 **페이지 단위의 Document**로 읽는다. 그래서 `len(documents)`는 chunk 수가 아니라 **페이지 수**다. 진짜 chunk(노드) 수는 인덱싱 후에 나온다.

```python
index = VectorStoreIndex.from_documents(documents, show_progress=True)
nodes = list(index.docstore.docs.values())
print(f"총 생성된 청크 수 : {len(nodes)}")
```

`chunk_size=300`, `chunk_overlap=30`은 글자 수가 아니라 **토큰 수** 기준이다.

<!-- TODO: 내 PDF의 페이지 수 vs 청크 수 출력 결과 붙이기 -->

## 🔌 ChromaVectorStore와 StorageContext

ChromaDB와 LlamaIndex는 서로 다른 팀이 만든 별개의 도구라서 그냥은 연결이 안 된다. `ChromaVectorStore`가 **LlamaIndex가 알아듣는 형태로 포장해주는 어댑터**다.

```python
client = chromadb.PersistentClient("store")
coll = client.get_or_create_collection("my_collection")

store = ChromaVectorStore(chroma_collection=coll)          # 어댑터
ctx = StorageContext.from_defaults(vector_store=store)     # 저장 위치 설정 상자
```

- `StorageContext`는 "**어디에 저장할지**"를 담은 설정 상자다. 안 넘기면 기본값인 메모리에 저장돼서 프로그램을 끄면 사라진다.
- 다른 DB(Pinecone, FAISS 등)도 같은 패턴의 어댑터가 있어서, DB를 바꿔도 어댑터 부분만 교체하면 된다.

## ⚖️ from_documents vs from_vector_store

| | `from_documents` | `from_vector_store` |
|---|---|---|
| 넘기는 것 | PDF에서 읽은 문서 | ChromaDB 연결(store) |
| 하는 일 | 청킹 → 임베딩 → **저장** | **연결만** (임베딩 X) |
| 속도 | 느림 | 빠름 |
| 언제 | 처음 데이터 넣을 때 | 저장된 걸 검색할 때 |

비유하면 `from_documents`는 **책을 사 와서 분류하고 책장에 꽂는 일**, `from_vector_store`는 **이미 꽂힌 책장을 여는 일**이다.

`from_documents`는 실행할 때마다 데이터가 또 쌓여서 중복 저장이 생길 수 있다. 그래서 03에서는 이렇게 막아뒀다.

```python
if coll.count() == 0:       # 비어 있을 때만 저장
    pages = SimpleDirectoryReader("data", required_exts=[".pdf"]).load_data()
    VectorStoreIndex.from_documents(pages, storage_context=ctx, show_progress=True)
```

<!-- TODO: 저장 후 coll.count() 출력 결과 붙이기 -->

## 🧩 context_str / query_str - 빈칸 채우기

```python
("user", "[소설 본문 발췌]\n{context_str}\n\n[질문]\n{query_str}")
```

f-string이 아닌데 어떻게 알아먹나 싶었는데, 이건 **나중에 `.format()`으로 채우는 빈칸 양식**이었다.

```python
template = "안녕 {name}, 오늘 {day}이야"
print(template.format(name="철수", day="월요일"))
```

코드를 짤 때는 어떤 소설 조각이 검색될지, 사용자가 뭘 물을지 아직 모른다. 그래서 일단 빈칸으로 두고, 실행 중에 LlamaIndex가 채워준다.

1. 사용자가 질문 입력 → `query_str`
2. ChromaDB에서 비슷한 조각 top_k개 검색 → `context_str`
3. 템플릿에 채워서 LLM에 전달

중요한 건 `context_str`, `query_str`이 **LlamaIndex가 약속한 이름**이라는 점이다. 마음대로 `{my_context}`라고 쓰면 채워줄 값이 없다.

## 🌊 response_gen과 source_nodes

OpenAI SDK로 스트리밍할 땐 스트림 객체 자체를 for문에 넣었는데, LlamaIndex는 응답 객체(상자)에서 꺼내 쓴다.

```python
resp = engine.query(question)
for chunk in resp.response_gen:      # 상자 안의 글자 흐름
    print(chunk, end="", flush=True)
```

LlamaIndex는 RAG라서 답변 말고도 돌려줄 게 있기 때문이다. 대표적인 게 **근거 조각**이다.

```python
for i, node in enumerate(resp.source_nodes, start=1):
    page = node.metadata.get("page_label", "?")
    print(f"[{i}] {page}쪽 유사도 : {node.score:.4f}")
    print(node.text[:80])
```

어떤 페이지의 어떤 내용을 보고 답했는지 보여줄 수 있어서 RAG 서비스의 신뢰도를 높일 수 있다.

<!-- TODO: 실제 답변 + 근거 출력 결과 붙이기 -->

## 💬 query_engine vs chat_engine

04에는 두 가지 엔진이 있다.

| | `as_query_engine` | `as_chat_engine` |
|---|---|---|
| 대화 기억 | 없음 (매번 새 질문) | **있음** (이전 대화 참고) |
| 호출 | `engine.query()` | `engine.chat()` / `engine.stream_chat()` |
| 용도 | 단발성 Q&A | 꼬리 질문이 이어지는 챗봇 |

`condense_plus_context` 모드는 이전 대화와 새 질문을 합쳐서 **혼자서도 이해되는 질문으로 바꾼 뒤(condense)** 검색하고(context) 답한다. "그 사람은 왜 그랬어?" 같은 질문도 앞 대화를 보고 해석할 수 있다.

> ⚠️ 확인해볼 점: chat_engine의 `context_prompt`에는 보통 `{context_str}`만 채워지고 질문은 대화 메시지로 따로 들어간다고 한다. 내가 쓴 `{query_str}`가 채워지지 않고 글자 그대로 남아 있을 수도 있어서, 실제 프롬프트를 출력해서 확인해볼 예정이다.

<!-- TODO: 확인 결과 적기 -->

## 🤔 감이 안 왔던 부분, 다시 짚어보기

**LlamaIndex의 index는 번호가 아니다.** 리스트의 `a[0]` 같은 위치 번호가 아니라, "**빨리 찾기 위해 미리 정리해둔 검색 가능한 문서 구조 전체**"를 담은 객체다. 뿌리는 DB 인덱스(색인)와 같지만 키워드가 아니라 벡터(의미) 기반이다.

**메타데이터가 AI로 자동 할당되는 건 아니다.** 기본으로 붙는 건 `page_label`, `file_name` 같은 파일 정보다. 키워드나 요약 같은 AI 기반 메타데이터는 Extractor를 따로 붙여야 생긴다.

**메서드를 고르는 감은 흐름에서 온다.** 메서드부터 외우는 대신 "내가 지금 몇 번 단계지?"를 먼저 보면 된다.

```
읽기 → 쪼개기 → 임베딩 → 저장 → 검색 → LLM 답변
```

이름도 힌트다. `from_~`는 ~로부터 만들기, `as_~`는 ~로 변환, `get_or_create`는 있으면 가져오고 없으면 생성이다.

## 💡 정리하면

| 하고 싶은 일 | 쓰는 것 |
|---|---|
| 모델 등록 | `Settings.llm`, `Settings.embed_model` |
| 조각 크기 설정 | `Settings.chunk_size`, `chunk_overlap` (토큰 기준) |
| 파일 읽기 | `SimpleDirectoryReader(...).load_data()` |
| DB 연결 | `ChromaVectorStore`, `StorageContext.from_defaults` |
| 처음 저장 | `VectorStoreIndex.from_documents(..., storage_context=ctx)` |
| 저장된 DB 연결 | `VectorStoreIndex.from_vector_store(store)` |
| 검색만 확인 | `index.as_retriever().retrieve()` |
| 단발 질문 | `index.as_query_engine()` → `engine.query()` |
| 대화형 질문 | `index.as_chat_engine()` → `engine.stream_chat()` |
| 스트리밍 / 근거 | `resp.response_gen` / `resp.source_nodes` |

- 저장 코드와 질문 코드는 **짝**이다. 저장은 한 번만, 이후엔 불러와서 쓴다.
- 임베딩 모델을 바꾸면 기존 벡터와 호환이 안 되니 **DB를 다시 만들어야** 한다.
- 답변이 이상하면 **검색부터 따로 확인**한다.
- 작게 시작해서 한 단계씩 확인하고 붙여나가는 방식이 에러 원인을 좁히기 가장 좋았다.