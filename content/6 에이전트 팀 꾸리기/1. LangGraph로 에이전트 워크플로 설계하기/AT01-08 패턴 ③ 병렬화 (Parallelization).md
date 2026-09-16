# [에이전트 팀 꾸리기] - 8. 패턴 ③ 병렬화 (Parallelization)

---

## 패턴 3. 병렬화 (Parallelization)

세 번째 패턴은 **병렬화(parallelization)** 입니다. 이번 섹션은 성공 코드가 아니라 **에러 메시지**로 시작합니다. 병렬화에는 앞선 두 패턴에 없던 새로운 문제가 하나 등장하는데, 그 문제를 먼저 겪어 보는 것이 이해에 훨씬 좋습니다.
(에러가 노트북을 멈추지 않도록 try/except 로 감싸 두었습니다.)

**실습: 리듀서 없이 병렬 쓰기 → 일부러 에러 보기 (키 불필요)**

```python
from typing_extensions import TypedDict
from langgraph.graph import StateGraph, START, END

# ── 리듀서 없이, 평범한 list 필드를 두 노드가 동시에 쓰려고 하면?
class BadState(TypedDict):
    checks: list

def check_terms(state: BadState):
    return {"checks": ["용어 정의 점검 완료"]}

def check_example(state: BadState):
    return {"checks": ["예시 유무 점검 완료"]}

def aggregate(state: BadState):
    return {}

b = StateGraph(BadState)
b.add_node("check_terms", check_terms)
b.add_node("check_example", check_example)
b.add_node("aggregate", aggregate)

# START에서 두 노드로 동시에 나간다 → 두 노드가 같은 스텝에 실행된다
b.add_edge(START, "check_terms")
b.add_edge(START, "check_example")
b.add_edge("check_terms", "aggregate")
b.add_edge("check_example", "aggregate")
b.add_edge("aggregate", END)

try:
    print(b.compile().invoke({"checks": []}))
except Exception as e:
    print(f"❌ {type(e).__name__}")
    print(str(e))
```

에러가 발생했습니다.

```
❌ InvalidUpdateError
At key 'reviews': Can receive only one value per step.
Use an Annotated key to handle multiple values.
For troubleshooting, visit: https://docs.langchain.com/oss/python/langgraph/errors/INVALID_CONCURRENT_GRAPH_UPDATE
```

"키 reviews 에서: 한 스텝당 하나의 값만 받을 수 있습니다. 여러 값을 처리하려면 Annotated 키를 쓰세요." 라고 충고를 해주는군요. LangGraph는 한 스텝에서 여러 노드가 동시에 같은 키(Key)에 데이터를 쓰려고 하면, 덮어쓰지 않고 명확하게 에러를 냅니다. 조금 더 자세히 살펴볼까요.

### 슈퍼스텝(super-step)이라는 실행 단위

LangGraph는 노드를 한 번에 하나씩 실행하는 것이 아니라 **스텝(step) 단위로 실행**합니다. 한 스텝에서는 **실행 준비가 된 모든 노드를 동시에** 실행하고, 그 스텝이 끝나면 모든 반환값을 State에 한꺼번에 병합합니다. 그다음 스텝으로 넘어갑니다.

방금 그래프에서 START 에서 연결된 check_accuracy 와 check_empathy 는 **같은 스텝에서 동시에 실행**되었습니다. 그리고 두 노드가 모두 {"reviews": [...]} 를 반환해서 문제가 발생했습니다. LangGraph의 State 기본 병합 규칙은 덮어쓰기인데, 같은 스텝에 두 값이 같은 키로 들어오면, 두 데이터 중 무엇을 살리고 무엇을 버려야 할지 알 수 없기 때문입니다. LangGraph는 이 상황에서 임의로 하나를 고르지 않고 에러를 발생시킵니다.

### 리듀서(reducer) — 병합 방법을 직접 알려주기

