# [에이전트 팀 꾸리기] - 6. 패턴 ① 프롬프트 체이닝 (Prompt Chaining)

---

## 데이터 소개

지금부터 다섯 개 섹션에 걸쳐 워크플로 패턴을 하나씩 만듭니다. 각 패턴의 입력으로는 '모두런'이라는 가상의 온라인 강의 플랫폼의 강의 노트와 강의 녹취를 쓰겠습니다.

**데이터에 대한 안내**: 오늘은 LangGraph 입문 시간이므로, 데이터 수집·전처리에 시간을 쓰는 대신 그래프 구조에 집중하기 위해 가상의 더미 데이터로 진행합니다. '모두런'이라는 가상의 온라인 강의 플랫폼의 강의 노트와 강의 녹취 데이터로 진행하겠습니다.

**실습: 오늘 패턴 실습에 쓸 데이터 준비 (키 불필요)**

```python
# ── 오늘 패턴 실습에 사용할 예시 데이터 ────────────────────────────────
# 실제 서비스라면 이 부분이 CMS 조회나 강의 노트 저장소 API 호출이 된다.

NOTE_SAMPLES = [
    {"id": "S01", "label": "개념설명",
     "text": "State는 노드들이 공유하는 하나의 딕셔너리다. 노드는 자기가 바꾼 키만 돌려주며 "
             "LangGraph가 그것을 기존 State에 병합한다."},
    {"id": "S02", "label": "코드예제",
     "text": "builder.add_conditional_edges(\"review\", gate, {\"send\": END, \"polish\": \"polish\"}) "
             "처럼 딕셔너리를 주면 라우터가 돌려준 라벨이 노드 이름으로 매핑된다."},
    {"id": "S03", "label": "실습과제",
     "text": "직접 해보세요. measure 노드가 log를 반환하지 않도록 바꾼 뒤, 최종 log 값이 어떻게 "
             "달라지는지 실행 전에 예상하고 확인해 보세요."},
    {"id": "S04", "label": "개념설명",
     "text": "조건부 엣지는 라우터 함수의 반환값으로 다음 노드를 고른다. 라우터는 LLM일 필요가 없고 "
             "평범한 파이썬 함수여도 된다."},
    {"id": "S05", "label": "코드예제",
     "text": "class GoodState(TypedDict): checks: Annotated[list, operator.add] — 타입 힌트에 "
             "리듀서를 붙이면 값을 덮어쓰는 대신 이어 붙인다."},
    {"id": "S06", "label": "실습과제",
     "text": "과제. 검수 노드 하나를 정확성·공감·실행가능성 세 축의 병렬 평가로 바꾸고, 평균 4.0점 "
             "미만이면 재작성 노드로 보내도록 그래프를 수정하세요."},
]

# 프롬프트 체이닝 실습용 — 강의 녹취를 그대로 옮긴 구어체 초안 (핵심 용어가 흩어져 있다)
NOTE_DRAFT = (
    "자 그러니까 리듀서라는 게 뭐냐면요, 음 State에 있는 같은 키에다가 두 노드가 동시에 쓰려고 하면 "
    "LangGraph가 이걸 어떻게 합쳐야 할지를 몰라서 InvalidUpdateError를 던지거든요. "
    "그래서 그럴 때 타입 힌트에 operator.add를 붙여 주면 덮어쓰는 대신 리스트를 이어 붙여 줍니다. "
    "뭐 병렬로 뭔가를 할 때는 거의 항상 필요하다고 보시면 되고요, 안 붙이면 에러 난다는 것만 "
    "기억하시면 될 것 같습니다."
)

print(f"노트 표본 {len(NOTE_SAMPLES)}건 준비 완료")
print("라벨 분포:", {l: sum(1 for n in NOTE_SAMPLES if n["label"] == l)
                     for l in sorted({n["label"] for n in NOTE_SAMPLES})})
print(f"\n녹취 초안 {len(NOTE_DRAFT)}자:\n  {NOTE_DRAFT[:70]}…")
```

## 패턴 1. 프롬프트 체이닝 (Prompt Chaining)

첫 번째 패턴은 **프롬프트 체이닝(prompt chaining)** 입니다. 프롬프트 체이닝은 하나의 큰 작업을 **순서가 정해진 여러 단계로 쪼개고**, 각 단계의 LLM 호출이 이전 단계의 출력을 입력으로 받아 처리하는 워크플로입니다.

일단 한번 만들어볼까요? 구어체 강의 녹취를 받아서 ① 학습자가 읽을 3문장 노트로 다듬고, ② 핵심 용어가 다듬는 과정에서 사라지지 않았는지 프로그램으로 검사하고, ③ 통과하면 학습 목표 문장을 뽑는 파이프라인을 만들어봅시다.

