+++
title = "[RAG] Haystack 정리: connect로 이어 붙이고 YAML로 저장하는 파이프라인"
date = 2026-10-28
draft = false
categories = ["dev"]
tags = ["RAG", "Haystack", "Pipeline", "Component", "하이브리드검색", "리랭킹", "YAML", "Python"]
summary = "RAG 프레임워크 비교 수업의 두 번째, Haystack을 정리했다. 컴포넌트를 connect로 명시적으로 연결하면 뭐가 좋은지, BM25와 임베딩 검색을 합치는 하이브리드 구성, 파이프라인을 YAML로 저장하고 복원하는 방법을 정리했다."
+++

RAG 프레임워크 비교 시리즈 두 번째, **Haystack**이다. 수업에서 "운영 친화적"이라는 말이 계속 나왔는데, 처음엔 무슨 뜻인지 와닿지 않았다. 코드를 보고 나니 이해가 됐다.

## 목차
- 🧭 Haystack은 어떤 프레임워크?
- 🧩 핵심 개념: Component · Pipeline · DocumentStore
- ⚙️ 같은 코퍼스로 RAG 구현하기
- 🔀 하이브리드 검색 + 리랭킹
- 💾 YAML로 저장하고 복원하기
- 🛠️ 헷갈렸던 부분
- 💡 정리하면

---

## 🧭 Haystack은 어떤 프레임워크?

deepset에서 만든 프레임워크로, **명시적인 컴포넌트 파이프라인으로 검색/QA를 조립**한다. 특징은 세 가지다.

- 각 단계를 **컴포넌트**로 만든다.
- 컴포넌트의 출력 포트와 입력 포트를 **`connect`로 직접 연결**한다.
- 파이프라인을 **YAML로 직렬화**해서 저장·공유·서빙할 수 있다.

수업에서 세 프레임워크를 이렇게 한 줄로 비교했다.

| 프레임워크 | 한 줄 정체성 |
|---|---|
| LangChain | 유연한 조립 |
| LlamaIndex | 빠른 인덱싱 |
| Haystack | 명시적·운영 친화 파이프라인 |

## 🧩 핵심 개념: Component · Pipeline · DocumentStore

| 개념 | 설명 |
|---|---|
| `Component` | 입력/출력 포트가 정의된 처리 단위. `@component`로 직접 만들 수도 있음 |
| `Pipeline` | `add_component`와 `connect`로 만든 실행 그래프. 분기·병합 가능 |
| `DocumentStore` | 문서 저장소. InMemory부터 Elasticsearch, Qdrant, pgvector까지 교체 가능 |

기본 RAG 흐름은 이렇게 생겼다.

```
DocumentStore → Retriever → PromptBuilder → Generator(LLM)
```

문서를 임베딩해서 저장하는 부분은 이렇다.

```python
store = InMemoryDocumentStore()
docs = [Document(content=d["text"], meta={"topic": d["topic"]}) for d in SAMPLE_DOCS]
doc_emb = SentenceTransformersDocumentEmbedder(model=EMB_MODEL); doc_emb.warm_up()
store.write_documents(doc_emb.run(docs)["documents"])   # 임베딩 후 저장
```

LlamaIndex는 `from_documents` 한 줄이었는데, Haystack은 **임베딩 → 저장을 내가 직접 한 단계씩** 한다. 코드는 길어지지만 어떤 일이 벌어지는지 다 보인다.

## ⚙️ 같은 코퍼스로 RAG 구현하기

```python
pipe.add_component('retriever', InMemoryEmbeddingRetriever(store, top_k=3))
pipe.add_component('llm', OpenAIGenerator(api_base_url=…, model=MLAPI_MODEL))

pipe.connect('text_embedder.embedding', 'retriever.query_embedding')
pipe.connect('retriever.documents', 'prompt.documents')
pipe.connect('prompt.prompt', 'llm.prompt')
```

`connect`의 인자는 **`"컴포넌트이름.출력포트"` → `"컴포넌트이름.입력포트"`** 형태다. 세 줄만 읽어도 흐름이 보인다.

```
text_embedder → retriever → prompt → llm
```

실행 결과는 LlamaIndex와 같은 문서를 찾았다.

```
출처(retriever): ['파인튜닝', 'RAG', '평가']
답변: RAG는 검색해 답하고, 파인튜닝은 가중치를 추가학습해 작업 성능↑.
```

## 🔀 하이브리드 검색 + 리랭킹

수업에서 하이브리드 검색과 리랭킹을 `connect` 몇 줄로 만들었다.

- **하이브리드**: BM25(키워드 검색) + 임베딩(의미 검색) 결과를 `DocumentJoiner`(RRF)로 합친다.
- **리랭킹**: `TransformersSimilarityRanker`를 joiner 뒤에 연결한다.