해결책은 에러 메시지가 알려 준 대로 Annotated 를 쓰는 것입니다. State의 checks 타입을 list 대신 Annotated로 선언합니다.

```python
# 에러가 발생하는 State
class BadState(TypedDict):
    checks: list

# 리듀서를 활용한 State
class GoodState(TypedDict):
    checks: Annotated[list, operator.add]
```

리듀서는 "같은 키에 여러 값이 들어올 때 어떻게 합칠지"를 정하는 함수입니다. Annotated[list, operator.add] 라고 선언하면 "이 키는 리스트이고, 여러 값이 오면 + 연산으로 이어 붙여라"라는 뜻이 됩니다.

Annotated[타입, 메타데이터] 는 파이썬 표준 타입 힌트 문법으로, "타입은 이건데 여기에 추가 정보를 붙여 둔다"는 의미입니다. 파이썬 자체는 이 메타데이터를 무시하지만 LangGraph가 이것을 읽어서 병합 규칙으로 씁니다.

operator.add 는 파이썬 표준 라이브러리의 함수로 operator.add(a, b) == a + b 입니다. 리스트에 + 를 쓰면 이어 붙기이므로, 결과적으로 두 노드의 리스트가 합쳐집니다.

섹션 5에서 쓴 MessagesState 에도 이미 리듀서가 붙어 있었습니다. 그래서 agent 노드가 {"messages": [새 메시지]} 를 반환할 때마다 이전 대화가 지워지지 않고 쌓였던 것입니다.

아래 셀에서 같은 그래프에 리듀서만 붙여 고쳐 보겠습니다.

**실습: 리듀서를 붙여 팬아웃/팬인 성공시키기 (키 불필요)**

```python
import operator
from typing import Annotated
from typing_extensions import TypedDict
from langgraph.graph import StateGraph, START, END

# ── 리듀서를 붙였다: "여러 값이 오면 + 로 이어 붙여라"
class GoodState(TypedDict):
    checks: Annotated[list, operator.add]
    report: str

def check_terms(state: GoodState):
    return {"checks": ["용어 정의 점검 완료"]}

def check_example(state: GoodState):
    return {"checks": ["예시 유무 점검 완료"]}

def check_summary(state: GoodState):
    return {"checks": ["요약 유무 점검 완료"]}

def aggregate(state: GoodState):
    # 팬인(fan-in) 노드: 갈라진 결과가 모두 도착한 뒤에 한 번 실행된다
    return {"report": f"총 {len(state['checks'])}건 수집 → " + " / ".join(sorted(state["checks"]))}

b = StateGraph(GoodState)
for name, fn in [("check_terms", check_terms),
                 ("check_example", check_example),
                 ("check_summary", check_summary),
                 ("aggregate", aggregate)]:
    b.add_node(name, fn)

for name in ["check_terms", "check_example", "check_summary"]:
    b.add_edge(START, name)          # 팬아웃(fan-out): 동시에 세 갈래로
    b.add_edge(name, "aggregate")    # 팬인(fan-in): 하나로 모임
b.add_edge("aggregate", END)

fan_graph = b.compile()
out = fan_graph.invoke({"checks": [], "report": ""})
print("checks:", out["checks"])
print("report :", out["report"])
print("\n── 그래프 구조 (실선 = 항상 실행) ──")
display(Image(fan_graph.get_graph().draw_mermaid_png()))
```

START에서 동시에 나간 세 노드의 결과가 모두 살아남았습니다. 그리고 aggregate 는 세 결과가 다 모인 뒤에 한 번만 실행되었습니다(report 에 "총 3건"이라고 찍혔습니다).

그래프 시각화 출력을 보면 __start__ 에서 나가는 세 화살표가 모두 **실선**입니다. 라우팅 그래프에서는 세 갈래가 점선이었고 하나만 실행됐습니다. 병렬화에서는 세 갈래가 실선이고 셋 다 실행됩니다.

