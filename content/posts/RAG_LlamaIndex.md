+++
title = "[RAG] LlamaIndex 정리: 문서를 넣고 바로 질의하는 가장 빠른 길"
date = 2026-10-28
draft = false
categories = ["dev"]
tags = ["RAG", "LlamaIndex", "VectorStoreIndex", "QueryEngine", "리랭킹", "SummaryIndex", "Python"]
summary = "RAG 프레임워크 비교 수업의 첫 번째, LlamaIndex를 정리했다. from_documents 한 줄로 청킹·임베딩·인덱싱이 끝나는 이유, 리랭커를 한 줄로 붙이는 방법, 그리고 '전체 요약'에는 왜 벡터 검색이 부실한지까지 정리했다."
+++

RAG 프레임워크 4가지(LlamaIndex, Haystack, RAGFlow, DSPy)를 같은 코퍼스로 비교하는 수업을 들었다. 내용이 많아서 프레임워크별로 나눠 정리하기로 했고, 이번 글은 그 첫 번째인 **LlamaIndex**다.

## 목차
- 🧭 LlamaIndex는 어떤 프레임워크?
- 🔄 핵심 흐름: Document → Node → Index → QueryEngine
- ⚙️ 같은 코퍼스로 RAG 구현하기
- 🎯 노드 후처리: 리랭킹을 한 줄로
- 🗂️ 인덱스 종류와 질문 유형
- 🛠️ 헷갈렸던 부분
- 💡 정리하면

---

## 🧭 LlamaIndex는 어떤 프레임워크?

한 줄로 말하면 **"데이터를 LLM이 쓰기 좋게 인덱싱하고 질의하는 데 특화된 프레임워크"**다.

| 강점 | 내용 |
|---|---|
| ① 인덱스 추상화 | 청킹·임베딩·적재가 `from_documents` 한 줄 |
| ② 다양한 인덱스 | 벡터, 요약, 트리, 키워드 인덱스를 골라 쓸 수 있음 |

수업에서 정리한 한 줄 정체성은 **"문서를 넣고 바로 질의"**. 문서 RAG를 만드는 가장 빠른 길이라는 뜻이다. 대신 추상화가 두꺼워서 세밀한 제어는 LangChain보다 덜 투명하다는 단점이 있다.

## 🔄 핵심 흐름: Document → Node → Index → QueryEngine

```
Document(원문) → Node(청킹+메타) → VectorStoreIndex → QueryEngine(검색+생성)
```

| 용어 | 뜻 |
|---|---|
| `Document` | 원문 그대로 |
| `Node` | 청크 단위. 메타데이터와 노드 간 관계를 가지고 있어서 앞뒤 문맥 확장도 가능 |
| `Index` | 노드를 검색할 수 있게 저장해 둔 구조 |
| `QueryEngine` | retrieve → 후처리 → synthesize(답변 생성)를 묶어 둔 고수준 인터페이스 |

그리고 `resp.source_nodes`로 **답변의 근거 출처까지 같이 돌려준다.** 어떤 문서를 보고 답했는지 바로 확인할 수 있어서 좋았다.

## ⚙️ 같은 코퍼스로 RAG 구현하기

```python
Settings.embed_model = HuggingFaceEmbedding('multilingual-e5-small')
Settings.llm = OpenAILike(model=MLAPI_MODEL, api_base=…, is_chat_model=True)

index = VectorStoreIndex.from_documents(docs)   # 청킹·임베딩·인덱싱 한 줄
resp = index.as_query_engine(similarity_top_k=3).query('RAG와 파인튜닝의 차이는?')
```

- `Settings.embed_model`: 임베딩 모델 지정. 한국어가 섞인 문서라 다국어 모델(`multilingual-e5-small`)을 썼다.
- `Settings.llm`: 답변을 만드는 LLM 지정
- `from_documents(docs)`: **여기서 청킹 → 임베딩 → 인덱싱이 전부 처리된다**
- `similarity_top_k=3`: 질문과 가장 비슷한 노드 3개를 가져오라는 뜻

실행 결과는 이렇게 나왔다.

```
출처(source_nodes): ['파인튜닝', 'RAG', '평가']
답변: RAG는 외부 지식을 검색해 답하고, 파인튜닝은 가중치를 추가학습…
```

질문이 "RAG와 파인튜닝의 차이"인데 검색된 출처가 `파인튜닝`, `RAG`, `평가` 문서라서 **검색이 제대로 된 것**을 출처로 바로 확인할 수 있었다.

