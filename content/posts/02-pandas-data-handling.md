+++
title = "[Data Analysis] Pandas 예제로 익히는 데이터 가공 (loc·iloc·groupby·merge)"
date = 2026-07-10
draft = false
tags = ["Python", "Pandas", "Data Preprocessing", "EDA"]
categories = ["data-analysis"]
math = false
+++

[앞선 NumPy 글](/posts/)에서 배열을 다뤘다면, 이번에는 그 위에 표(table)를 얹는 **Pandas**다.

Pandas 예제 65문제를 풀었는데, 예제용으로 깔끔하게 정리된 데이터가 아니라 **서버 점검 · 군수품 청구 · 장병 복지시설** 세 가지 실제 성격의 CSV를 놓고 풀어서 훨씬 실전에 가까웠다.

컬럼도 많고 결측치도 있는 데이터라, 함수 하나하나가 왜 필요한지 자연스럽게 체감됐다.

> 📓 **실습 노트북 전체**: [GitHub에서 보기](https://github.com/heo-aio/heo-aio.github.io/blob/main/static/notebooks/pandas-examples.ipynb) · [다운로드 (.ipynb)](/notebooks/pandas-examples.ipynb)
>
> 이 글은 아래 노트북 65문제 중 핵심만 추린 정리본이다. 전체 문제와 실행 결과는 노트북에서 볼 수 있다.

---

# 📌 Pandas 작업의 큰 흐름

실제로 데이터를 받으면 대체로 이 순서로 움직였다.

```text
데이터 읽기

↓

구조 파악 (shape / info / describe)

↓

행·열 선택 (loc / iloc)

↓

조건 필터링

↓

정렬 · 그룹 집계

↓

결측 · 파생 · 결합
```

이 흐름을 따라 정리한다.

---

# 1️⃣ 데이터 구조부터 파악하기

데이터를 받으면 값을 보기 전에 규모와 타입부터 확인하는 습관을 들였다.

```python
df.shape                 # (행, 열) 규모
df.info()                # 각 열의 dtype과 비결측 개수
df.describe()            # 수치형 열의 분포 요약
df.head(7)               # 앞 7행 육안 확인
df.sample(5, random_state=42)   # 무작위 표본
```

특히 `info()`에서 날짜가 `object`(문자열)로 들어와 있으면, 나중에 `to_datetime`이 필요하다는 신호로 읽었다.

---

# 2️⃣ loc와 iloc — 라벨이냐 위치냐

가장 기본이면서 가장 자주 헷갈린 것이 `loc`과 `iloc`이었다.

정리하면 `loc`은 **이름(라벨)** 기반, `iloc`은 **정수 위치** 기반이다.

```python
df.loc[:, ["server_id", "location", "cpu_usage"]]   # 열 '이름'으로 선택
df.iloc[:5, :3]                                      # 앞 5행, 앞 3열 (위치)
df.iloc[-1]                                          # 마지막 행
```

헷갈릴 때는 "loc은 label의 L"이라고 외우기로 했다.

그리고 `df["col"]`은 Series, `df[["col"]]`은 DataFrame이 나온다는 것도 대괄호 개수 때문에 처음엔 은근 실수했다.

---

# 3️⃣ 조건 필터 — and 대신 &

복합 조건 필터에서 파이썬 습관대로 `and`를 썼다가 에러가 났다.

Pandas에서는 `&`, `|`를 쓰고, 각 조건을 **괄호로 감싸야** 한다.

```python
# 틀린 코드: df[df["fix_required"] == 1 and df["issue_detected"] == 1]  → 에러
df[(df["fix_required"] == 1) & (df["issue_detected"] == 1)]   # 올바른 코드
```

여러 값 중 하나면 되는 조건은 긴 OR 체인 대신 `isin`으로 깔끔하게 쓴다.

```python
df[df["check_type"].isin(["일간", "주간"])]     # 둘 중 하나
df[~df["check_type"].isin(["일간", "주간"])]    # 부정은 ~
df[df["score"].between(60, 80)]                # 구간은 between
```

---

# 4️⃣ 정렬과 그룹 집계(groupby)

정렬은 `sort_values`, 상위 n개는 `head`와 조합한다.

```python
df.sort_values("cpu_usage", ascending=False).head(8)
df.sort_values(["location", "check_date"])       # 다중 키 정렬
```

`groupby`는 처음엔 결과가 왜 이렇게 나오나 싶었는데, **"그룹으로 쪼개고, 각 그룹에서 계산하고, 다시 합친다"** 는 흐름으로 이해하니 납득됐다.

```python
df.groupby("issue_category")["cpu_usage"].mean()                  # 그룹별 평균
df.groupby("check_type")["fix_duration_hours"].agg(["mean", "max"])  # 여러 통계 한 번에
df.groupby(["region", "facility_type"])["satisfaction"].mean()   # 다중 그룹
```

몇 가지 헷갈렸던 차이를 메모해 두었다.

- `size()`는 결측과 무관한 **그룹의 행 수**, `count()`는 **비결측 값만** 센다.
- 다중 그룹 결과가 길면 `unstack()`으로 표(피벗) 형태로 펼 수 있다.

---

# 5️⃣ 결측 · 중복 · 파생 변수

결측치는 열 단위로 개수부터 확인한다.

```python
df.isna().sum()                       # 열별 결측 개수
df["Age"].fillna(df["Age"].median())  # 중앙값으로 대체
```

중복은 완전 중복 행을 세거나, 키 기준으로 제거한다.

```python
df.duplicated().sum()                       # 완전 중복 행 개수
df.drop_duplicates(subset=["supply_id"])    # 키 기준 중복 제거
```

파생 변수는 `assign`으로 원본을 보존하며 만들고, 연속값은 `pd.cut`으로 구간을 나눈다.

```python
df.assign(usage_sum=df["cpu_usage"] + df["memory_usage"] + df["disk_usage"])
pd.cut(df["ratio"], bins=3, labels=["low", "mid", "high"])
```

---

# 6️⃣ 문자열 · 시계열 다루기

문자열은 `str` 접근자로 처리한다.

```python
df["item_name"].str.upper()                      # 대문자 통일
df["location"].str.contains("사단", na=False)     # 패턴 포함 여부
```

시계열은 문자열을 먼저 datetime으로 바꿔야 아무것도 시작된다.

```python
df["check_date"] = pd.to_datetime(df["check_date"], errors="coerce")
df["check_date"].dt.year        # .dt 접근자로 연/월/요일 추출
df["check_date"].dt.dayofweek   # 월=0 … 일=6
```

깨진 값은 `errors="coerce"`로 NaT 처리한다는 점, `dayofweek`는 월요일이 0이라는 점을 따로 적어두었다.

---

# 7️⃣ merge — inner인데 행이 늘어났다

군수품과 복지 데이터를 `unit_code`로 inner merge 했는데, 결과 행 수가 원본보다 많아져서 당황했다.

같은 키가 양쪽에 여러 개 있으면 조합만큼 행이 곱해진다(다대다).

```python
merged = pd.merge(df_sup, df_wel, on="unit_code", how="inner",
                  suffixes=("_sup", "_wel"))
print(merged.shape)   # 원본보다 커질 수 있다
```

"merge하면 무조건 행이 줄어든다"고 막연히 생각했던 것이 틀렸던 것이다.

집계값을 다시 원본에 붙이는 패턴(`location`별 평균 CPU를 각 행에 merge로 얹기)도 처음 봤는데, 나중에 피처를 만들 때 쓸 것 같았다.

> **핵심 정리**
>
> merge는 항상 행을 줄이는 연산이 아니다. 키가 양쪽에서 유일하지 않으면 행이 오히려 늘어날 수 있으므로, merge 전에 키의 유일성을 확인하는 습관이 필요하다.

---

# 📊 Pandas 작업 요약

| 단계 | 대표 메서드 |
|------|-------------|
| 구조 파악 | `shape`, `info`, `describe`, `head` |
| 선택 | `loc`(라벨), `iloc`(위치) |
| 필터 | `&`/`|` + 괄호, `isin`, `between` |
| 집계 | `groupby`, `agg`, `size`/`count` |
| 결측·파생 | `isna`, `fillna`, `assign`, `cut` |
| 시계열 | `to_datetime`, `.dt` |
| 결합 | `merge`, `concat`, `pivot_table` |

---

# 💡 공부하면서 느낀 점

세 가지 성격이 다른 데이터를 같은 흐름으로 반복하다 보니, Pandas가 단순한 함수 모음이 아니라 **"데이터를 이해하는 절차"** 라는 감이 잡혔다.

특히 merge에서 행이 폭증하는 경험은, 코드가 에러 없이 돌아가더라도 결과가 틀릴 수 있다는 걸 알려줬다. "돌아가니까 맞겠지"가 아니라 행 수와 분포를 직접 확인하는 습관이 중요하다는 걸 배웠다.

다음 글에서는 이렇게 가공한 데이터를 눈으로 확인하는 **Matplotlib** 시각화 원칙을 정리한다.