이렇게 한 노드에서 여러 노드로 갈라지는 것을 **팬아웃(fan-out)**, 여러 노드가 하나로 모이는 것을 **팬인(fan-in)** 이라고 합니다. LangGraph는 팬인 노드를 모든 선행 노드가 끝날 때까지 기다린 뒤 실행합니다.

한 가지 주의할 점은 **순서가 보장되지 않는다는 것**입니다. checks 에 담긴 순서는 노드가 끝난 순서에 따라 달라질 수 있습니다. 그래서 위 코드에서 sorted() 로 정렬했습니다. 순서가 중요하다면 각 항목에 라벨을 함께 담아(("정확성", 점수, 코멘트) 처럼 튜플로) 나중에 정렬하거나 찾아 쓰는 방식을 택해야 합니다.

### LLM으로 병렬 평가하기

병렬화가 실제로 유용한 상황을 만들어 보겠습니다. 성의 없이 쓴 노트 초안을 **세 가지 다른 기준으로 동시에 평가**하는 것입니다. 정확성, 명확성, 예시를 각각 평가할 수 있도록 구성해보겠습니다.

**실습: 병렬화 — 노트 초안을 3개 축으로 동시 평가**

```python
import operator
from typing import Annotated
from typing_extensions import TypedDict
from pydantic import BaseModel, Field
from langchain_openai import ChatOpenAI
from langgraph.graph import StateGraph, START, END

llm = ChatOpenAI(model="gpt-4o-mini", temperature=0)

class Score(BaseModel):
    """한 축에 대한 평가 결과."""
    score: int = Field(description="1점(매우 나쁨)부터 5점(매우 좋음)까지의 정수")
    comment: str = Field(description="그 점수를 준 이유를 한 문장으로")

scorer = llm.with_structured_output(Score)

class NoteReviewState(TypedDict):
    note: str
    reviews: Annotated[list, operator.add]   # 리듀서 필수
    report: str

# ── 세 개의 평가 항목. 각 축은 자기 기준만 본다.
AXES = [
    ("accuracy", "정확성", "설명이 사실에 맞고, 조건이나 예외를 구체적으로 밝히는가"),
    ("clarity", "명확성", "처음 배우는 사람이 읽고 뜻을 이해할 수 있는가"),
    ("example", "예시", "개념을 뒷받침하는 구체적인 예시나 코드가 있는가"),
]

def make_reviewer(key, label, criterion):
    """평가 항목 하나를 담당하는 노드 함수를 만들어 돌려준다."""
    def reviewer(state: NoteReviewState):
        r = scorer.invoke(
            f"다음 강의 노트를 '{label}' 기준으로만 평가하라.\n"
            f"판단 기준: {criterion}\n\n[노트]\n{state['note']}"
        )
        return {"reviews": [(label, r.score, r.comment)]}
    return reviewer

def aggregate(state: NoteReviewState):
    rs = sorted(state["reviews"])
    avg = sum(s for _, s, _ in rs) / len(rs)
    lines = [f"- {label}: {score}점 — {comment}" for label, score, comment in rs]
    verdict = "게시 가능" if avg >= 4.0 else "보강 필요"
    return {"report": f"평균 {avg:.1f}점 → {verdict}\n" + "\n".join(lines)}

b = StateGraph(NoteReviewState)
for key, label, criterion in AXES:
    b.add_node(key, make_reviewer(key, label, criterion))
    b.add_edge(START, key)          # 팬아웃
    b.add_edge(key, "aggregate")    # 팬인
b.add_node("aggregate", aggregate)
b.add_edge("aggregate", END)

review_graph = b.compile()
display(Image(review_graph.get_graph().draw_mermaid_png()))
```

성의 없이 쓴 노트를 기반으로 병렬화 평가 패턴을 실행시켜 봅시다.

```python
# 성의 없이 쓴 노트 초안
BAD_NOTE = "리듀서는 값을 합치는 거다. 안 쓰면 에러 난다. 그냥 쓰면 된다."

out = review_graph.invoke({"note": BAD_NOTE, "reviews": [], "report": ""})
print("평가 대상:", BAD_NOTE)
print()
print(out["report"])
```

