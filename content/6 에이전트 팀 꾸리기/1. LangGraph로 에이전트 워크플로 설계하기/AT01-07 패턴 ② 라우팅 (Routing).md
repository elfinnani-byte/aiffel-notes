# [에이전트 팀 꾸리기] - 7. 패턴 ② 라우팅 (Routing)

---

## 패턴 2. 라우팅 (Routing)

두 번째 패턴은 **라우팅(routing)** 입니다. 이미 조건부 엣지로 조건에 따라 동작을 다르게 하는 갈림길을 앞서 만들어 봤습니다. 라우팅 패턴이 다른 점은 **판단을 규칙이 아니라 LLM이 한다**는 점입니다.

강의 노트를 기반으로 각각의 유형에 따라 적절한 학습자료를 만드는 자동화를 생각해봅시다. 앞선 예제에서 조건은 "글자 수가 60자 미만이면"이라는 명확한 규칙이 있었습니다. 하지만 강의 노트를 개념설명·코드예제·실습과제로 나누는 일은 글자 수나 키워드로 되지 않습니다. "직접 해보세요"라는 말이 있으면 실습과제라고 규칙을 쓸 수 있을 것 같지만, "직접 해보면 알 수 있듯이 리듀서는…"으로 시작하는 개념설명도 있습니다. 텍스트의 의미를 이해해야 하는 일이므로 LLM에게 맡기는 것이 맞습니다.

### 구조화 출력(structured output)

문제는 LLM의 출력이 자유 텍스트라는 점입니다. "이 노트는 코드 예제로 보입니다"처럼 답하면 우리가 그 문자열을 파싱해서 분기해야 하고, 모델이 조금 다르게 표현하면 파싱이 깨집니다. 이것을 해결하는 기술이 **구조화 출력(structured output)** 입니다. 아래 셀에서 직접 보겠습니다.

**실습: with_structured_output으로 분류 결과를 구조화하기**

```python
from typing import Literal
from typing_extensions import TypedDict
from pydantic import BaseModel, Field
from langchain_openai import ChatOpenAI
from langchain_core.messages import HumanMessage, SystemMessage
from langgraph.graph import StateGraph, START, END
from IPython.display import Image, display

llm = ChatOpenAI(model="gpt-4o-mini", temperature=0)

# ── ① 원하는 출력의 '모양'을 pydantic 클래스로 선언한다
class Route(BaseModel):
    """강의 노트 유형 분류 결과."""
    category: Literal["개념설명", "코드예제", "실습과제"] = Field(
        description="노트 유형. 반드시 세 값 중 하나여야 한다.")
    reason: str = Field(description="그렇게 분류한 근거를 한 문장으로")

# ── ② with_structured_output으로 모델을 감싸면, 응답이 Route 인스턴스로 온다
router_llm = llm.with_structured_output(Route)

decision = router_llm.invoke([
    SystemMessage(content="강의 노트를 개념설명/코드예제/실습과제 중 하나로 분류하라."),
    HumanMessage(content="아래 코드를 직접 고쳐서 실행해 보고 결과를 예상과 비교하세요."),
])

print("반환 타입 :", type(decision).__name__)
print("category  :", decision.category)   # 문자열 파싱이 필요 없다
print("reason    :", decision.reason)
print("\n분기에 바로 쓸 수 있다:", decision.category in ["개념설명", "코드예제", "실습과제"])
```

"아래 코드를 직접 고쳐서 실행해 보고 결과를 예상과 비교하세요."라는 문장에 대해서 반환된 값을 확인할 수 있습니다. 입력 문장에서 '실습'이나 '과제' 같은 키워드가 없었음에도 LLM이 문장의 의미를 파악해서 '실습과제'로 잘 분류를 해준 것을 볼 수 있습니다.

이처럼 구조화 출력은 모델의 응답을 우리가 지정한 형식으로 강제하는 기능입니다. 내부적으로는 우리가 정의한 Route 클래스를 JSON 스키마로 변환해 모델에게 "이 스키마에 맞는 JSON을 만들어라"고 지시합니다. 앞에서 도구를 정의할 때 독스트링이 스키마가 되던 것과 같은 원리입니다. Field(description=...) 에 적은 설명이 모델에게 전달되므로, 여기를 성실하게 쓰는 것이 분류 품질에 직접 영향을 줍니다. 특히 Literal["개념설명", "코드예제", "실습과제"] 라는 타입 지정이 강력합니다. 이것은 스키마에 enum 으로 들어가서 모델이 이 세 값 외에는 내놓을 수 없게 만듭니다.

