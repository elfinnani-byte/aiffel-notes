# [에이전트 팀 꾸리기] - 10. 패턴 ⑤ 평가자-최적화자 (Evaluator-Optimizer)

---

## 패턴 5. 평가자-최적화자 (Evaluator-Optimizer)

마지막 패턴은 **평가자-최적화자(evaluator-optimizer)** 입니다. 지금까지 만든 네 패턴은 모두 데이터가 앞(START)에서 뒤(END)의 한 방향으로만 흘렀습니다. 이번에는 처음으로 **되돌아가는 순환 구조**를 가집니다. 초안을 쓰고 → 평가하고 → 피드백을 반영해 다시 쓰는 반복 과정을 구현할 때 흔히 사용되는 구조입니다.

하지만 되돌아가는 그래프에는 새로운 위험이 생깁니다. **끝나지 않을 수 있다는 것**입니다.

**실습: 종료 조건 없는 루프 → GraphRecursionError 보기 (키 불필요)**

```python
from typing_extensions import TypedDict
from langgraph.graph import StateGraph, START, END

class CountState(TypedDict):
    n: int

def increment(state: CountState):
    return {"n": state["n"] + 1}

b = StateGraph(CountState)
b.add_node("increment", increment)
b.add_edge(START, "increment")
b.add_edge("increment", "increment")   # 자기 자신으로 — 종료 조건이 없다!
loop_graph = b.compile()

try:
    loop_graph.invoke({"n": 0}, {"recursion_limit": 8})
except Exception as e:
    print(f"❌ {type(e).__name__}")
    print(str(e))
```

## 이 에러가 알려주는 안전장치

```
❌ GraphRecursionError
Recursion limit of 8 reached without hitting a stop condition.
You can increase the limit by setting the `recursion_limit` config key.
For troubleshooting, visit: https://docs.langchain.com/oss/python/langgraph/errors/GRAPH_RECURSION_LIMIT
```

LangGraph는 **스텝 수에 상한**을 두고 있습니다. 기본값은 25이고, invoke() 의 두 번째 인자로 {"recursion_limit": 8} 처럼 바꿀 수 있습니다. 상한에 닿으면 GraphRecursionError 를 냅니다. 이 에러가 그래프 실행이 무한 루프에 빠져 API 호출이 끝없이 일어나는 것을 막아 줍니다. 하지만 예외가 발생하면 시스템이 강제 종료되기 때문에 recursion_limit 에 의존하면 예외가 터지는 순간 그동안 재작성하며 다듬어온 중간 결과물까지 모두 날아가 버립니다.

그래서 실무에서는 그래프 안에 명시적인 종료 조건을 둡니다. 반복 횟수를 State에 세어 두고, 정해진 횟수를 넘으면 "포기하고 사람에게 넘긴다" 같은 정상 종료 경로로 빠지게 만듭니다. recursion_limit 은 그 설계가 실패했을 때를 대비한 이중 안전망으로 남겨 둡니다.

## 나쁜 노트 초안을 스스로 고쳐 쓰게 하기

이전에 활용했던 무성의한 노트 초안(BAD_NOTE)을 다시 활용해보겠습니다. 이번에는 평가만 하지 않고 평가 결과를 근거로 다시 쓰게 만드는 파이프라인을 만들어봅시다.

그래프 모양이 이전 패턴들과 다릅니다. START 에서 바로 평가자(evaluate)로 들어가고, 평가 결과에 따라 합격이면 종료, 불합격이면 최적화자(optimize)로 보내고, 최적화자는 다시 평가자로 돌아옵니다.

**실습: 평가자-최적화자 루프 (상한 + 정상 종료 경로)**

