+++
title = "[LangGraph] RAG에서 에이전트까지 - 도구 호출 루프와 State 이해하기"
date = 2026-10-12
draft = false
categories = ["dev"]
tags = ["LangChain", "LangGraph", "RAG", "Agent", "ToolNode", "bind_tools", "add_messages", "Python"]
summary = "RAG 코드를 따라 치다가 'messages[-1].content는 어떻게 아는 거지?', 'bind_tools가 왜 필요하지?'가 궁금해서 하나씩 파고들었다. 도구 호출 루프, State와 add_messages, LangChain과 LangGraph의 차이, 그리고 오늘 만난 에러 3개를 정리했다."
+++

RAG 코드와 LangGraph 에이전트 예제를 따라 치면서 "돌아가긴 하는데 왜 이렇게 쓰지?" 싶은 부분이 계속 나왔다. 이번 글은 그 **"왜?"를 하나씩 풀어본 기록**이다. 결론부터 말하면 에이전트는 **판단 + 실행 + 반복**이다.

## 목차
- 📚 RAG 답변 생성 노드
- 🧰 도구 호출: 판단과 실행은 다르다
- 🗂️ State와 add_messages
- 🔁 에이전트 = 판단 + 실행 + 반복
- ⚖️ LangChain vs LangGraph
- 🚨 오늘 만난 에러 3개
- 🛠️ 헷갈렸던 부분
- 💡 정리하면

---

## 📚 RAG 답변 생성 노드

RAG는 **오픈북 시험**과 같다. 질문을 받고, 교과서(문서)에서 관련 페이지를 찾고, 그걸 보면서 답을 쓴다.

```python
def generate_answer(state):
    messages = state["messages"]
    query = messages[-1].content          # 마지막 메시지 = 방금 한 질문

    system_prompt = (
        "당신은 질문-답변 전문가입니다. 다음 정보를 활용해 답하세요. "
        "모르면 모른다고 답하세요.\n{context}"
    )
    prompt = ChatPromptTemplate.from_messages(
        [("system", system_prompt), ("human", "{input}")]
    )

    question_answer_chain = create_stuff_documents_chain(llm, prompt)
    rag_chain = create_retrieval_chain(retriever, question_answer_chain)

    response = rag_chain.invoke({"input": query})
    return {"messages": [response["answer"]]}
```

| 부품 | 비유 | 역할 |
|---|---|---|
| `retriever` | 도서관 사서 | 질문과 관련된 문서 검색 |
| `create_stuff_documents_chain` | 요약 노트 작성 + 답 쓰기 | 찾은 문서를 `{context}`에 채워 LLM에 전달 |
| `create_retrieval_chain` | 컨베이어 벨트 | 검색 → 답변 순서로 연결 |

시스템 프롬프트의 "모르면 모른다고 답하라"는 LLM이 지어내는 **환각**을 줄이는 안전장치다.

### `.content`는 어떻게 알고 쓰는 걸까?

`messages[-1]`은 문자열이 아니라 **LangChain 메시지 객체**(택배 상자)라서 `.content`로 내용을 꺼낸다. 모르는 객체는 외우지 말고 열어본다.

```python
print(type(messages[-1]))   # 무슨 타입인지
print(dir(messages[-1]))    # 쓸 수 있는 속성/메서드
```

`type()` → `dir()` → 공식 문서 순서가 기본 습관이다.

## 🧰 도구 호출: 판단과 실행은 다르다

```python
@tool
def get_weather(location: str):
    """Call to get the current weather."""   # LLM이 이 설명을 읽고 쓸지 판단
    ...

tools = [get_weather, get_coolest_cities]
tool_node = ToolNode(tools)                  # 실행 담당
llm_with_tools = llm.bind_tools(tools)       # 판단 담당
```

| | 하는 일 | 못 하는 일 |
|---|---|---|
| `llm.bind_tools(tools)` | 어떤 도구를 어떤 값으로 쓸지 **판단** | 실제 실행 |
| `ToolNode(tools)` | 요청서대로 **실행** | 판단 (자연어를 이해 못 함) |

식당으로 치면 LLM은 주문을 받아 주문서를 쓰는 **홀 직원**, ToolNode는 주문서만 읽는 **주방**이다. 그래서 `bind_tools` 없이 ToolNode만 쓰면 "무슨 도구를 쓰라는 건지" 모른다.

### invoke 하면 뭐가 나올까?

```python
out = llm_with_tools.invoke("what's the weather in sg?")
print(out.tool_calls)
# [{'name': 'get_weather', 'args': {'location': 'sg'}, 'id': '...'}]
```