**실습: 프롬프트 체이닝 — 다듬기 → 게이트 검사 → 학습 목표**

```python
from typing import Literal
from typing_extensions import TypedDict
from langchain_openai import ChatOpenAI
from langgraph.graph import StateGraph, START, END
from IPython.display import Image, display


# 모델 정의
llm = ChatOpenAI(model="gpt-4o-mini", temperature=0)

# 핵심 용어 정의
KEY_TERMS = ["리듀서", "InvalidUpdateError", "operator.add"]

# State 정의
class NoteChainState(TypedDict):
    draft: str        # 녹취 초안
    note: str         # 다듬은 학습 노트
    gate_log: str     # 게이트 검사 이력
    objectives: str   # 학습 목표 문장

# 노드 정의
def polish_note(state: NoteChainState):
    """① 구어체 녹취를 학습 노트로 다듬는다."""
    msg = llm.invoke(
        "다음 강의 녹취를 학습자가 읽을 3문장 이내의 노트로 다듬어라. "
        "기술 용어와 API 이름은 반드시 그대로 유지한다. 노트만 출력한다.\n\n"
        f"{state['draft']}"
    )
    return {"note": msg.content, "gate_log": "polish"}

# 라우터 함수 정의
def gate_check(state: NoteChainState) -> Literal["pass", "fail"]:
    """② 게이트: LLM이 아니라 '코드'가 검사한다. 핵심 용어가 살아남았는지 확인."""
    missing = [t for t in KEY_TERMS if t not in state["note"]]
    print(f"[gate] 핵심 용어 보존 검사 → {'통과' if not missing else f'실패(누락: {missing})'}")
    return "pass" if not missing else "fail"

def fix_note(state: NoteChainState):
    """②-b 게이트에서 걸렸을 때만 실행되는 보정 노드."""
    msg = llm.invoke(
        f"다음 노트에 이 용어들을 반드시 그대로 포함해 다시 써라: {', '.join(KEY_TERMS)}. "
        "3문장 이내로 쓰고 노트만 출력한다.\n\n"
        f"{state['note']}"
    )
    return {"note": msg.content, "gate_log": state["gate_log"] + " → fix"}

def write_objectives(state: NoteChainState):
    """③ 통과한 노트에서 학습 목표를 뽑는다."""
    msg = llm.invoke(
        "다음 학습 노트를 읽고 학습 목표를 '~할 수 있다' 형태로 2개 만들어라. "
        "목표만 번호를 붙여 출력한다.\n\n"
        f"{state['note']}"
    )
    return {"objectives": msg.content, "gate_log": state["gate_log"] + " → objectives"}
```

```python
# 그래프 정의
builder = StateGraph(NoteChainState)
builder.add_node("polish_note", polish_note)
builder.add_node("fix_note", fix_note)
builder.add_node("write_objectives", write_objectives)

builder.add_edge(START, "polish_note")
builder.add_conditional_edges("polish_note", gate_check,
                              {"pass": "write_objectives", "fail": "fix_note"})
builder.add_edge("fix_note", "write_objectives")   # 보정 후에는 검사 없이 다음 단계로
builder.add_edge("write_objectives", END)

note_chain = builder.compile()
display(Image(note_chain.get_graph().draw_mermaid_png()))
```

```python
result = note_chain.invoke({"draft": NOTE_DRAFT})
print("\n── 다듬은 학습 노트 ──")
print(result["note"])
print("\n── 학습 목표 ──")
print(result["objectives"])
print("\n── 지나온 경로 ──")
print(result["gate_log"])
```

프롬프트 체이닝은 이처럼 하나의 큰 작업을 여러 단계로 쪼개서 LLM 호출이 순서대로 처리할 수 있게 구성한 워크플로입니다.

만약 방금 한 일을 LLM 한 번의 호출로 처리한다고 상상해 보세요. 프롬프트는 "이 녹취를 3문장 노트로 다듬고, 핵심 용어가 빠지지 않았는지 확인하고, 확인됐으면 학습 목표까지 뽑아서 한꺼번에 출력해라"가 됩니다. 이런 프롬프트는 실제로 잘 동작하지 않습니다. 이유가 세 가지 있습니다.

**한 호출에 요구가 많아지면 모델이 일부를 흘립니다.** 세 가지를 시켰는데 두 개만 하거나, 학습 목표를 만드는 데 집중하다가 노트에서 용어를 빠뜨리는 일이 생깁니다. 요구사항이 늘어날수록 각 요구의 수행 품질은 떨어지는 경향이 있습니다.