```python
from typing import Literal
from typing_extensions import TypedDict
from pydantic import BaseModel, Field
from langchain_openai import ChatOpenAI
from langgraph.graph import StateGraph, START, END

llm = ChatOpenAI(model="gpt-4o-mini", temperature=0)

# ── 검수 규칙: 코드로 검사하기 어려운 '판단'이 섞여 있어 LLM 게이트가 필요하다
RULES = ("① 핵심 용어를 한 문장으로 정의한다 "
         "② 그것이 왜 필요한지(없으면 무슨 일이 나는지) 설명한다 "
         "③ 구체적인 코드나 예시를 1개 이상 포함한다 "
         "④ 전체 5문장 이내다")

MAX_ROUNDS = 3   # 명시적 상한 — recursion_limit에 의존하지 않는다
BAD_NOTE = "리듀서는 값을 합치는 거다. 안 쓰면 에러 난다. 그냥 쓰면 된다."

class Verdict(BaseModel):
    """검수 결과."""
    grade: Literal["합격", "불합격"] = Field(description="그대로 학습자에게 게시해도 되는지")
    feedback: str = Field(description="불합격이면 어떤 규칙을 왜 못 지켰고 어떻게 고쳐야 하는지 구체적으로")

evaluator_llm = llm.with_structured_output(Verdict)

class OptState(TypedDict):
    topic: str
    note: str
    grade: str
    feedback: str
    rounds: int

def evaluate(state: OptState):
    """평가자: 규칙 충족 여부를 판정하고 피드백을 남긴다."""
    v = evaluator_llm.invoke(
        f"검수 규칙:\n{RULES}\n\n"
        "아래 강의 노트가 네 규칙을 모두 지켰는지 엄격하게 검수하라. 하나라도 어기면 불합격이다.\n\n"
        f"[노트]\n{state['note']}"
    )
    return {"grade": v.grade, "feedback": v.feedback}

def optimize(state: OptState):
    """최적화자: 피드백을 반영해 노트를 다시 쓴다."""
    msg = llm.invoke(
        f"노트 주제:\n{state['topic']}\n\n"
        f"현재 노트:\n{state['note']}\n\n"
        f"검수 피드백:\n{state['feedback']}\n\n"
        f"피드백을 반영해 노트를 다시 써라. 노트만 출력한다.\n검수 규칙:\n{RULES}"
    )
    return {"note": msg.content, "rounds": state.get("rounds", 0) + 1}

def hand_off(state: OptState):
    """정상 종료 경로: 상한에 도달했으면 사람에게 넘긴다 (예외를 던지지 않는다)."""
    return {"note": state["note"] + f"\n\n[자동 검수 {MAX_ROUNDS}회 실패 → 강사 확인 필요]"}

def decide(state: OptState) -> Literal["accept", "retry", "give_up"]:
    if state["grade"] == "합격":
        return "accept"
    if state.get("rounds", 0) >= MAX_ROUNDS:
        return "give_up"
    return "retry"

b = StateGraph(OptState)
b.add_node("evaluate", evaluate)
b.add_node("optimize", optimize)
b.add_node("hand_off", hand_off)

b.add_edge(START, "evaluate")   # 초안이 이미 있으니 평가부터
b.add_conditional_edges("evaluate", decide,
                        {"accept": END, "retry": "optimize", "give_up": "hand_off"})
b.add_edge("optimize", "evaluate")   # ← 되돌아가는 엣지: 여기가 루프다
b.add_edge("hand_off", END)

optimizer_graph = b.compile()
display(Image(optimizer_graph.get_graph().draw_mermaid_png()))
```

```python
print("초안:", BAD_NOTE, "\n")
print("── 루프가 도는 과정을 stream()으로 관찰 ──")

last = {}
for event in optimizer_graph.stream(
        {"topic": "리듀서는 무엇이고 왜 필요한가", "note": BAD_NOTE, "rounds": 0}):
    for node, update in event.items():
        print(f"[{node}]")
        for k, v in update.items():
            print(f"  {k}: {str(v)[:120]}")
        last.update(update)   # 마지막 상태를 모아 둔다

print("\n── 루프가 끝난 뒤의 최종 노트 ──")
print(last.get("note", BAD_NOTE))
```

evaluate 노드가 몇번 실행되었나요? 루프가 끝난 뒤의 최종 노트는 원하는 대로 개선이 되었나요?

실행할 때마다 다른 결과를 내겠지만, 초안이 무성의하게 적혔기 때문에 evaluate 노드는 1번 이상 실행되었을 겁니다. optimize 노드가 서술을 변경하였지만 원래 의도했던 대로 작성이 되지는 않았을지도 모릅니다. 평가자의 평가 기준에는 "① 용어 정의가 있는가", "② 예시 코드가 있는가"만 있었을 뿐, "원문 강의 노트의 기술적 주제를 유지하고 있는가"라는 기준이 없었기 때문입니다.

평가자-최적화자 루프는 작성된 글의 본질을 깊이 이해하며 고쳐 쓰는 것이 아니라, 오직 주어진 '평가 규칙 목록'을 충족시키는 방향으로 텍스트를 대체해 나갑니다. 무엇을 지켜야 하는지 적는 것만큼이나, '무엇을 잃지 말아야 하는가'를 검증 게이트로 걸어두는 설계가 필수적입니다.