- 도구가 필요하면 `tool_calls`에 값이 들어간다. (실행 결과가 아니라 **주문서**다.)
- 도구가 필요 없으면 `tool_calls == []`이고, `content`에 모델이 직접 쓴 답변이 들어간다.
- `content`가 비어 있는지는 모델마다 다르다. **판단 기준은 항상 `tool_calls`** 다.

### ToolNode에 넣는 입력 규격

```python
tool_node.invoke({"messages": [llm_with_tools.invoke("what's the weather in sf?")]})
```

- ToolNode는 `"messages"` key에서 마지막 메시지의 `tool_calls`를 읽어 실행한다.
- 결과는 `{"messages": [ToolMessage(...)]}` 형태로 나온다.
- 위 한 줄은 판단 → 실행을 **한 번** 손으로 이어본 실험 코드다.

ToolNode가 대신해주는 일은 사실 이 정도다. (실제로는 에러 처리 등이 더 들어간다.)

```python
call = out.tool_calls[0]
tool_map = {"get_weather": get_weather}
result = tool_map[call["name"]].invoke(call["args"])
```

내가 직접 만드는 건 ToolNode가 아니라 **도구 함수의 내용**이다.

<!-- TODO: tool_calls 출력 결과 캡처 붙이기 -->

## 🗂️ State와 add_messages

```python
class AgentState(TypedDict):
    messages: Annotated[Sequence[BaseMessage], add_messages]
    grounds: str   # retrieval로 가져온 추가 정보
    topic: str     # 블로그 작성용 토픽
```

- `Annotated[타입, 규칙]`은 타입에 꼬리표를 붙이는 파이썬 문법이다.
- `add_messages`는 LangGraph가 제공하는 **리듀서**(합치는 규칙)다. 노드가 `{"messages": [새 메시지]}`를 반환하면 기존 기록을 덮어쓰지 않고 **뒤에 이어 붙인다.**
- 이 규칙이 없는 칸(`grounds`, `topic`)은 기본값인 **덮어쓰기**로 동작한다.
- `BaseMessage`는 `HumanMessage`, `AIMessage`, `ToolMessage`의 부모 클래스다.

```
messages = [
  HumanMessage("sf 날씨 알려줘"),
  AIMessage(tool_calls=[get_weather]),
  ToolMessage("20 degrees and foggy"),
  AIMessage("샌프란시스코는 20도, 안개예요"),
]
```

채팅방처럼 말풍선이 순서대로 쌓이니까, LLM이 다시 호출될 때 앞의 흐름을 보고 판단할 수 있다.

## 🔁 에이전트 = 판단 + 실행 + 반복

```
[agent: LLM] -- tool_calls 있음 --> [tools: ToolNode] --+
     ^                                                  |
     +-------------------- 결과 전달 -------------------+
     |
     +-- tool_calls 없음 --> 종료 (content가 최종 답변)
```

```python
graph.add_node("agent", call_llm)
graph.add_node("tools", ToolNode(tools))
graph.add_conditional_edges("agent", tools_condition)  # 분기 (신호등)
graph.add_edge("tools", "agent")                       # 반복
```

- `tools_condition`이 하는 일의 핵심은 `if out.tool_calls:` 한 줄이다.
- 단순 함수 호출과 에이전트를 가르는 건 **결과를 보고 다시 판단하는 루프**다.
- 도구 안쪽은 그냥 파이썬 함수라서 부담이 크지 않다. 진짜 어려운 건 LLM이 도구를 잘 고르도록 **docstring과 도구를 설계하는 일**이다.

## ⚖️ LangChain vs LangGraph

| | LangChain | LangGraph |
|---|---|---|
| 흐름 | 정해진 순서로 한 번 | 분기와 반복 가능 |
| 비유 | 컨베이어 벨트 | 신호등 있는 도로 |
| 데이터 | 단계마다 값 전달 | `State`(공유 장부)에 누적 |
| 적합한 용도 | RAG, 요약, 번역 같은 고정 파이프라인 | 에이전트, 도구 루프, 재시도 |
| 핵심 부품 | Chain, Prompt, Retriever | Node, Edge, State |

둘은 경쟁 관계가 아니다. LangChain이 부품(`ChatOpenAI`, `@tool`, `retriever`)을 제공하고, LangGraph가 그 부품을 어떤 순서로 연결할지 정한다. 순서가 고정이면 LangChain, **상황을 보고 판단하거나 반복해야 하면 LangGraph**라고 기억하기로 했다.

## 🚨 오늘 만난 에러 3개

### 1) `Could not connect to tenant default_tenant`