몇 점이 나왔나요? 성의 없이 쓴 노트에 대해서는 낮은 점수를 잘 부여하고 있는 것 같습니다. 각 코멘트를 살펴보면, 각자 자기의 평가 항목에 대한 이야기만 진행하고 있고, 점수도 항목마다 다르게 책정되어 있는 걸 알 수 있습니다. 각 평가가 독립적으로 실행되었다는 뜻이죠.

한 번의 LLM 호출로 정확성, 명확성, 예시 적절성을 한꺼번에 평가해달라고 요구할 수도 있지만 위와 같이 병렬화하면 가지는 장점이 있습니다.

- **주의력 분산 차단**: 하나의 프롬프트에 너무 많은 평가 기준을 넣으면 모델은 가장 눈에 띄는 결함에 쏠려 나머지 기준을 얕게 평가하게 됩니다. 평가 축을 분리하면 각 노드가 자신만의 관점에 집중합니다.
- **독립적인 기준 수정 및 확장**: "예시" 평가 항목의 프롬프트만 고치고 싶을 때 다른 평가 로직을 건드릴 필요가 없습니다. 새로운 평가 기준을 추가하는 것 역시 리스트에 노드 하나를 더하는 일로 간단해집니다.
- **속도 이득**: 3번의 연속 호출 보다 동시 실행으로 1번의 호출 시간 만에 결과가 반환됩니다.

추가적으로, 병렬화를 사용해서 같은 작업을 여러 번 실행해 결과를 모아 다수결이나 임계값으로 판정하는 하는 패턴을 만들 수도 있습니다. 코드 취약점 검토에서 서로 다른 프롬프트로 여러 번 훑어 하나라도 문제를 발견하면 플래그를 세우는 방식입니다.

![병렬화 — 여러 LLM 호출을 동시에 실행(팬아웃)한 뒤 결과를 하나로 합친다(팬인, Aggregator)](anthropic-parallelization-diagram.png)<span class="img-caption">그림: 병렬화 — 여러 LLM 호출을 동시에 실행(팬아웃)한 뒤 결과를 하나로 합친다(팬인, Aggregator) (출처: Anthropic, Building Effective Agents)</span>

> [!question]+ 객관식 퀴즈
> Q. 병렬로 실행되는 두 노드가 모두 State의 `reviews` 키에 값을 반환했을 때 `InvalidUpdateError`가 발생했습니다. 그 원인은 무엇일까요?
>
> 1. 병렬 실행은 LangGraph에서 지원하지 않는 기능이라 항상 에러가 난다
> 2. State의 기본 병합 규칙이 '덮어쓰기'이므로, 같은 스텝에 같은 키로 두 값이 오면 무엇을 남길지 정할 수 없어 LangGraph가 임의 선택 대신 에러를 낸다
> 3. 두 노드가 같은 이름의 함수를 쓰고 있어 이름 충돌이 발생했다
> 4. 리스트 타입은 State에서 사용할 수 없고 문자열이나 정수만 쓸 수 있다

### 🚀 더 해보기 — 병렬화의 변형 만들어 보기

**1. 평가 축을 추가하고 가중 평균 내기**

- 접근 힌트: AXES 에 ("length", "분량", "복습 문항을 만들 만큼 내용이 충분한가") 를 추가해 보세요. 노드 코드는 한 줄도 고칠 필요가 없습니다. 그다음 aggregate 에서 정확성에 2배 가중치를 주도록 바꿔 보세요.
- 성공하면: AXES 리스트에 튜플 하나를 더하는 것만으로 평가 축이 늘어나는 것을 확인합니다. 관심사를 데이터로 분리해 두면 확장이 얼마나 쉬워지는지 체감할 수 있습니다.

**2. 투표(voting) 방식으로 바꿔 보기**