**왜 이것이 라우팅에 필수인가.** 라우터 함수는 노드 이름에 대응하는 값을 반환해야 합니다. 분류 결과가 enum 으로 제한되어 있으면 {"개념설명": "make_term_card", ...} 같은 매핑을 안심하고 쓸 수 있습니다. 만약 모델이 '개념 설명'(공백 포함)이나 'concept' 같은 변형을 내놓을 수 있다면 그 매핑은 KeyError 로 터집니다. 구조화 출력이 그 위험을 없애 줍니다.

### 라우팅 그래프 만들기

분류가 이루어지니 분류에 따라 처리 경로를 나눌 수 있습니다. '개념설명'에서는 용어 정의 카드를 뽑고, '코드예제'에는 한 줄 주석을 달고, '실습과제'에는 채점 기준을 만드는 자동화를 완성시켜봅시다. 아래 그래프는 노트를 분류한 뒤 용어 카드·코드 주석·채점 기준 세 노드 중 하나로 보내면 각 노드가 자기 전문 프롬프트로 학습 자료를 만드는 파이프라인을 구현했습니다.

**실습: 라우팅 그래프 구성 + 분류 정확도 측정**

```python
# 라우팅 그래프의 State 정의
class NoteRouteState(TypedDict):
    text: str
    category: str
    reason: str
    output: str

def classify(state: NoteRouteState):
    d = router_llm.invoke([
        SystemMessage(content=(
            "온라인 강의 플랫폼 '모두런'의 강의 노트를 분류하라.\n"
            "- 개념설명: 용어나 원리를 설명하는 서술\n"
            "- 코드예제: 코드나 API 사용법을 보여 주는 내용\n"
            "- 실습과제: 학습자에게 직접 해보라고 요구하는 내용")),
        HumanMessage(content=state["text"]),
    ])
    return {"category": d.category, "reason": d.reason}

# 유형별 후처리: 같은 노트라도 유형에 따라 만들어야 하는 학습 자료가 다르다
def make_term_card(state: NoteRouteState):
    r = llm.invoke("너는 학습 콘텐츠 담당자다. 아래 개념설명 노트에서 핵심 용어 하나를 골라 "
                    f"'용어 — 한 문장 정의' 형식의 카드를 2개 만들어라.\n\n노트: {state['text']}")
    return {"output": r.content}

def annotate_code(state: NoteRouteState):
    r = llm.invoke("너는 학습 콘텐츠 담당자다. 아래 코드예제 노트의 코드가 무슨 일을 하는지 "
                    f"한 줄 주석 형태로 2줄 설명하라.\n\n노트: {state['text']}")
    return {"output": r.content}

def draft_rubric(state: NoteRouteState):
    r = llm.invoke("너는 학습 콘텐츠 담당자다. 아래 실습과제 노트를 학습자가 제대로 수행했는지 "
                    f"확인할 채점 기준을 2개 만들어라.\n\n노트: {state['text']}")
    return {"output": r.content}

# 라우터 함수: 분류 결과를 노드 이름으로 번역한다
HANDLER = {"개념설명": "make_term_card", "코드예제": "annotate_code", "실습과제": "draft_rubric"}

def route_to_handler(state: NoteRouteState) -> Literal["make_term_card", "annotate_code", "draft_rubric"]:
    return HANDLER[state["category"]]

builder = StateGraph(NoteRouteState)
builder.add_node("classify", classify)
builder.add_node("make_term_card", make_term_card)
builder.add_node("annotate_code", annotate_code)
builder.add_node("draft_rubric", draft_rubric)

builder.add_edge(START, "classify")
builder.add_conditional_edges("classify", route_to_handler,
                              ["make_term_card", "annotate_code", "draft_rubric"])
for handler in ["make_term_card", "annotate_code", "draft_rubric"]:
    builder.add_edge(handler, END)

router_graph = builder.compile()
display(Image(router_graph.get_graph().draw_mermaid_png()))
```

### 평가하기

라우팅 그래프를 실행하고 준비해 둔 정답 라벨로 분류 정확도를 숫자로 측정해보겠습니다.

```python
# ── 준비한 6건을 모두 흘려보내고 정확도를 측정한다
hit = 0
for s in NOTE_SAMPLES:
    out = router_graph.invoke({"text": s["text"]})
    # router_graph의 출력 "category"와 원본 "label"이 일치하면 +1
    ok = out["category"] == s["label"]
    hit += ok
    print(f"{s['id']} 정답={s['label']:5s} 예측={out['category']:5s} {'✅' if ok else '❌'}")
    print(f"  원문: {s["text"]}")
    print(f"  근거: {out['reason']}")
    print(f"  산출: {out['output'][:70].replace(chr(10), ' ')}…\n")

print(f"분류 정확도: {hit}/{len(NOTE_SAMPLES)} = {hit/len(NOTE_SAMPLES):.0%}")
```

