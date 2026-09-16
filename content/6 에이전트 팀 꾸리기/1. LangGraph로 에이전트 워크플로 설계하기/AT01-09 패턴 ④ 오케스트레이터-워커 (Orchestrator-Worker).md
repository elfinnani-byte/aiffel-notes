# [에이전트 팀 꾸리기] - 9. 패턴 ④ 오케스트레이터-워커 (Orchestrator-Worker)

---

## 패턴 4. 오케스트레이터-워커 (Orchestrator-Worker)

병렬화 패턴에는 숨은 전제가 하나 있었습니다. 실행할 하위 작업의 개수와 종류를 개발자가 코드 작성 시점에 미리 알고 있어야 한다는 점입니다. 앞선 예시에서 평가 항목은 세 개로 정해져 있었고, 그래서 노드도 세 개를 미리 만들어 엣지로 연결할 수 있었습니다.

그런데 이런 요구를 받으면 어떻게 할까요.

"이 주제로 학습 가이드 문서를 만들어 줘. 필요한 섹션 개수와 목차는 주제를 보고 알아서 짜줘"

주제에 따라 섹션이 3개가 될 수도, 7개가 될 수도 있습니다. 개발자가 미리 노드 개수를 고정할 수 없습니다. add_node 는 그래프를 조립할 때(컴파일 전에) 호출하는 함수이므로, 실행 중에 노드를 새로 추가하는 것은 불가능합니다.

이 문제를 푸는 것이 네 번째 패턴 **오케스트레이터-워커(orchestrator-worker)** 이고, 그 열쇠가 **Send** 입니다.

### Send의 동작 확인하기

Send 의 동작만 먼저 보겠습니다.

**실습: Send로 워커를 런타임에 동적 생성하기 (키 불필요)**

```python
import operator
from typing import Annotated
from typing_extensions import TypedDict
from langgraph.graph import StateGraph, START, END
from langgraph.types import Send

class OrchState(TypedDict):
    n: int                                  # 몇 개의 일을 만들지 (실행 시점에 정해진다)
    tasks: list                             # 오케스트레이터가 만든 작업 목록
    results: Annotated[list, operator.add]  # 워커들이 채운다 → 리듀서 필수

# Send 전용 상태 정의
class WorkerInput(TypedDict):
    task: str                               # 워커 하나가 받는 '자기 몫'
    results: Annotated[list, operator.add]

def orchestrator(state: OrchState):
    tasks = [f"작업-{i+1}" for i in range(state["n"])]
    print(f"[orchestrator] 작업 {len(tasks)}개 생성: {tasks}")
    return {"tasks": tasks}

def worker(state: WorkerInput):
    # 워커는 전체 State가 아니라 Send로 받은 '자기 몫'만 본다
    return {"results": [f"{state['task']} 처리 완료"]}

def assign_workers(state: OrchState):
    """라우터 자리에서 Send 객체 리스트를 반환한다 → 워커가 그 개수만큼 동시에 생성된다."""
    return [Send("worker", {"task": t}) for t in state["tasks"]]

def synthesize(state: OrchState):
    return {}

b = StateGraph(OrchState)
b.add_node("orchestrator", orchestrator)
b.add_node("worker", worker)
b.add_node("synthesize", synthesize)

b.add_edge(START, "orchestrator")
b.add_conditional_edges("orchestrator", assign_workers, ["worker"])
b.add_edge("worker", "synthesize")
b.add_edge("synthesize", END)

send_graph = b.compile()

for n in [3, 6]:
    out = send_graph.invoke({"n": n, "tasks": [], "results": []})
    print(f"  → 워커 {len(out['results'])}개 실행됨: {out['results']}\n")

print("── 그래프 구조: worker 노드는 '하나'로 그려진다 ──")
display(Image(send_graph.get_graph().draw_mermaid_png()))
```

**같은 그래프인데 첫 실행에서는 워커가 3번, 두 번째에서는 6번 실행**되었습니다. 그래프 구조는 전혀 바뀌지 않았습니다. 그런데도 실행되는 워커의 수가 달라졌습니다.

draw_mermaid() 출력을 보면 worker 노드가 **하나만** 그려져 있습니다. orchestrator -.-> worker 라는 점선 하나뿐입니다. 즉 그래프의 정적 구조에는 워커가 하나이고, 실행 시점에 그것이 필요한 만큼 **복제**되는 것입니다.

Send("노드이름", {전용상태})는 지정한 노드를 해당 파라미터로 동시 실행하라는 지시서입니다. 라우터 함수가 Send 객체의 리스트를 반환하면, LangGraph는 그래프 구조를 변경하지 않고 동일한 워커 노드를 리스트 길이만큼 복제하여 동시에 실행합니다.

