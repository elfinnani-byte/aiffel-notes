# [에이전트 팀 꾸리기] - 3. LangGraph의 세 가지 재료 — State, Node, Edge

---

## LangGraph의 세 가지 재료 — State, Node, Edge

### 실습 환경 준비하기

먼저 오늘 사용할 라이브러리를 설치합니다. langgraph가 우리의 그래프 엔진이고, langchain-openai는 OpenAI 모델을 연결하는 어댑터입니다.

> 이 교안은 langgraph 1.2, langchain-core 1.6, langchain 1.4, langchain-openai 1.6 기준으로 작성하고 실제로 실행해 검증했습니다.

**환경: 라이브러리 설치와 버전 확인**

```python
! pip install -q langgraph langchain langchain-openai

from importlib.metadata import version
for pkg in ["langgraph", "langchain-core", "langchain", "langchain-openai"]:
    try:
        print(f"{pkg:18s}: {version(pkg)}")
    except Exception:
        print(f"{pkg:18s}: 미설치")
```

### 첫 그래프 만들기

본격적인 개념 설명 전에 일단 그래프 하나를 직접 만들어 돌려보겠습니다. LLM을 제외하고 LangGraph의 기본 뼈대만 경험해봅시다.

raw 텍스트를 입력하면 공백을 정리하고 글자 수를 계산하는 파이프라인을 만들어보겠습니다.

**실습: LLM 없이 만드는 첫 LangGraph 그래프**

```python
from typing_extensions import TypedDict
from langgraph.graph import StateGraph, START, END

# ── ① State: 노드들이 함께 읽고 쓰는 공용 데이터의 '모양'을 선언한다
class NoteState(TypedDict):
    raw: str        # 들어온 원본 강의 노트
    cleaned: str    # 정리된 노트
    length: int     # 글자 수
    log: str        # 처리 이력

# ── ② Node: 상태를 받아 '바꿀 부분만' 딕셔너리로 돌려주는 함수
def clean_text(state: NoteState):
    cleaned = " ".join(state["raw"].split())            # 중복 공백·앞뒤 공백 제거
    return {"cleaned": cleaned,                         # 손대지 않은 키(raw)는 적지 않는다
            "log": state["log"] + " → clean"}

def measure(state: NoteState):
    return {"length": len(state["cleaned"]),
            "log": state["log"] + " → measure"}

def summarize_meta(state: NoteState):
    return {"log": state["log"] + " → summarize_meta(done)"}

# ── ③ Edge: 노드를 어떤 순서로 이을지 그린다
builder = StateGraph(NoteState)
builder.add_node("clean_text", clean_text)
builder.add_node("measure", measure)
builder.add_node("summarize_meta", summarize_meta)

builder.add_edge(START, "clean_text")          # START: 그래프의 진입점
builder.add_edge("clean_text", "measure")
builder.add_edge("measure", "summarize_meta")
builder.add_edge("summarize_meta", END)        # END: 그래프의 종료점

graph = builder.compile()                       # 컴파일하면 실행 가능한 객체가 된다

result = graph.invoke({
    "raw": "   리듀서는   값을 덮어쓰는 대신  이어 붙인다.  ",
    "log": "start",
})
for k, v in result.items():
    print(f"{k:>8} : {v!r}")
```

출력 결과를 보시면 raw 값은 그대로 남아있고, log 값은 계속 뒤에 이어 붙었다는 걸 알 수 있습니다. LangGraph의 3대 요소를 하나씩 더 자세하게 살펴보면서 출력 결과를 이해해보겠습니다.

#### State

State는 그래프가 실행되는 동안 노드들이 함께 읽고 쓰는 공용 데이터입니다.

노드는 State를 입력으로 받아서 노드에서 변환한 부분만 딕셔너리로 반환합니다. LangGraph는 반환된 딕셔너리를 기존 State에 병합하는 방식으로 동작합니다. 그래서 `clean_text`가 `{"cleaned": ..., "log": ...}`만 반환해도 `raw`는 사라지지 않습니다.

이 방식을 **부분 업데이트(partial update)**라고 합니다. 상태 필드가 스무 개로 늘어나도 각 노드는 자기 관심사만 다루면 되니, 노드를 독립적으로 만들고 테스트하기가 쉬워집니다.

```python
# ── ① State: 노드들이 함께 읽고 쓰는 공용 데이터의 '모양'을 선언한다
class NoteState(TypedDict):
    raw: str        # 들어온 원본 강의 노트
    cleaned: str    # 정리된 노트
    length: int     # 글자 수
    log: str        # 처리 이력

builder = StateGraph(NoteState)
```

#### Node

노드는 작업이 이루어지는 단위입니다. State를 받아서 딕셔너리로 반환하는 파이썬 함수로 만들 수 있습니다. 노드가 그냥 평범한 파이썬 함수라는 점이 LangGraph의 큰 장점입니다. 예시처럼 문자열을 다듬어도 되고, 데이터베이스를 조회해도 되고, LLM을 불러도 되고, 다른 그래프를 실행해도 됩니다.

함수로 정의한 노드는 `add_node("이름", 함수)`로 등록할 수 있고, 이때 지정한 이름이 엣지를 연결할 때의 주소가 됩니다.

```python
# ── ② Node: 상태를 받아 '바꿀 부분만' 딕셔너리로 돌려주는 함수
def clean_text(state: NoteState):
    cleaned = " ".join(state["raw"].split())            # 중복 공백·앞뒤 공백 제거
    return {"cleaned": cleaned,                         # 손대지 않은 키(raw)는 적지 않는다
            "log": state["log"] + " → clean"}

# add_node("이름", 함수명)를 통해 node 등록
builder.add_node("clean_text", clean_text)
```