**어디서 틀렸는지 알 수 없습니다.** 결과가 나쁠 때 다듬기가 문제인지 목표 추출이 문제인지 구분할 방법이 없습니다. 단계를 쪼개 두면 각 단계의 중간 산출물이 State에 남으므로, result["note"] 만 꺼내 보면 어느 단계까지 정상이었는지 즉시 알 수 있습니다.

**중간에 개입할 수 없습니다.** 한 번의 호출 안에서는 "용어가 살아 있는지"를 우리가 확인할 지점이 없습니다. 단계를 쪼개면 단계 사이가 열리고, 그 틈에 우리 코드를 끼워 넣을 수 있습니다.

프롬프트 체이닝 사이사이 게이트(gate)를 통해서 프로세스가 정상적으로 진행되고 있는지 확인할 수 있게 됩니다. LLM의 출력은 확률적이라 같은 프롬프트에도 다른 결과가 나올 수 있지만 게이트를 통해 파이썬 코드 기반의 결정적 검사를 끼워 넣어 전체 시스템의 신뢰도를 끌어올릴 수 있습니다.

하지만 현재 게이트가 검사하는 것은 누락뿐입니다. 용어가 들어 있기만 하면 그 용어를 틀리게 설명해도 통과합니다. 결정적 연산의 한계점도 존재할 수 있다는 사실을 알아두면 좋습니다.

![프롬프트 체이닝 — LLM 호출을 순서대로 잇고, 그 사이에 프로그램적 검사(Gate)를 둔다](anthropic-prompt-chaining-diagram.png)<span class="img-caption">그림: 프롬프트 체이닝 — LLM 호출을 순서대로 잇고, 그 사이에 프로그램적 검사(Gate)를 둔다 (출처: Anthropic, Building Effective Agents)</span>

위 공식 다이어그램과 우리가 만든 그래프를 비교해 보면 대응이 명확합니다. In 이 START, LLM Call 1 이 polish_note, Gate 가 gate_check, LLM Call 2 가 write_objectives 입니다. 다이어그램에서 게이트가 아래쪽 Exit 으로 빠지는 화살표는 우리 그래프의 fail → fix_note 경로에 해당합니다. 공식 그림에서는 그냥 종료해 버리지만, 우리는 종료 대신 보정 노드로 보냈습니다. 어느 쪽이 맞는지는 상황에 따라 다릅니다. 학습 자료 배포라면 용어가 뭉개진 노트를 내보내는 것보다 중단하고 사람을 부르는 편이 나을 수도 있습니다.

프롬프트 체이닝은 단계가 자연스럽게 순서를 갖고, 각 단계의 품질을 중간에 확인할 수 있는 작업에 적절합니다. 녹취를 노트로 다듬은 뒤 학습 목표를 뽑는 일, 문서의 개요를 먼저 잡고 그 개요가 기준을 충족하는지 확인한 다음 본문을 쓰는 일 같은 작업입니다.

반대로 잘 맞지 않는 경우도 있습니다. 단계가 서로 독립적이어서 순서가 의미 없다면 체이닝 대신 병렬화가 맞습니다. 어떤 단계를 밟을지가 입력에 따라 달라진다면 라우팅이 더 적절할 겁니다.

> [!question]+ 객관식 퀴즈
> Q. 프롬프트 체이닝에서 '게이트(gate)'를 두는 가장 핵심적인 이유는 무엇일까요?
>
> 1. LLM 호출 횟수를 줄여 비용을 절감하기 위해서
> 2. 확률적인 LLM 단계들 사이에 결정적인 검사를 끼워 넣어, 잘못된 중간 결과가 다음 단계로 흘러가지 않게 막기 위해서
> 3. 여러 단계를 동시에 실행해 전체 실행 시간을 줄이기 위해서
> 4. LLM이 스스로 다음 단계를 선택할 수 있게 자율성을 주기 위해서

> [!question]+ 주관식 퀴즈
> Q. 방금 실습에서 게이트를 LLM이 아니라 파이썬 문자열 검사로 구현했습니다. 그런데 "이 노트가 처음 배우는 사람에게 충분히 이해되는가?"를 검사하고 싶다면 이 방식으로는 어렵습니다. 왜 어려운지, 그리고 그런 검사는 어떻게 구현해야 할지 설명해 보세요.

### 🚀 더 해보기 — 체이닝을 확장하기

**1. 게이트를 일부러 실패시켜 보기**