이때 중요한 점은 Send로 전달되는 딕셔너리가 워커 전용의 독립된 상태(State)라는 것입니다. 위 코드에서는 각 worker 마다 WorkerInput 상태를 가집니다. 워커 노드는 메인 State 전체를 보지 못하고 자기가 맡은 단 하나의 섹션 정보만 받아서 일합니다. 이후 워커가 반환한 {"results": [...]} 는 메인 State로 병합됩니다.

- Map (Send): 중앙 오케스트레이터가 작업을 여러 갈래로 분해하여 독립된 워커들에게 뿌립니다.
- Reduce (리듀서): 개별 워커들이 완료한 결과물은 메인 State의 results 키로 올려져 하나로 합쳐집니다.

이 구조를 데이터 처리에서는 **맵-리듀스(map-reduce)** 라고 부릅니다. 작업을 여러 갈래로 펼쳐(map) 각각 처리한 뒤 하나로 접는(reduce) 방식입니다. Send 가 map이고 리듀서가 reduce입니다.

### 오케스트레이터-워커 패턴 활용하기

오케스트레이터가 주제를 보고 **필요한 섹션을 스스로 계획**하고, 워커들이 각 섹션을 동시에 작성한 뒤, 종합 노드가 하나의 문서로 합치는 파이프라인을 만들어보겠습니다.

**실습: 오케스트레이터-워커로 학습 가이드 동적 생성**

```python
import operator
from typing import Annotated
from typing_extensions import TypedDict
from pydantic import BaseModel, Field
from langchain_openai import ChatOpenAI
from langchain_core.messages import HumanMessage, SystemMessage
from langgraph.graph import StateGraph, START, END
from langgraph.types import Send

llm = ChatOpenAI(model="gpt-4o-mini", temperature=0)

class GuideSection(BaseModel):
    """학습 가이드 한 섹션의 설계."""
    heading: str = Field(description="섹션 제목")
    intent: str = Field(description="이 섹션에서 반드시 다뤄야 할 내용을 한 문장으로")

class GuidePlan(BaseModel):
    """학습 가이드 전체 계획."""
    sections: list[GuideSection] = Field(
        description="필요한 섹션 목록. 주제에 따라 개수를 스스로 정한다.")

planner = llm.with_structured_output(GuidePlan)

class GuideState(TypedDict):
    topic: str
    sections: list
    written: Annotated[list, operator.add]
    document: str

class GuideWorkerInput(TypedDict):
    section: GuideSection
    written: Annotated[list, operator.add]

def orchestrator(state: GuideState):
    """주제를 보고 필요한 섹션을 스스로 계획한다. 개수는 정해주지 않는다."""
    plan = planner.invoke([
        SystemMessage(content="온라인 강의 플랫폼 '모두런'의 학습 가이드 문서 섹션을 계획하라. "
                              "주제를 충실히 덮는 데 필요한 만큼만 만들되 3~5개로 한다."),
        HumanMessage(content=f"주제: {state['topic']}"),
    ])
    print(f"[orchestrator] {len(plan.sections)}개 섹션 계획:")
    for s in plan.sections:
        print(f"  · {s.heading}")
    return {"sections": plan.sections}

def write_section(state: GuideWorkerInput):
    """워커: 자기가 받은 섹션 하나만 작성한다."""
    s = state["section"]
    msg = llm.invoke([
        SystemMessage(content="학습 가이드의 한 섹션을 마크다운으로 작성한다. 제목은 '### '로 시작하고 "
                              "본문은 3문장 이내로 쓴다. 서론이나 맺음말 없이 바로 작성한다."),
        HumanMessage(content=f"제목: {s.heading}\n담을 내용: {s.intent}"),
    ])
    return {"written": [msg.content]}

def assign_writers(state: GuideState):
    """계획된 섹션 수만큼 워커를 동시에 띄운다."""
    return [Send("write_section", {"section": s}) for s in state["sections"]]

def synthesize(state: GuideState):
    return {"document": "\n\n".join(state["written"])}

b = StateGraph(GuideState)
b.add_node("orchestrator", orchestrator)
b.add_node("write_section", write_section)
b.add_node("synthesize", synthesize)

b.add_edge(START, "orchestrator")
b.add_conditional_edges("orchestrator", assign_writers, ["write_section"])
b.add_edge("write_section", "synthesize")
b.add_edge("synthesize", END)

guide_graph = b.compile()
out = guide_graph.invoke({"topic": "LangGraph의 리듀서와 병렬 실행",
                          "sections": [], "written": [], "document": ""})

print(f"\n[synthesize] 워커 산출물 {len(out['written'])}개를 하나의 문서로 합침\n")
print(out["document"])
```