6개 중에 4건 밖에 맞히지 못했네요. 이렇게 분류가 틀렸을 때 모델이 왜 그렇게 판단했는지 원인 파악을 위해 reason 필드를 같이 받았습니다. 제대로 분류하지 못한 두 건의 근거를 확인해봅시다.

S02: "딕셔너리를 사용하여 라우터가 노드 이름을 매핑하는 원리를 설명하고 있기 때문입니다."
S05: "타입 힌트에 리듀서를 붙이면 값을 덮어쓰는 대신 이어 붙인다는 원리를 설명하고 있다."

모델은 "원리를 설명하고 있기 때문"이라는 이유로 개념설명을 골랐습니다. 원문과 함께 보니, 그렇게 LLM의 판단이 틀리지 않았던 것 같네요.

S02 원문: builder.add_conditional_edges("review", gate, {"send": END, "polish": "polish"}) 처럼 딕셔너리를 주면 라우터가 돌려준 라벨이 노드 이름으로 매핑된다.
S05 원문: class GoodState(TypedDict): checks: Annotated[list, operator.add] — 타입 힌트에 리듀서를 붙이면 값을 덮어쓰는 대신 이어 붙인다.

둘 다 코드를 담고 있지만, 그 코드가 무엇을 하는지 설명하는 문장이 함께 붙어 있습니다. 코드예제로도 볼 수 있고 개념설명으로도 볼 수 있습니다. '개념설명'과 '코드예제'는 배타적인 두 부류가 아니라 한 노트가 동시에 가질 수 있는 두 속성입니다. 배타적이지 않은 라벨로 단일 라벨 분류를 시키면, 프롬프트를 아무리 고쳐도 정확도는 올라가지 않습니다. 분류에서는 이런 식으로 LLM의 판단 실패 외에도 정확도를 낮추는 여러가지 원인이 있습니다. 이런 경우를 대비해서 판단의 원인 파악을 위한 reason 필드를 같이 확인하도록 워크플로를 구성하는 것이 좋습니다.

### 라우팅 패턴 정리

라우팅은 입력을 먼저 분류하고, 그 분류 결과에 따라 특화된 후속 처리 경로 하나로 보내는 워크플로입니다.

왜 라우팅 패턴이 필요할까요? 하나의 프롬프트로 모든 종류의 노트를 처리하려고 하면 조건이 점차 늘어나게 됩니다. "개념설명이면 용어 카드를 만들고, 코드예제면 주석을 달고, 실습과제면 채점 기준을 만들고, 단 코드가 있는 개념설명이면…" 이런 식으로 프롬프트가 길어지겠죠. 조건이 늘어날수록 프롬프트는 길어지고 서로 간섭하기 시작합니다.

라우팅 패턴으로 분류를 먼저 하면 각 노드는 자기 일만 알면 됩니다. 방금 코드에서 make_term_card 의 프롬프트에는 용어 카드 이야기만 있고, 채점 기준 이야기는 없습니다. 프롬프트가 짧고 명확해지므로 각각의 품질이 올라가고, 나중에 채점 기준 양식이 바뀌어도 draft_rubric 하나만 고치면 됩니다.

**비용 최적화 측면에서의 라우팅**
라우팅의 대상은 프롬프트만이 아닙니다. 모델도 라우팅할 수 있습니다. 쉽고 흔한 작업은 작고 값싼 모델로 보내고, 어렵고 드문 작업만 크고 비싼 모델로 보내는 것입니다. "이 코드에 주석 한 줄 달기"에 최고 성능 모델을 쓰는 것은 낭비입니다. 분류 자체는 값싼 모델로 하고, 분류 결과에 따라 처리 모델을 바꾸면 전체 품질을 크게 떨어뜨리지 않으면서 비용을 크게 줄일 수 있습니다.

![라우팅 — 입력을 분류(Router)한 뒤 여러 갈래 중 '하나'의 전문 경로로만 보낸다](anthropic-routing-diagram.png)<span class="img-caption">그림: 라우팅 — 입력을 분류(Router)한 뒤 여러 갈래 중 '하나'의 전문 경로로만 보낸다 (출처: Anthropic, Building Effective Agents)</span>