- 접근 힌트: KEY_TERMS 에 "팬인" 처럼 녹취에 없는 용어를 하나 추가하거나, polish_note 의 프롬프트에서 '기술 용어와 API 이름은 반드시 그대로 유지한다'는 문장을 지워 보세요. 후자가 더 현실적인 실험입니다.
- 성공하면: gate_log 가 polish → fix → objectives 로 바뀌는 것을 확인하며, 조건부 엣지의 다른 분기가 실제로 살아 있음을 검증할 수 있습니다. 그리고 프롬프트의 한 문장을 지웠을 때 용어가 실제로 사라지는지 보면, 그 문장이 하던 일을 눈으로 확인할 수 있습니다.

**2. 체인을 한 단계 더 늘리기**

- 접근 힌트: 학습 목표 뒤에 make_title 노드를 추가해 노트 제목(20자 이내)까지 만들게 해 보세요. State에 title 키를 추가하고 write_objectives → make_title → END 로 엣지를 다시 연결하면 됩니다.
- 성공하면: State에 필드를 더하고 엣지를 다시 잇는 작업이 얼마나 국소적인지 확인할 수 있습니다. 기존 노드는 한 줄도 고치지 않고 단계를 늘릴 수 있다는 것이 이 구조의 장점입니다.

**3. 게이트를 통과할 때까지 반복하게 만들기**

- 접근 힌트: 지금은 fix_note 후 검사 없이 다음 단계로 갑니다. fix_note → polish_note 로 되돌려 게이트를 다시 통과해야 하게 바꿔 보세요. 단, 무한 루프를 막기 위해 State에 attempts 카운터를 두고 3회를 넘으면 END 로 가게 해야 합니다.
- 성공하면: 체이닝과 평가 루프의 경계를 직접 체험합니다. 섹션 10에서 배울 평가자-최적화자 패턴을 미리 절반쯤 구현하게 되며, '루프에는 반드시 상한이 필요하다'는 원칙을 코드로 익힙니다.

**4. 실제 데이터로 교체하기**

- 접근 힌트: NOTE_DRAFT 를 여러분이 수강한 강의의 실제 녹취나 회의록으로 바꿔 보세요. 공개 데이터로 하려면 한국어 뉴스 요약 데이터셋(예: Hugging Face daekeun-ml/naver-news-summarization-ko)을 datasets.load_dataset 으로 받아 본문 → 노트 → 목표 체인을 돌려 볼 수 있습니다. 데이터셋에 사람이 쓴 정답 요약이 있으니 비교도 가능합니다.
- 성공하면: 더미 데이터에서는 보이지 않던 문제들이 드러납니다. 본문이 길어 컨텍스트 윈도우와 비용이 문제가 되고, 무엇보다 KEY_TERMS 를 데이터마다 손으로 적을 수 없다는 벽에 부딪힙니다. 그러면 핵심 용어를 원문에서 자동으로 뽑는 단계가 체인 앞에 하나 더 필요해집니다 — 게이트의 기준 자체를 만들어 내는 문제로 넘어가는 것입니다.

### 📚 프롬프트 체이닝 참고 자료

- [Anthropic, Building Effective Agents](https://www.anthropic.com/engineering/building-effective-agents)
- [LangGraph 공식 문서 — Prompt chaining 예제](https://docs.langchain.com/oss/python/langgraph/workflows-agents)
- [LangChain — 구조화 출력(structured output) 가이드](https://docs.langchain.com/oss/python/langchain/structured-output)

---

## 전체 커리큘럼 구성표

|  강좌 번호   | 강좌명                                     |
| :------: | :--------------------------------------- |
|  **1강**  | [[AT01-01 LangGraph로 에이전트 워크플로 설계하기]]           |
|  **2강**  | [[AT01-02 에이전트의 재료 — 도구, 지식 증강, 메모리, 그리고 프레임워크]]           |
|  **3강**  | [[AT01-03 LangGraph의 세 가지 재료 — State, Node, Edge]]           |
|  **4강**  | [[AT01-04 조건부 엣지로 분기 만들고 시각화 하기]]           |
|  **5강**  | [[AT01-05 에이전트에게 도구(Tool)를 쥐여주기]]           |
|  **6강**  | **6강. 패턴 ① 프롬프트 체이닝 (Prompt Chaining) (현재)**           |
|  **7강**  | [[AT01-07 패턴 ② 라우팅 (Routing)]]           |
|  **8강**  | [[AT01-08 패턴 ③ 병렬화 (Parallelization)]]           |
|  **9강**  | [[AT01-09 패턴 ④ 오케스트레이터-워커 (Orchestrator-Worker)]]           |
|  **10강**  | [[AT01-10 패턴 ⑤ 평가자-최적화자 (Evaluator-Optimizer)]]           |
|  **11강**  | [[AT01-11 어떤 패턴을 고를 것인가 — 선택 기준과 LangSmith 관측]]           |
|  **12강**  | [[AT01-12 정리]]           |