프롬프트에 "3~5개로 한다"는 범위만 주었고, 오케스트레이터가 "LangGraph의 리듀서와 병렬 실행"이라는 주제를 보고 다섯 섹션이 필요하다고 판단했습니다. 명시적인 섹션의 개수를 주지 않았기 때문에 주제를 "조건부 엣지와 라우터"로 바꾸면 다른 개수, 다른 섹션이 나옵니다.

하지만 실제 출력물을 자세히 읽어보면 한계가 드러나는 부분들이 있습니다.

1. 지식의 환각(Hallucination): 근거 자료(강의 노트) 없이 제목만 받은 워커 LLM이 'LangGraph의 리듀서' 대신 학습 데이터에서 훨씬 흔한 'JavaScript Redux의 useReducer'를 설명해 버립니다.
2. 작업의 중복: 오케스트레이터가 계획을 세울 때 경계를 명확히 해주지 않으면 1번 워커와 2번 워커가 거의 비슷한 내용을 중복해서 작성합니다.
3. 워커 간의 격리: 워커들은 오직 자기 몫만 바라보므로 타 워커가 무슨 내용을 썼는지 알 수 없습니다.

## 병렬화와 오케스트레이터-워커의 차이

두 패턴은 그림이 비슷합니다. 여러 갈래로 갈라져 동시에 실행되고 하나로 모입니다.
결정적 차이는 **하위 작업을 누가, 언제 정하는가**입니다.

- **병렬화**에서는 개발자가 하위 작업을 **미리** 정합니다. 평가 축 세 개는 코드에 적혀 있었고, 어떤 입력이 들어와도 항상 그 세 개입니다. 그래서 노드도 세 개를 미리 만들었습니다.
- **오케스트레이터-워커**에서는 LLM이 하위 작업을 **실행 중에** 정합니다. 개수도 내용도 입력에 따라 달라집니다. 그래서 노드를 미리 만들 수 없고 Send 로 복제합니다.

Anthropic은 이 차이를 **유연성(flexibility)** 이라고 표현합니다. 구조는 비슷하지만 "하위 작업이 미리 정의되는 것이 아니라 특정 입력에 따라 오케스트레이터가 결정한다"는 것입니다.

언제 이 패턴이 필요할까요? 필요한 하위 작업을 예측할 수 없는 복잡한 작업입니다. 대표적인 예가 **코딩 작업**입니다. "이 기능을 추가해 줘"라는 요청에 몇 개의 파일을 고쳐야 하고 각 파일에서 무엇을 바꿔야 하는지는 요청마다 다릅니다. 오케스트레이터가 "이 세 파일을 이렇게 고쳐야 한다"고 계획하고, 워커들이 각 파일을 맡는 구조가 자연스럽습니다. 또 하나는 **검색 작업**입니다. 여러 출처에서 정보를 모아야 하는데, 어떤 출처를 몇 개 봐야 할지는 질문에 따라 달라집니다.

**대가도 분명합니다.** 오케스트레이터의 계획이 나쁘면 그 아래 모든 워커의 결과가 나빠집니다. 계획 단계가 단일 실패 지점이 되고, 워커 수를 LLM이 정하므로 비용도 예측하기 어렵습니다. 방금 본 중복과 환각처럼 워커 간 조율 부재와 근거 부재도 감수해야 합니다.

![오케스트레이터-워커 — 중앙 LLM이 작업을 동적으로 분해해 워커에게 위임하고 결과를 종합한다](anthropic-orchestrator-worker-diagram.png)<span class="img-caption">그림: 오케스트레이터-워커 — 중앙 LLM이 작업을 동적으로 분해해 워커에게 위임하고 결과를 종합한다 (출처: Anthropic, Building Effective Agents)</span>

> [!question]+ 객관식 퀴즈
> Q. `Send("write_section", {"section": s})`의 두 번째 인자로 넘긴 딕셔너리는 워커 노드에서 어떻게 보이는가?
>
> 1. 전체 State에 병합된 뒤 워커가 전체 State를 받는다
> 2. 그 워커 실행에만 전달되는 전용 상태이며, 워커는 전체 State의 다른 키(topic, sections 등)를 볼 수 없다
> 3. 무시된다. 워커는 항상 메인 State를 그대로 받는다
> 4. 워커의 반환값을 담을 빈 그릇으로만 쓰이고 값은 전달되지 않는다

> [!question]+ 주관식 퀴즈
> Q. 병렬화 그래프와 오케스트레이터-워커 그래프는 그림이 비슷합니다. 두 패턴의 결정적 차이가 무엇인지, 그리고 '평가 축 3개로 노트를 검수하는 일'에는 왜 오케스트레이터-워커가 과한 선택인지 설명해 보세요.