- 원인: Chroma(벡터DB) 연결 문제. `persist_directory` 폴더 손상, chromadb 버전 불일치, 같은 폴더 중복 접근이 흔하다.
- 해결: 커널 재시작 → DB 폴더 삭제 후 재생성 → `pip install -U chromadb langchain-chroma`
- 팁: `retriever.invoke("테스트")`를 단독으로 돌려서 Chroma 문제인지 도구 연결 문제인지 범위를 좁힌다.

### 2) `AuthenticationError 401 invalid_issuer`

- 원인: `llm`에만 `base_url`을 지정하고 `OpenAIEmbeddings()`에는 지정하지 않아서, 임베딩 요청이 기본 OpenAI 서버로 갔다. 다른 서비스에서 발급받은 키를 OpenAI 서버에 보낸 셈이다.
- 해결: 임베딩에도 같은 `base_url`과 올바른 모델명을 지정하거나 환경변수 `OPENAI_BASE_URL`을 설정한다.
- 교훈: 설정은 **객체마다 따로** 들어간다. "이 객체가 내 설정을 실제로 받았나?"를 의심하자.

### 3) `Function tools with reasoning_effort are not supported ... /v1/chat/completions`

- 원인: reasoning 모델에서 `reasoning_effort`가 켜진 채 `bind_tools`로 도구를 함께 보내면 chat completions 방식에서 거절된다.
- 해결:

```python
llm = ChatOpenAI(model="...", base_url="...", reasoning_effort="none")
# 또는 use_responses_api=True (서버와 라이브러리 버전이 지원해야 함)
```

- 주의: 설정을 바꾼 뒤에는 `llm`을 만든 셀부터 다시 실행해서 `llm_with_tools`도 새로 만들어야 한다.

세 에러 모두 **트레이스백 맨 아래 한두 줄**에 원인이 있었다.


## 🛠️ 헷갈렸던 부분

**1) bind_tools 없이 ToolNode만 쓰면 안 되나?**
→ 안 된다. ToolNode는 `tool_calls`가 담긴 메시지를 입력으로 기대한다. 자연어 질문을 이해해서 도구를 고르는 건 LLM의 몫이다.

**2) 지금 실습 코드에는 `add_messages` import가 없는데?**
→ 그래프(State)를 만들지 않고 ToolNode를 **단독 실험**하고 있어서 필요가 없었다. State 클래스를 정의하는 단계에서 import하면 된다.

**3) State의 key를 `messages`가 아닌 다른 이름으로 써도 되나?**
→ 내가 만든 노드만 쓴다면 자유다. 다만 `ToolNode`, `MessagesState` 같은 기성품은 `messages`를 기본으로 읽기 때문에, 사실상 `messages`로 맞추는 게 표준이다.

**4) 이 기술들은 얼마나 새로운 걸까?**
→ RAG 개념은 몇 년 됐지만 LangChain, LangGraph는 변화가 빠르다. 예제가 안 돌아가면 내 실수보다 **버전 차이**일 때가 많다. 프레임워크 이름은 바뀌어도 "검색해서 근거를 붙인다 / LLM이 도구를 고르고 코드가 실행한다 / 결과를 보고 반복한다"는 원리는 그대로라서, 원리를 이해하는 쪽이 오래 간다.

## 💡 정리하면

| 개념 | 한 줄 요약 |
|---|---|
| RAG | 문서를 먼저 찾아 보여주고 거기서 답하게 하는 오픈북 방식 |
| `bind_tools` | LLM에게 쓸 수 있는 도구 메뉴판을 건네는 것 (판단) |
| `ToolNode` | `tool_calls`대로 도구를 실행해 `ToolMessage`로 돌려주는 기성품 (실행) |
| `tool_calls` | 도구 호출 여부의 진짜 증거. 비어 있으면 모델이 직접 답한 것 |
| `add_messages` | 메시지를 덮어쓰지 않고 이어 붙이는 리듀서 |
| 에이전트 | 판단 + 실행 + 반복 |
| LangGraph | 부품을 분기/반복이 있는 그래프로 연결하는 도구 |

- 에러는 **종류 / 메시지 / 위치**로 쪼개서 읽고, 질문하기 전에 `type()`이나 단독 `invoke()`로 먼저 확인한다.
- 보고 → 덮고 → 짜고 → 바꿔본다. 읽기만 하면 이해한 착각이 생긴다.
- 다음엔 `StateGraph`로 날씨 에이전트를 직접 조립하고, `retriever`를 `@tool`로 감싸 Agentic RAG까지 만들어볼 예정이다.