### stream()과 invoke()의 차이

이번 실습에서 처음으로 invoke() 대신 stream() 을 썼습니다.

invoke() 는 그래프가 끝날 때까지 기다렸다가 **최종 State만** 돌려줍니다.
반면 stream() 은 **각 스텝이 끝날 때마다 그 스텝의 업데이트를 하나씩** 내보내는 제너레이터입니다. 그래서 노드가 실행되는 순서와 각 노드가 무엇을 바꿨는지를 실시간으로 볼 수 있습니다.

루프가 있는 그래프에서 stream() 이 특히 유용한 이유가 여기 있습니다. invoke() 로 실행하면 최종 결과만 보이니 루프가 몇 번 돌았는지, 각 라운드에서 피드백이 어떻게 바뀌었는지 알 수 없습니다. 방금 우리가 '주제가 바뀌었다'는 것을 발견할 수 있었던 것도 초안과 중간 산출물과 최종 결과를 나란히 볼 수 있었기 때문입니다. 디버깅할 때는 stream(), 서비스에서 결과만 필요할 때는 invoke() 를 쓰는 것이 보통입니다.

## 평가자-최적화자 패턴 정리

평가자-최적화자 패턴은 하나의 LLM 호출이 결과를 생성하고, 다른 LLM 호출이 그것을 평가하고 피드백을 주는 과정을 기준을 만족할 때까지 반복하는 워크플로입니다.

왜 이 패턴이 필요할까요? 사람이 좋은 문서를 쓸 때를 생각해 보세요. 한 번에 완성하지 않습니다. 초안을 쓰고, 읽어 보고, 어색한 곳을 고치고, 다시 읽습니다. 이 반복이 품질을 만듭니다. LLM도 마찬가지입니다. "잘 써라"라고 한 번 시키는 것보다, 쓰게 하고 → 구체적으로 무엇이 부족한지 지적하고 → 그 지적을 반영해 다시 쓰게 하는 것이 결과가 좋습니다.

하지만 평가 기준이 명확하지 않으면 피드백이 "더 좋게 써 보세요" 수준으로 겉돌고, 반복해도 나아지지 않습니다. 그래서 이 패턴을 쓸 때는 규칙을 먼저 명문화하는 것이 중요합니다. 실습에서 RULES 를 네 항목으로 구체적으로 적어 둔 것이 그 작업입니다. "좋은 노트를 써라"라고만 했다면 피드백도 모호했을 것입니다.

**대표적인 활용처.** 문학 번역이 좋은 예입니다. 번역가가 처음에 놓친 뉘앙스를 평가자 LLM이 지적할 수 있습니다. 복잡한 검색 작업도 그렇습니다. 여러 차례 검색과 분석이 필요할 때, 평가자가 "이 정보로는 부족하니 추가 검색이 필요하다"고 판정하는 역할을 맡습니다. 코드 작성에서 "테스트를 돌려 실패하면 고쳐 쓴다"는 루프도 같은 구조이며, 이때는 평가자가 LLM이 아니라 **테스트 실행 결과**입니다. 평가자가 결정적일 수 있다면 그편이 훨씬 낫습니다.

**비용에 주의하세요.** 이 패턴은 반복 횟수만큼 LLM 호출이 곱해집니다. 한 라운드에 생성 1회 + 평가 1회이므로, 3라운드면 6회입니다. 실습에서 MAX_ROUNDS = 3 으로 상한을 둔 것은 품질과 비용 사이에서 타협한 것입니다. 실무에서는 이 값을 정할 때 "라운드를 늘리면 실제로 품질이 올라가는가"를 측정해 봐야 합니다.

![평가자-최적화자 — 생성과 평가가 순환하며, 평가를 통과할 때까지 피드백을 반영해 다시 생성한다](anthropic-evaluator-optimizer-diagram.png)<span class="img-caption">그림: 평가자-최적화자 — 생성과 평가가 순환하며, 평가를 통과할 때까지 피드백을 반영해 다시 생성한다 (출처: Anthropic, Building Effective Agents)</span>

> [!question]+ 객관식 퀴즈
> Q. 루프가 있는 그래프에서 `recursion_limit`에만 의존해 반복을 멈추게 하는 것이 나쁜 설계인 이유는?
>
> 1. `recursion_limit`은 설정할 수 없는 고정값이라 바꿀 수 없다
> 2. 상한에 도달하면 `GraphRecursionError` 예외가 발생해 그때까지 만든 중간 결과를 전혀 받을 수 없다
> 3. `recursion_limit`은 노드 개수만 세고 루프 반복은 세지 않는다
> 4. `recursion_limit`을 설정하면 LangGraph가 병렬 실행을 비활성화한다