- 접근 힌트: 같은 기준으로 temperature=1.0 인 평가자 5개를 병렬로 돌리고, 3개 이상이 3점 미만을 주면 '보강 필요'로 판정하게 만들어 보세요. AXES 대신 range(5) 로 노드를 만들면 됩니다.
- 성공하면: 같은 입력에 같은 프롬프트인데도 점수가 흔들리는 것을 보게 됩니다. LLM 판정의 분산을 직접 관찰하고, 여러 번 물어 다수결로 안정화하는 기법의 효과를 확인할 수 있습니다.

**3. 사실 검증 축을 붙여 보기**

- 접근 힌트: 본문에서 정확성 평가자가 리듀서를 '일반적인 집계 함수'로 설명한 것을 보았습니다. 평가자에게 **근거 노트를 함께 주고** "이 근거에 어긋나는 서술이 있는가"만 판정하는 축을 추가해 보세요. NOTE_DB["N01"]["body"] 를 근거로 쓸 수 있습니다.
- 성공하면: 같은 LLM인데도 근거를 주면 판정이 달라지는 것을 확인합니다. 그리고 이것이 마지막 프로젝트에서 '근거를 지문에서 그대로 인용하게 만드는' 설계로 이어지는 이유를 이해하게 됩니다.

**4. 실행 시간을 측정해 이득을 확인하기**

- 접근 힌트: time.perf_counter() 로 병렬 그래프의 소요 시간을 재고, 같은 세 평가를 for 문으로 순차 실행한 시간과 비교해 보세요. 그리고 response.usage_metadata 로 토큰 사용량도 함께 비교하세요.
- 성공하면: '시간은 줄지만 비용은 줄지 않는다'는 본문의 주장을 여러분의 환경에서 숫자로 검증할 수 있습니다. 이런 측정 습관은 실무에서 최적화 결정을 내릴 때의 근거가 됩니다.

### 📚 병렬화와 리듀서 참고 자료

- [Anthropic, Building Effective Agents](https://www.anthropic.com/engineering/building-effective-agents)
- [LangGraph 오류 레퍼런스 — INVALID_CONCURRENT_GRAPH_UPDATE](https://docs.langchain.com/oss/python/langgraph/errors/INVALID_CONCURRENT_GRAPH_UPDATE)
- [LangGraph — Graph API: State reducers](https://docs.langchain.com/oss/python/langgraph/graph-api) — operator.add 외에 직접 만든 함수를 리듀서로 쓰는 방법

---

## 전체 커리큘럼 구성표

|  강좌 번호   | 강좌명                                     |
| :------: | :--------------------------------------- |
|  **1강**  | [[AT01-01 LangGraph로 에이전트 워크플로 설계하기]]           |
|  **2강**  | [[AT01-02 에이전트의 재료 — 도구, 지식 증강, 메모리, 그리고 프레임워크]]           |
|  **3강**  | [[AT01-03 LangGraph의 세 가지 재료 — State, Node, Edge]]           |
|  **4강**  | [[AT01-04 조건부 엣지로 분기 만들고 시각화 하기]]           |
|  **5강**  | [[AT01-05 에이전트에게 도구(Tool)를 쥐여주기]]           |
|  **6강**  | [[AT01-06 패턴 ① 프롬프트 체이닝 (Prompt Chaining)]]           |
|  **7강**  | [[AT01-07 패턴 ② 라우팅 (Routing)]]           |
|  **8강**  | **8강. 패턴 ③ 병렬화 (Parallelization) (현재)**           |
|  **9강**  | [[AT01-09 패턴 ④ 오케스트레이터-워커 (Orchestrator-Worker)]]           |
|  **10강**  | [[AT01-10 패턴 ⑤ 평가자-최적화자 (Evaluator-Optimizer)]]           |
|  **11강**  | [[AT01-11 어떤 패턴을 고를 것인가 — 선택 기준과 LangSmith 관측]]           |
|  **12강**  | [[AT01-12 정리]]           |
