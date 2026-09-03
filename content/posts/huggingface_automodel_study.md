+++
title = "[Hugging Face] AutoModel vs AutoModelForXXX - 헤드, hidden_states, generate()까지 파고들기"
date = 2026-09-04
draft = false
tags = ["HuggingFace", "Transformers", "NLP", "AI"]
categories = ["dev"]
math = false
+++

허깅페이스 강의를 들으면서 `AutoModel`이랑 `AutoModelForSequenceClassification` 같은 걸 그냥 "이름이 다르니 다른 모델이겠지" 하고 넘어갔었다. 근데 실습 코드를 직접 돌려보니 `outputs`로 뭐가 나오는지, `logits`가 뭔지, `generate()`가 어떻게 문장을 만들어내는지 하나도 설명이 안 됐다. 그래서 실습 파일들(`09_auto_model.py`, `10_last_hidden_state.py`, `04_chat.py` 등)을 다시 열어놓고 "이게 왜 이렇게 나오지"를 하나씩 파고들면서 정리했다. 이 글은 그 과정에서 잡힌 개념들을 순서대로 기록한 것이다.

**목차**

1. [🧩 AutoModel과 AutoModelForXXX, 뭐가 다른가](#-automodel과-automodelforxxx-뭐가-다른가)
2. [🎯 여기서 말하는 "태스크"가 정확히 뭔가](#-여기서-말하는-태스크가-정확히-뭔가)
3. [🔩 헤드(head)의 정체 - 사실 Linear 레이어 하나였다](#-헤드head의-정체---사실-linear-레이어-하나였다)
4. [📐 hidden_states vs pooler_output - 뭘 언제 쓰나](#-hidden_states-vs-pooler_output---뭘-언제-쓰나)
5. [🔢 logit과 softmax - 점수를 확률로 바꾸는 과정](#-logit과-softmax---점수를-확률로-바꾸는-과정)
6. [🔁 generate()는 forward()를 반복하는 함수였다](#-generate는-forward를-반복하는-함수였다)
7. [💡 정리하면](#-정리하면)

---

## 🧩 AutoModel과 AutoModelForXXX, 뭐가 다른가

처음엔 `AutoModel.from_pretrained(model_id)`랑 `AutoModelForSequenceClassification.from_pretrained(model_id)`가 그냥 같은 걸 부르는 다른 방법인 줄 알았다. 근데 `10_last_hidden_state.py`로 `AutoModel`을 직접 불러서 `output_hidden_states` 안 켜고 그냥 출력해보니, `last_hidden_state`라는 게 나왔다.

```python
model = AutoModel.from_pretrained(model_id)
outputs = model(**inputs, output_hidden_states=False)

lhs = outputs.last_hidden_state
print(f"last_hidden_states shape = {lhs.shape}") # [1, 13, 768]
```

`[1, 13, 768]` — 문장 1개, 토큰 13개(특수토큰 포함), 각 토큰마다 768차원 숫자 벡터. 근데 이게 "이 문장이 긍정이다" 같은 결과가 전혀 아니다. 그냥 숫자 덩어리다.

반면 `09_auto_model.py`에서 쓴 `AutoModelForSequenceClassification`은 결과에 `logits`가 딸려 나왔고, 그걸 softmax 통과시키니 진짜 "POSITIVE 몇%" 같은 결과가 나왔다.

```python
model = AutoModelForSequenceClassification.from_pretrained(model_id, dtype=torch.float16)
outputs = model(**inputs)
# outputs.logits 존재
```

정리하면 이렇다.

- `AutoModel` = 백본(BERT/GPT 본체)만 불러옴 → 숫자 벡터(hidden_states)만 반환
- `AutoModelForXXX` = 백본 + 태스크 전용 레이어(헤드) → 사람이 쓸 수 있는 결과(라벨 확률, 다음 토큰 등) 반환

![AutoModel과 AutoModelForXXX의 구조 차이. AutoModel은 백본만 거쳐 hidden_states를 반환하고, AutoModelForXXX는 백본 뒤에 태스크 전용 헤드가 붙어 최종 결과를 반환한다](/images/HF/hf_backbone_head_flow.svg)

> ✅ **핵심**: `AutoModel`은 문장을 벡터로 바꾸는 것까지만 하고, `AutoModelForXXX`는 그 벡터를 가지고 "그래서 결론이 뭔데"까지 답해준다. 둘 다 백본은 똑같이 쓰고, 뒤에 뭘 더 붙였느냐의 차이다.

---

## 🎯 여기서 말하는 "태스크"가 정확히 뭔가

강의자료(`auto.txt`)에 있던 표를 다시 보니, "태스크"라는 게 그냥 "AI가 하는 일" 정도로 뭉뚱그려 이해하면 안 되는 거였다. 정확히는 **입력을 받아서 어떤 형태의 출력을 낼 것인가에 대한 문제 정의**였다.

```
클래스명                            | 태스크           | 입력 -> 출력
AutoModelForSequenceClassification | 텍스트 분류      | 텍스트 -> 클래스별 확률
AutoModelForTokenClassification    | 토큰 단위 분류    | 텍스트 토큰들 -> 각 토큰의 태그
AutoModelForQuestionAnswering      | 질의응답 (추출형) | 질문+본문 -> 정답 시작/끝 위치
AutoModelForCausalLM                | 언어 생성        | 프롬프트 -> 다음 토큰
AutoModelForSeq2SeqLM               | 시퀀스 변환      | 입력 시퀀스 -> 출력 시퀀스
```

같은 "텍스트를 얻고 싶다"는 목적이어도 번역, 요약, 다음 문장 생성은 전부 다른 태스크다. 왜냐면 입력과 출력의 관계 구조가 다르기 때문이다. 그래서 클래스 이름도 `Seq2SeqLM`, `CausalLM`으로 나뉘고, 실제로 헤드 구조 자체가 서로 다르게 설계된다.

> ✅ **핵심**: 태스크는 "AI가 뭘 해주는가"가 아니라 "입력 → 출력을 어떤 형태로 매핑할 것인가"에 대한 정의다. 그래서 클래스 이름 뒤에 `Classification`이 붙으면 카테고리를 나누는 거고, `LM`이 붙으면 글을 이어 쓰는 거라고 구분하면 헷갈리지 않는다.

---

## 🔩 헤드(head)의 정체 - 사실 Linear 레이어 하나였다

"헤드"라는 단어가 뭔가 대단한 구조처럼 느껴졌는데, 실제로 뜯어보면 정말 단순했다. 개념을 단순화한 코드로 보면 이렇다.

```python
class BertForSequenceClassification(nn.Module):
    def __init__(self, config):
        self.bert = BertModel(config)                  # 백본
        self.dropout = nn.Dropout(0.1)
        self.classifier = nn.Linear(768, num_labels)    # 이게 헤드!

    def forward(self, input_ids, attention_mask, labels=None):
        outputs = self.bert(input_ids, attention_mask)
        pooled_output = outputs[1]                      # [CLS] 토큰 벡터
        pooled_output = self.dropout(pooled_output)
        logits = self.classifier(pooled_output)          # 헤드 통과 -> 최종 점수

        loss = None
        if labels is not None:
            loss = CrossEntropyLoss()(logits, labels)
        return logits, loss
```

핵심은 `self.classifier = nn.Linear(768, num_labels)` 이 한 줄이 헤드의 전부라는 거였다. 백본(BERT)이 파라미터의 95% 이상을 차지하고, 헤드는 그 위에 붙는 정말 얇은 레이어 하나다. 백본이 이미 문장 의미를 잘 압축해놨기 때문에, 마지막 판단은 Linear 레이어 하나로도 충분한 거였다.

그리고 태스크마다 헤드 구조가 달라진다는 것도 여기서 이해가 됐다.

- 문장 분류: `Linear(768 → 클래스수)`, 문장 전체 벡터 1개만 사용
- 토큰 분류: `Linear(768 → 라벨수)`, 모든 토큰 벡터에 각각 적용
- 생성(CausalLM): `Linear(768 → 어휘사전크기)`, 다음 토큰 확률 분포 출력

파인튜닝할 때 `from_pretrained(model_id, num_labels=3)`처럼 불러오면 백본은 사전학습된 가중치 그대로, 헤드는 랜덤 초기화 상태로 시작한다는 것도 처음 알았다. 그래서 파인튜닝 초반에 예측이 엉망인 게 정상이고, 학습이 진행되면서 헤드가 태스크에 맞게 적응해가는 거였다.

> ✅ **핵심**: 헤드는 백본 위에 붙는 작은 레이어(대부분 `Linear` 1개)이고, 백본이 만든 벡터를 우리가 원하는 최종 형태(라벨/다음 토큰 등)로 바꿔주는 역할을 한다. 파인튜닝 시 주로 학습되는 부분도 이 헤드다.

---

## 📐 hidden_states vs pooler_output - 뭘 언제 쓰나

`10_last_hidden_state.py`를 돌리면서 `last_hidden_state`는 봤는데, `AutoModel`을 쓸 때 자주 같이 언급되는 `pooler_output`이 뭔지 헷갈렸다. 둘 다 BERT 출력에서 나오는 건데 모양(shape)부터 다르다.

- **last_hidden_state**: `[배치, 토큰수, 768]` — 문장의 **모든 토큰마다** 벡터가 하나씩 나온다. 토큰 단위 작업(개체명 인식 등)에 쓴다.
- **pooler_output**: `[배치, 768]` — `[CLS]` 토큰 벡터만 뽑아서 Linear + Tanh를 한 번 더 통과시킨, **문장 전체를 대표하는 벡터 1개**다. 문장 단위 작업(분류)에 전통적으로 쓰였다.

![last_hidden_state와 pooler_output 비교. hidden_states는 토큰마다 768차원 벡터가 나오고, pooler_output은 CLS 토큰만 뽑아 Linear+Tanh를 거친 문장 대표 벡터 1개다](/images/HF/hf_hidden_pooler.svg)

실무 팁으로 알게 된 건데, 요즘은 `pooler_output`보다 `hidden_states`를 평균 내는 mean pooling 방식을 문장 임베딩에 더 많이 쓴다고 한다. `pooler_output`이 원래 BERT 사전학습 때 쓰던 NSP(다음 문장 예측) 태스크에 맞춰진 거라, 요즘 기준으로는 성능이 애매할 때가 있어서라고.

> ✅ **핵심**: hidden_states는 토큰별 벡터 묶음, pooler_output은 문장 전체를 압축한 벡터 1개. 지금은 pooler_output보다 hidden_states를 직접 평균 내는 방식이 문장 임베딩엔 더 선호된다.

---

## 🔢 logit과 softmax - 점수를 확률로 바꾸는 과정

`09_auto_model.py`에서 감정분석 모델을 돌렸을 때 이 흐름을 그대로 체감했다.

```python
outputs = model(**inputs)
# outputs.logits = 반환된 순수 숫자 (이게 확률이 얼마나 높은건지 알 수 없다.)

prob = torch.softmax(outputs.logits, dim=-1).tolist()
print(f"POSITIVE : {prob[0][0] * 100:.2f}%")
print(f"NEGATIVE : {prob[0][1] * 100:.2f}%")
```

`logits`는 헤드(Linear)를 통과한 **원점수**다. `[2.1, -0.5]` 같은 형태로 나오는데, 이 자체로는 "긍정일 확률 몇%"인지 알 수 없고 범위도 정해져 있지 않다. 여기에 `softmax`를 적용하면 "합이 1이 되는 확률분포"로 바뀌어서, 사람이 이해할 수 있는 퍼센트 형태가 된다.

여기서 헷갈렸던 포인트 하나: 학습 코드(`BertForSequenceClassification` 예시)에서는 `CrossEntropyLoss()(logits, labels)`처럼 `logits`를 그대로 넣지, softmax를 씌운 확률값을 넣지 않는다. `CrossEntropyLoss` 내부에 이미 softmax 계산이 포함돼 있기 때문이다. 여기에 softmax를 한 번 더 씌우면 이중 적용하는 실수가 된다.

그리고 무조건 softmax만 쓰는 것도 아니었다. "한 문장에 정답이 딱 하나"인 다중 클래스 분류에는 softmax를 쓰지만, 여러 라벨이 동시에 정답일 수 있는 다중 라벨 분류에는 각 클래스별로 독립적인 sigmoid를 쓴다.

> ✅ **핵심**: logit = 확률로 변환되기 전 원점수, softmax = 그걸 확률분포로 바꿔주는 함수. 학습 시 loss 함수 내부에 softmax가 이미 포함된 경우가 많으니 이중 적용하지 않도록 주의해야 한다.

---

## 🔁 generate()는 forward()를 반복하는 함수였다

`04_chat.py`에서 `model.generate()`를 처음 썼을 때는 그냥 "질문 넣으면 답이 나오는 마법 함수"처럼 느껴졌다.

```python
outputs = model.generate(
    **inputs,
    max_new_tokens=256,
    temperature=0.7,
    do_sample=True
)
resp_text = tokenizer.decode(outputs[0], skip_special_tokens=True)
```

근데 `forward()`는 입력에 대해 딱 한 번 `logits`를 계산하는 함수라는 걸 먼저 이해하고 나니, `generate()`가 뭘 하는 건지 감이 왔다. 문장 하나를 통째로 뱉는 게 아니라, **"다음 토큰 하나 예측 → 입력에 이어붙이기"를 반복하는 루프**였다.

```
"오늘 날씨는" → forward() → 다음 토큰 "좋" 예측
"오늘 날씨는 좋" → forward() → 다음 토큰 "다" 예측
"오늘 날씨는 좋다" → ... (반복)
```

![generate() 함수의 자기회귀 생성 루프. forward()로 logits를 계산하고 softmax와 샘플링으로 다음 토큰을 선택한 뒤, 그 토큰을 입력에 이어붙여 다시 forward()를 호출하는 과정을 max_new_tokens만큼 반복한다](/images/HF/hf_generate_loop.svg)

여기서 `do_sample`, `temperature` 파라미터가 왜 필요한지도 정리됐다. 매번 가장 확률 높은 토큰만 뽑으면(greedy) 뻔하고 반복적인 문장이 나오기 쉬운데, `do_sample=True`로 약간의 무작위성을 주면 더 자연스럽고 다양한 문장이 나온다. `07_pipeline_param.py`에서 봤던 `top_k`, `top_p`도 결국 "다음 토큰 후보를 얼마나 넓게/좁게 볼 것인가"를 조절하는 파라미터였다.

그리고 이 원리는 GPT2 같은 작은 모델이든, ChatGPT나 Claude 같은 대형 LLM이든 근본적으로는 똑같다는 걸 알게 됐다. 규모와 후처리 학습(RLHF 등)이 다를 뿐, "다음 토큰을 예측해서 이어붙인다"는 자기회귀 방식 자체는 공통이다.

> ✅ **핵심**: `generate()`는 `forward()`를 반복 호출하면서 매번 다음 토큰을 선택해 입력에 이어붙이는 루프다. 그 선택 방식(greedy/sampling)을 조절하는 게 `do_sample`, `temperature`, `top_p` 같은 파라미터다.

---

## 💡 정리하면

`AutoModel`과 `AutoModelForXXX`를 그냥 "이름이 다른 클래스" 정도로 넘겼다면, `outputs`에 뭐가 담겨오는지, `logits`가 왜 확률이 아닌지, `generate()`가 왜 한 번에 문장을 안 뱉고 반복문처럼 동작하는지 하나도 설명을 못 했을 것 같다. 직접 코드를 돌리면서 shape를 찍어보고, 헤드가 실제로 `Linear` 레이어 한 줄이라는 걸 확인하고 나니 "블랙박스"처럼 느껴지던 것들이 하나씩 풀렸다.

**오늘의 핵심 5가지**

- 🧩 `AutoModel`은 백본만, `AutoModelForXXX`는 백본 + 태스크 전용 헤드까지 포함한다
- 🎯 태스크는 "입력 → 출력을 어떤 형태로 매핑할 것인가"에 대한 정의다
- 🔩 헤드는 대부분 `Linear` 레이어 1개 수준으로 단순하고, 파인튜닝 시 주로 이 부분이 학습된다
- 📐 hidden_states는 토큰별 벡터, pooler_output은 문장 전체를 압축한 벡터 1개
- 🔁 `generate()`는 `forward()` + 다음 토큰 선택을 반복하는 자기회귀 루프다