### 🚀 더 해보기 — 평가 루프를 실무 수준으로

**1. 주제 이탈을 막는 게이트를 붙이기 (가장 먼저 해볼 것)**

- 접근 힌트: 본문에서 최종 노트가 자바스크립트 reduce 로 옮겨 간 것을 보았습니다. evaluate 앞에 코드 게이트를 하나 두고, 원본에서 뽑은 핵심 용어(예: ["리듀서", "InvalidUpdateError"])가 노트에 남아 있는지 검사해 하나라도 없으면 LLM 평가 없이 바로 optimize 로 되돌리세요. 섹션 6의 gate_check 를 거의 그대로 가져올 수 있습니다.
- 성공하면: 규칙을 늘려도 막지 못했던 실패를 **코드 두 줄로** 막게 됩니다. 그리고 LLM 호출을 줄여 비용과 지연까지 낮아집니다. '코드로 되는 것은 코드로'라는 원칙이 품질과 비용 양쪽에서 값을 하는 것을 확인하는 과제입니다.

**2. 규칙을 모호하게 바꿔 비교하기**

- 접근 힌트: RULES 를 "좋은 학습 노트로 써라" 한 문장으로 바꾸고 같은 초안을 넣어 보세요. 라운드 수와 피드백의 구체성이 어떻게 달라지는지 위 퀴즈의 예상과 비교하세요.
- 성공하면: '평가 기준의 구체성이 곧 루프의 성능'이라는 것을 실험으로 확인합니다. 프롬프트 엔지니어링에서 가장 효과가 큰 개입이 어디인지 감을 잡게 됩니다.

**3. 라운드별 변화를 기록하기**

- 접근 힌트: State에 history: Annotated[list, operator.add] 를 추가해 매 라운드의 노트와 판정을 쌓고, 루프가 끝난 뒤 라운드별로 무엇이 어떻게 바뀌었는지 출력해 보세요. MAX_ROUNDS 를 5로 올려 실험하세요.
- 성공하면: '2~3라운드 이후 개선이 미미해진다'는 본문의 주장을 여러분의 데이터로 검증할 수 있습니다. 그리고 라운드가 늘어날수록 원본에서 점점 멀어지는지도 함께 관찰하세요 — 본문에서 본 주제 이탈이 라운드 수와 관계있는지 확인하는 실험이 됩니다.

**4. 평가자를 결정적인 것으로 바꿔 보기**

- 접근 힌트: LLM 평가자 대신 '테스트 실행'을 평가자로 쓰는 버전을 만들어 보세요. 예를 들어 LLM에게 간단한 파이썬 함수를 쓰게 하고, assert 테스트를 돌려 실패하면 그 에러 메시지를 피드백으로 넘겨 다시 쓰게 합니다.
- 성공하면: 평가자가 결정적일 때 이 패턴이 얼마나 강력해지는지 체감합니다. 코딩 에이전트가 실제로 이렇게 동작하며, 본문에서 본 'LLM 평가자가 사실 오류를 통과시키는' 문제가 사라지는 것을 확인할 수 있습니다.

### 📚 평가 루프와 스트리밍 참고 자료

- [LangGraph 오류 레퍼런스 — GRAPH_RECURSION_LIMIT](https://docs.langchain.com/oss/python/langgraph/errors/GRAPH_RECURSION_LIMIT)
- [LangGraph 공식 문서 — Streaming](https://docs.langchain.com/oss/python/langgraph/streaming) — stream() 의 여러 모드(values, updates, messages) 확인
- [Self-Refine 논문 — Iterative Refinement with Self-Feedback](https://arxiv.org/abs/2303.17651)

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
|  **8강**  | [[AT01-08 패턴 ③ 병렬화 (Parallelization)]]           |
|  **9강**  | [[AT01-09 패턴 ④ 오케스트레이터-워커 (Orchestrator-Worker)]]           |
|  **10강**  | **10강. 패턴 ⑤ 평가자-최적화자 (Evaluator-Optimizer) (현재)**           |
|  **11강**  | [[AT01-11 어떤 패턴을 고를 것인가 — 선택 기준과 LangSmith 관측]]           |
|  **12강**  | [[AT01-12 정리]]           |