#### Edge

Edge는 노드에서 노드로의 이동입니다. `add_edge(A, B)`는 "A 노드가 끝나면 B 노드로 간다"는 뜻입니다.

`START`와 `END`는 LangGraph가 미리 정의해 둔 특별한 지점입니다. `START`는 그래프의 진입점이고 `END`는 종료점입니다. 우리가 `invoke()`에 넘긴 딕셔너리가 초기 State가 되어 `START`에서 출발하고, `END`에 닿으면 실행이 끝나면서 그 시점의 State 전체가 반환값이 됩니다.

마지막으로 `compile()`을 통해서 설계한 그래프를 실행 가능한 객체로 바꾸는 단계입니다.

```python
builder.add_edge(START, "clean_text")          # START: 그래프의 진입점
builder.add_edge("clean_text", "measure")
builder.add_edge("measure", "summarize_meta")
builder.add_edge("summarize_meta", END)        # END: 그래프의 종료점

graph = builder.compile()   

result = graph.invoke({
    "raw": "   리듀서는   값을 덮어쓰는 대신  이어 붙인다.  ",
    "log": "start",
})
```

![방금 만든 첫 그래프 — 세 노드를 고정 엣지로 일렬로 연결한 가장 단순한 형태](langgraph-first-graph-three-nodes-diagram.jpg)<span class="img-caption">그림: 방금 만든 첫 그래프 — 세 노드를 고정 엣지로 일렬로 연결한 가장 단순한 형태</span>

#### 🚀 더 해보기

노드나 엣지를 변형시켜보면서 LangGraph가 작동하는 방식에 대해서 파악해봅시다.

**Q. State 병합 규칙을 직접 확인하기**

```python
# Q1. measure 노드가 log를 갱신하지 않도록 (즉 반환값에서 "log" 키를 지워서) 바꿔 보세요.
#     최종 log 값이 어떻게 달라질까요? 실행하기 전에 먼저 예상해 보고 확인하세요.
def measure_v2(state: NoteState):
    return {"length": len(state["cleaned"])}      # log를 반환하지 않는다

# Q2. 노드 순서를 바꿔서 measure를 clean_text보다 먼저 실행하면 어떻게 될까요?
#     (힌트: cleaned 키에는 아직 아무 값도 없습니다. KeyError가 날 겁니다.
#      TypedDict는 '선언'일 뿐 실행 시점의 값 존재를 보장하지 않는다는 점을 확인하세요.)

b2 = StateGraph(NoteState)
b2.add_node("clean_text", clean_text)
b2.add_node("measure", measure_v2)

b2.add_edge(START, "clean_text")
b2.add_edge("clean_text", "measure")
b2.add_edge("measure", END)

r2 = b2.compile().invoke({"raw": "  조건부   엣지는 다음 노드를  고른다.  ", "log": "start"})
print("log    :", r2["log"])      # 'start → clean' 에서 멈춰 있을 것이다
print("length :", r2["length"])
```

> [!question]+ 객관식 퀴즈
> Q. LangGraph에서 노드 함수가 `{"cleaned": "정리된 문장"}`만 반환했다. 이때 State의 다른 키(`raw`, `length`, `log`)는 어떻게 될까요?
>
> 1. 반환값에 없는 키는 모두 None으로 초기화된다
> 2. 반환값에 없는 키는 이전 값을 그대로 유지한다
> 3. State 전체를 반환하지 않았으므로 에러가 발생한다
> 4. 반환값에 없는 키는 State에서 삭제된다

#### 📚 이 섹션의 개념을 더 확인할 자료

- [LangGraph — Use the Graph API](https://docs.langchain.com/oss/python/langgraph/graph-api)
- [LangGraph 가이드북(한국어) — StateGraph 이해하기](https://wikidocs.net/261579)
- [Python 공식 문서 — typing.TypedDict](https://docs.python.org/3/library/typing.html#typing.TypedDict)

---

## 전체 커리큘럼 구성표

|  강좌 번호   | 강좌명                                     |
| :------: | :--------------------------------------- |
|  **1강**  | [[AT01-01 LangGraph로 에이전트 워크플로 설계하기]]           |
|  **2강**  | [[AT01-02 에이전트의 재료 — 도구, 지식 증강, 메모리, 그리고 프레임워크]]           |
|  **3강**  | **3강. LangGraph의 세 가지 재료 — State, Node, Edge (현재)**           |
|  **4강**  | [[AT01-04 조건부 엣지로 분기 만들고 시각화 하기]]           |
|  **5강**  | [[AT01-05 에이전트에게 도구(Tool)를 쥐여주기]]           |
|  **6강**  | [[AT01-06 패턴 ① 프롬프트 체이닝 (Prompt Chaining)]]           |
|  **7강**  | [[AT01-07 패턴 ② 라우팅 (Routing)]]           |
|  **8강**  | [[AT01-08 패턴 ③ 병렬화 (Parallelization)]]           |
|  **9강**  | [[AT01-09 패턴 ④ 오케스트레이터-워커 (Orchestrator-Worker)]]           |
|  **10강**  | [[AT01-10 패턴 ⑤ 평가자-최적화자 (Evaluator-Optimizer)]]           |
|  **11강**  | [[AT01-11 어떤 패턴을 고를 것인가 — 선택 기준과 LangSmith 관측]]           |
|  **12강**  | [[AT01-12 정리]]           |