## 🎯 노드 후처리: 리랭킹을 한 줄로

LlamaIndex는 `node_postprocessors`로 **리랭커, 유사도 필터, 컨텍스트 압축**을 쿼리 엔진에 바로 끼울 수 있다.

```python
rerank = SentenceTransformerRerank(model='BAAI/bge-reranker-v2-m3', top_n=3)
qe = index.as_query_engine(similarity_top_k=8, node_postprocessors=[rerank])
```

흐름은 **2단계 검색**이다.

```
1단계: 벡터 검색으로 후보 8개를 넉넉히 가져옴 (similarity_top_k=8)
2단계: 리랭커가 질문과 더 잘 맞는 순서로 다시 정렬해서 상위 3개만 남김 (top_n=3)
```

처음엔 빨리 많이 찾고, 그다음 정교한 모델로 추려내는 방식이다. 이 2단계 검색이 **설정 한 줄**로 된다는 게 수업에서 말한 "리랭킹/압축이 1급 시민"이라는 뜻이었다. 한국어에서는 영어용 리랭커 대신 `bge-reranker-v2-m3` 같은 다국어 모델을 써야 한다.

## 🗂️ 인덱스 종류와 질문 유형

| 인덱스 | 특징 |
|---|---|
| `VectorStoreIndex` | 의미 검색(기본). 질문과 비슷한 top-k 청크를 가져옴 |
| `SummaryIndex` | 전체 노드를 순회해서 "전체 요약"을 만듦 |
| `TreeIndex` / `KeywordTableIndex` | 계층 요약을 따라 내려가며 탐색 / 키워드 매핑 |

여기서 가장 인상 깊었던 실험이 **"전체 문서를 3문장으로 요약해줘"** 였다.

```python
summary_qe = summary_index.as_query_engine(response_mode="tree_summarize")
summ = summary_qe.query("전체 문서를 3문장으로 요약해줘")
```

| 방식 | 읽은 source_nodes |
|---|---|
| SummaryIndex + tree_summarize | 10개 (전체 문서 10개를 모두 읽음) |
| VectorStore (top_k=3) | 3개 (일부만 봄) |

벡터 검색은 질문과 비슷한 3개만 가져오니까, **문서 전체를 아우르는 요약에는 부적합**하다. 정리하면 **질문 유형에 맞게 인덱스를 골라야 한다**는 것.

## 🛠️ 헷갈렸던 부분

**1) `from_documents` 한 줄이면 끝인데, 왜 굳이 설정을 건드리나?**
→ 기본값은 OpenAI 기반이라 API 키가 필요하고 한국어에서는 아쉬울 수 있다. 그래서 `Settings`로 임베딩과 LLM을 바꿔 주는 것이다. 한 줄은 "기본값으로 일단 돌리는" 용도다.

**2) LlamaIndex가 OpenAI 전용인가?**
→ 아니다. 기본값만 OpenAI일 뿐, LLM·임베딩·벡터DB 모두 교체할 수 있다.

**3) 그럼 단점은?**
→ 추상화가 두꺼워서 내부가 덜 보인다. 그리고 복잡한 에이전트 흐름은 LangGraph가 더 강하다. 그래서 **검색·인덱싱은 LlamaIndex, 에이전트 오케스트레이션은 LangGraph**로 같이 쓰는 팀도 많다고 한다.

## 💡 정리하면

| 개념 | 한 줄 요약 |
|---|---|
| `from_documents` | 청킹·임베딩·인덱싱을 한 줄로 처리 |
| `Node` | 메타데이터와 관계를 가진 청크 |
| `QueryEngine` | 검색 → 후처리 → 답변 생성을 묶은 인터페이스 |
| `source_nodes` | 답변의 근거 출처 |
| `node_postprocessors` | 리랭커·필터·압축을 쿼리 엔진에 끼우는 곳 |
| `SummaryIndex` | 전체 요약처럼 문서 전체를 봐야 하는 질문용 |

- LlamaIndex는 **문서 중심 RAG, 빠른 프로토타입, 요약/계층 질의**가 필요할 때 1순위다.
- 빠른 만큼 내부가 숨겨지니, 품질을 올리려면 청킹·임베딩·리랭커 설정을 직접 건드릴 줄 알아야 한다.
- 질문 유형(검색형 vs 전체 요약형)에 맞춰 인덱스를 고른다.