> [!question]+ 객관식 퀴즈
> Q. `with_structured_output`에 넘긴 스키마에서 `category: Literal["개념설명", "코드예제", "실습과제"]` 처럼 Literal을 쓰는 것이 라우팅에서 특히 중요한 이유는 무엇일까요?
>
> 1. Literal을 쓰면 LLM 호출 비용이 줄어든다
> 2. 모델이 그 세 값 외에는 반환할 수 없게 제약되어, 분류 결과를 노드 이름으로 매핑할 때 예상치 못한 값으로 KeyError가 나는 일을 구조적으로 막는다
> 3. Literal을 쓰면 모델이 분류 근거를 자동으로 함께 생성해 준다
> 4. Literal은 파이썬 타입 힌트일 뿐이므로 실행에는 아무 영향이 없고, 편집기 자동완성만 도와준다

### 🚀 더 해보기 — 라우팅을 실무 수준으로

**1. 라벨 정의를 배타적으로 고쳐 정확도를 되찾기 (가장 먼저 해볼 것)**

- 접근 힌트: 본문에서 진단한 문제를 직접 고치는 과제입니다. classify 의 시스템 프롬프트에 우선순위 규칙을 명시하세요 — 예를 들어 "코드나 API 호출이 본문에 포함되어 있으면 설명이 붙어 있더라도 코드예제로 분류한다. 학습자에게 무언가를 하라고 요구하면 코드가 있어도 실습과제로 분류한다"처럼 순서를 못 박습니다.
- 성공하면: 프롬프트 표현을 다듬는 것이 아니라 **판정 규칙의 우선순위를 정하는 것**이 진짜 해법이었음을 숫자로 확인합니다. 67%가 어떻게 변하는지 기록하고, 왜 이 개입이 효과가 있었는지 설명할 수 있으면 이 섹션의 핵심을 잡은 것입니다.

**2. 다중 라벨로 바꿔 보기**

- 접근 힌트: Route.category 를 categories: list[Literal["개념설명", "코드예제", "실습과제"]] 로 바꾸고, 해당하는 모든 처리 노드를 거치게 만들어 보세요. 라우터 함수가 문자열 하나 대신 **노드 이름 리스트**를 반환하면 LangGraph가 그 노드들을 동시에 실행합니다. 결과를 모으려면 리듀서가 필요합니다(섹션 8).
- 성공하면: '한 노트가 두 유형을 겸한다'는 문제를 라우팅이 아니라 병렬화로 푸는 방법을 배웁니다. 그리고 패턴 선택이 데이터의 성질에서 나온다는 것을 체감합니다 — 라벨이 배타적이면 라우팅, 겸할 수 있으면 병렬화입니다.

**3. 모델 라우팅으로 비용 최적화하기**

- 접근 힌트: Route 에 difficulty: Literal["쉬움", "어려움"] 필드를 추가하고, 각 처리 노드에서 difficulty 에 따라 gpt-4o-mini 와 더 큰 모델을 바꿔 쓰게 해 보세요. response.usage_metadata 로 토큰 사용량을 찍어 두면 비교가 됩니다.
- 성공하면: 같은 품질을 유지하면서 비용을 얼마나 줄일 수 있는지 숫자로 확인합니다. 실무에서 가장 자주 쓰이는 라우팅 활용법이며, 포트폴리오에 넣을 만한 결과가 나옵니다.

### 📚 라우팅과 구조화 출력 참고 자료

- [Anthropic, Building Effective Agents](https://www.anthropic.com/engineering/building-effective-agents)
- [LangChain — Structured output 가이드](https://docs.langchain.com/oss/python/langchain/structured-output) — with_structured_output 의 동작 방식과 pydantic 스키마 작성 요령
- [Pydantic 공식 문서 — Fields](https://docs.pydantic.dev/latest/concepts/fields/) — Field(description=...) 외에 기본값·검증 규칙을 주는 방법

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
|  **7강**  | **7강. 패턴 ② 라우팅 (Routing) (현재)**           |
|  **8강**  | [[AT01-08 패턴 ③ 병렬화 (Parallelization)]]           |
|  **9강**  | [[AT01-09 패턴 ④ 오케스트레이터-워커 (Orchestrator-Worker)]]           |
|  **10강**  | [[AT01-10 패턴 ⑤ 평가자-최적화자 (Evaluator-Optimizer)]]           |
|  **11강**  | [[AT01-11 어떤 패턴을 고를 것인가 — 선택 기준과 LangSmith 관측]]           |
|  **12강**  | [[AT01-12 정리]]           |