### 🚀 더 해보기 — 오케스트레이터-워커 다듬기

**1. 워커에게 근거를 주어 환각을 막기 (가장 먼저 해볼 것)**

- 접근 힌트: 섹션 5에서 만든 NOTE_DB 를 근거 자료로 쓰세요. Send 로 워커에게 section 과 함께 NOTE_DB["N01"]["body"] 를 넘기고, 워커 프롬프트에 '아래 근거 노트에 없는 내용은 절대 쓰지 말고, 근거에 없으면 "이 노트에서 다루지 않음"이라고 쓰라'를 추가하세요.
- 성공하면: useReducer 훅과 ParallelExecutor 같은 자바스크립트·가짜 API가 사라지는 것을 확인할 수 있습니다. 환각을 줄이는 가장 확실한 방법이 프롬프트를 정교하게 다듬는 것이 아니라 **근거를 제공하는 것**이라는 사실을 체감하게 됩니다. 마지막 프로젝트의 출제기가 쓰는 방식과 같습니다.

**2. 근거를 벗어났는지 코드로 검사하기 (1번의 다음 단계)**

- 접근 힌트: 워커에게 근거를 준 것만으로는 지켰는지 알 수 없습니다. 워커가 evidence 필드에 근거 노트의 한 구절을 **그대로 복사**해 넣게 만들고, synthesize 에서 evidence in note_body 를 검사해 통과하지 못한 섹션을 표시해 보세요.
- 성공하면: '근거에 없는 내용은 쓰지 말라'는 지시를 **검증 가능한 형태**로 바꾸는 방법을 배웁니다. 지시만으로는 확인할 수 없고 검사가 있어야 확인된다는 것이 이 과제의 요점이며, 마지막 프로젝트의 검사 ④가 정확히 이것입니다.

**3. 워커 간 중복을 없애 보기**

- 접근 힌트: 오케스트레이터의 프롬프트에 '각 섹션은 다른 섹션과 내용이 겹치지 않도록 담당 범위를 명확히 나눈다'를 추가하고, GuideSection 에 exclude: str(이 섹션에서 다루지 않을 것) 필드를 넣어 워커 프롬프트에 전달해 보세요.
- 성공하면: 실습에서 본 '리듀서란 무엇인가 / 리듀서 사용법' 중복이 줄어드는지 확인합니다. 워커 격리의 부작용을 계획 단계의 설계로 보완하는 방법을 배우게 됩니다.

**4. synthesize를 LLM으로 바꿔 보기**

- 접근 힌트: 지금은 "\n\n".join() 으로 단순히 이어 붙입니다. 이것을 LLM 호출로 바꿔 중복 문장을 정리하고 전체 톤을 통일한 최종 문서를 만들게 해 보세요.
- 성공하면: 맵-리듀스의 'reduce' 단계에도 LLM을 쓸 수 있다는 것을 확인합니다. 다만 문서가 길어지면 컨텍스트 윈도우 한계에 부딪히므로, 계층적으로 접는 방법이 필요해진다는 점도 함께 알게 됩니다.

**5. 워커 수에 상한을 두기**

- 접근 힌트: assign_writers 에서 state["sections"][:4] 처럼 상한을 걸고, 잘린 섹션 수를 State에 기록해 synthesize 에서 경고를 붙이게 만들어 보세요.
- 성공하면: LLM이 워커 수를 정하는 구조에서 비용이 폭주할 수 있다는 위험을 코드로 막는 방법을 익힙니다. 오케스트레이터가 20개 섹션을 계획하면 LLM 호출이 21번 일어난다는 점을 생각해 보세요.

### 📚 Send와 맵-리듀스 참고 자료

- [Anthropic, Building Effective Agents](https://www.anthropic.com/engineering/building-effective-agents)
- [LangGraph API 레퍼런스 — Send](https://reference.langchain.com/python/langgraph/types/Send)
- [LangGraph 공식 문서 — Orchestrator-worker 예제](https://docs.langchain.com/oss/python/langgraph/workflows-agents) — 보고서를 섹션별로 나눠 쓰는 공식 예제

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
|  **9강**  | **9강. 패턴 ④ 오케스트레이터-워커 (Orchestrator-Worker) (현재)**           |
|  **10강**  | [[AT01-10 패턴 ⑤ 평가자-최적화자 (Evaluator-Optimizer)]]           |
|  **11강**  | [[AT01-11 어떤 패턴을 고를 것인가 — 선택 기준과 LangSmith 관측]]           |
|  **12강**  | [[AT01-12 정리]]           |