```python
p = Pipeline()
p.add_component("te", SentenceTransformersTextEmbedder(model=EMB_MODEL))
p.add_component("emb_ret", InMemoryEmbeddingRetriever(store, top_k=top_k))   # dense
p.add_component("bm25_ret", InMemoryBM25Retriever(store, top_k=top_k))       # sparse
p.add_component("joiner", DocumentJoiner())                                  # 두 결과 병합
p.add_component("ranker", TransformersSimilarityRanker(model="BAAI/bge-reranker-v2-m3", top_k=top_r))

p.connect("te.embedding", "emb_ret.query_embedding")
p.connect("emb_ret.documents", "joiner.documents")     # dense → joiner
p.connect("bm25_ret.documents", "joiner.documents")    # sparse → joiner
p.connect("joiner.documents", "ranker.documents")      # joiner → ranker (핵심)
```

그림으로 그려 보면 이렇다.

```
        ┌→ emb_ret (의미 검색) ─┐
te ─────┤                       ├→ joiner → ranker
        └→ (질문 그대로) bm25_ret (키워드 검색) ─┘
```

용어를 정리하면 이렇다.

| 용어 | 뜻 |
|---|---|
| dense 검색 | 임베딩 벡터로 **의미**가 비슷한 문서 찾기 |
| sparse 검색(BM25) | **키워드**가 겹치는 문서 찾기 |
| RRF | 두 검색 결과의 **순위**를 기반으로 합치는 방법 |

수업의 한 줄 요약이 기억에 남는다. **"모듈 끼우고 빼기"가 Haystack에서는 `connect` 한 줄 추가/삭제로 시각화된다.** 리랭커를 빼고 싶으면 그 줄과 컴포넌트만 지우면 된다.

## 💾 YAML로 저장하고 복원하기

만든 파이프라인을 파일로 저장할 수 있다.

```python
hp = build_hybrid_ranker_pipeline()

YAML_PATH = "haystack_pipeline.yaml"
with open(YAML_PATH, "w", encoding="utf-8") as f:
    hp.dump(f)    # 파이프라인 → YAML 파일 생성

# 복원: YAML 파일만으로 동일 파이프라인 재구성
with open(YAML_PATH, encoding="utf-8") as f:
    reloaded = Pipeline.load(f)
```

YAML 파일 안에는 `components:` 아래에 각 컴포넌트의 종류와 설정값(`init_parameters`)이 들어 있고, 어떤 포트끼리 연결됐는지도 함께 저장된다. 복원 후 컴포넌트 목록을 찍어 보니 `['bm25_ret', 'emb_ret', 'joiner', 'ranker', 'te']`로 원래 구성이 그대로 돌아왔다.

이게 왜 **운영에 좋은지**를 내 말로 정리하면 이렇다.

| 장점 | 이유 |
|---|---|
| 팀 공유 | 파이프라인을 YAML 파일 하나로 전달 가능 |
| 배포 | 서버에서는 파일을 읽어서 실행 (수업에서는 hayhooks REST로 서빙) |
| 버전 관리 | Git에 YAML을 올려서 "top_k를 바꾼 버전" 같은 이력 관리 |
| 가시성 | 코드를 몰라도 YAML만 보면 구조를 읽을 수 있음 |

## 🛠️ 헷갈렸던 부분

**1) `connect`가 왜 장점인가? 그냥 귀찮아 보이는데?**
→ 맞다, 약점이기도 하다. 수업에서도 "포트 연결 보일러플레이트가 많다"고 했다. 하지만 데이터가 어디서 어디로 가는지가 코드에 그대로 드러나서, 문제가 생겼을 때 **어느 단계가 원인인지 추적하기 쉽다.**

**2) LangGraph와 비슷해 보이는데?**
→ 둘 다 "노드를 만들고 선으로 잇는" 구조라 코드가 비슷하다. 다만 Haystack은 검색 중심의 정해진 흐름을 운영하는 쪽이고, 동적으로 분기·반복하는 에이전트는 LangGraph가 더 강하다.

**3) 하이브리드에서 BM25는 한국어에 잘 되나?**
→ 공백 기준 토큰화를 쓰면 "RAG와"와 "RAG"가 다른 단어로 취급된다. 한국어에서는 형태소 분석기(Kiwi, Mecab 등)를 붙여야 한다는 게 수업의 공통 교훈이었다.

## 💡 정리하면

| 개념 | 한 줄 요약 |
|---|---|
| `Component` | 입력/출력 포트가 있는 처리 단위 |
| `Pipeline` | `add_component` + `connect`로 만든 실행 그래프 |
| `DocumentStore` | 문서 저장소 (InMemory ~ Elasticsearch, Qdrant, pgvector) |
| `DocumentJoiner(RRF)` | BM25와 임베딩 검색 결과를 순위로 합침 |
| `dump()` / `load()` | 파이프라인을 YAML로 저장 / 복원 |

- 강점은 **가시성, 운영/서빙, 컴포넌트 교체 용이성**이다.
- 약점은 **연결 코드가 길고, 동적인 에이전트에는 LangGraph보다 약하다**는 점이다.
- 검색 중심 서비스를 **명시적으로 관리·서빙·협업**해야 할 때 선택한다.
