# [에이전트 팀 꾸리기] - 5. 에이전트에게 도구(Tool)를 쥐여주기

---

## 에이전트에게 도구 전달하기

자, 뼈대를 잡았으니 이제 진짜 LLM을 넣어보겠습니다. 이 섹션부터는 OpenAI API 키가 필요합니다. 아래 방법으로 API 키를 설정하고 진행해주세요.

**키 설정 방법**: 아래 세 가지 중 편한 것을 쓰세요.

- 로컬 주피터: 프로젝트 폴더에 .env 파일을 만들고 OPENAI_API_KEY=sk-... 를 적은 뒤 from dotenv import load_dotenv; load_dotenv() 를 실행합니다.
- Colab: 왼쪽 열쇠 아이콘(보안 비밀)에 OPENAI_API_KEY 를 등록하고 from google.colab import userdata 로 읽어 옵니다.
- 임시: os.environ["OPENAI_API_KEY"] = "sk-..." 로 직접 넣습니다. **다만 이 방식은 노트북에 키가 그대로 남으므로, 저장·공유 전에 반드시 지우세요.**

**환경: API 키 설정**

```python
import os

# 방법 A) .env 파일 사용 (권장)
try:
    from dotenv import load_dotenv
    load_dotenv()
except ImportError:
    pass

# 방법 B) Colab 보안 비밀 사용
# from google.colab import userdata
# os.environ["OPENAI_API_KEY"] = userdata.get("OPENAI_API_KEY")

# 방법 C) 직접 입력 (노트북 공유 전 반드시 삭제)
# os.environ["OPENAI_API_KEY"] = "sk-..."

os.environ.setdefault("OPENAI_API_KEY", "")
print("키 설정 여부:", "설정됨" if os.environ.get("OPENAI_API_KEY") else "비어 있음 — 위 방법 중 하나로 채워주세요")
```

### 도구 정의하기

LangChain에서 도구를 만드는 가장 간단한 방법은 함수에 `@tool` 데코레이터를 붙이는 것입니다.

여기서 가장 중요한 것은 **독스트링(docstring)** 입니다. LangChain은 이 독스트링과 타입 힌트를 읽어 **JSON 스키마**를 만들고, 그 스키마를 LLM에게 전달해서 "이런 도구가 있다"고 알려 줍니다. 독스트링이 부실하면 LLM은 그 도구를 언제 써야 할지 몰라서 아예 부르지 않거나, 엉뚱한 인자를 넣어 부릅니다. 도구 호출이 엉뚱하게 된다면 프롬프트보다 독스트링을 먼저 의심하셔야 합니다.

일단 한번 만들어서 확인해보겠습니다. 더미 강의 노트 저장소를 가지고 강의 노트의 메타데이터를 조회하는 도구와 강의 노트의 분량을 파악하는 도구를 만들어보겠습니다.

```python
# ── 오늘 실습용 더미 강의 노트 데이터
NOTE_DB = {
    "N01": {"title": "State와 리듀서", "sentences": 4,
            "body": ("LangGraph에서 State는 노드들이 공유하는 하나의 딕셔너리다. 노드는 자기가 바꾼 "
                     "키만 돌려주며 LangGraph가 그것을 기존 State에 병합한다. 같은 키에 두 노드가 "
                     "동시에 쓰면 병합 규칙이 없어 InvalidUpdateError가 난다. 이때 필요한 것이 "
                     "리듀서이며, 타입 힌트에 operator.add를 붙이면 값을 덮어쓰는 대신 "
                     "리스트를 이어 붙인다.")},
    "N02": {"title": "조건부 엣지", "sentences": 3,
            "body": ("조건부 엣지는 라우터 함수의 반환값으로 다음 노드를 고른다. 라우터는 State를 받아 "
                     "다음 노드의 이름 문자열을 돌려주는 평범한 파이썬 함수이며 LLM일 필요가 없다. "
                     "add_conditional_edges에 딕셔너리를 주면 라벨을 노드 이름으로 매핑한다.")},
    "N03": {"title": "오늘의 공지", "sentences": 2,
            "body": "오늘 강의는 여기까지입니다. 다음 시간에 이어서 하겠습니다."},
}

# 강의 노트의 길이에 따른 복습 문항 출제 가능 기준을 미리 정의합니다.
MIN_CHARS, MIN_SENTENCES = 200, 3     # 출제 가능 기준
```

```python
from langchain_core.tools import tool

# @tool 데코레이터로 도구를 정의합니다. 
@tool(parse_docstring=True)
def lookup_note(note_id: str) -> str:
    """강의 노트의 메타데이터를 조회한다. 제목, 글자 수, 문장 수를 돌려준다.

    노트로 복습 문항을 만들 수 있는지 판정하기 전에 반드시 먼저 호출해야 한다.

    Args:
        note_id: 'N01' 형식의 노트 ID
    """
    n = NOTE_DB.get(note_id)
    if not n:
        return f"노트 {note_id}를 찾을 수 없습니다."
    return (f"제목='{n['title']}', 글자 수={len(n['body'])}자, "
            f"문장 수={n['sentences']}문장")


@tool(parse_docstring=True)
def check_quiz_eligibility(char_count: int, sentence_count: int) -> str:
    """노트의 분량으로 복습 문항 출제 가능 여부를 판정한다.

    기준: 200자 이상이고 3문장 이상이면 출제 가능, 둘 중 하나라도 미달이면 출제 불가.

    Args:
        char_count: 노트 본문의 글자 수
        sentence_count: 노트 본문의 문장 수
    """
    short = []
    if char_count < MIN_CHARS:
        short.append(f"글자 수 부족({char_count}자 < {MIN_CHARS}자)")
    if sentence_count < MIN_SENTENCES:
        short.append(f"문장 수 부족({sentence_count}문장 < {MIN_SENTENCES}문장)")
    if short:
        return "출제 불가 — " + ", ".join(short)
    return f"출제 가능 — {char_count}자 / {sentence_count}문장으로 기준을 충족합니다."


# 정의한 도구들은 리스트로 관리합니다.
TOOLS = [lookup_note, check_quiz_eligibility]
```

```python
# ── LangChain이 독스트링과 타입 힌트로 만들어 낸 스키마를 직접 확인하기
import json
for t in TOOLS:
    print(f"■ 도구 이름: {t.name}")
    print(f"  설명(LLM이 읽는 부분): {t.description}")
    print(f"  인자 스키마: {json.dumps(t.args, ensure_ascii=False)}")
    print()
```

출력을 보면 lookup_note 의 인자 스키마가 `{"note_id": {"description": "'N01' 형식의 노트 ID", "title": "Note Id", "type": "string"}}` 처럼 만들어져 있습니다. 우리가 파이썬으로 쓴 `note_id: str` 이라는 타입 힌트가 `"type": "string"`이 되고, 독스트링의 Args: 항목이 그 인자의 description 이 되었습니다.

`@tool(parse_docstring=True)` 라고 쓴 것에 주의하세요. 이 인자를 주지 않으면 LangChain은 독스트링의 Args: 를 파싱하지 않고 인자 설명 없이 `{"title": ..., "type": ...}`만 만듭니다. 독스트링을 성실하게 써 두고도 LLM에게 전달되지 않는 일이 생기므로 확인해 둘 값입니다.

LLM은 우리 파이썬 코드를 보는 게 아니라 이 스키마만 봅니다. 그러니 도구를 설계할 때는 "LLM이 이 설명만 읽고 언제·어떻게 이 함수를 불러야 할지 알 수 있을까?"를 기준으로 삼아야 합니다. 방금 lookup_note 의 독스트링에 "복습 문항을 만들 수 있는지 판정하기 전에 반드시 먼저 호출해야 한다" 라는 문장을 넣은 것이 그런 설계입니다.

### LLM에게 도구 전달하기

정의된 도구를 실제 LLM으로 전달하는 방법을 살펴보겠습니다.

편의상 State를 따로 정의하지 않고 `MessagesState`를 사용했습니다. LangGraph가 제공하는 미리 만들어진 State로, `messages` 라는 키 하나를 갖고 있고 여기에 리듀서가 붙어 있어서 노드가 반환한 메시지가 덮어쓰이지 않고 뒤에 쌓입니다. 대화 이력을 다룰 때 매번 직접 만들지 않아도 되게 준비된 것입니다. 리듀서에 대해서는 뒷부분에서 한번 더 다루겠습니다.

```python
from langchain_openai import ChatOpenAI
from langchain_core.messages import HumanMessage, SystemMessage
from langgraph.graph import StateGraph, START, END, MessagesState
from langgraph.prebuilt import ToolNode, tools_condition
from IPython.display import Image, display


# 출력 변동을 줄이기 위해서 gpt-4o-mini 모델을 사용합니다
llm = ChatOpenAI(model="gpt-4o-mini", temperature=0)
# bind_tools()를 사용하여 llm에게 정의한 도구를 전달합니다. 
llm_with_tools = llm.bind_tools(TOOLS)

SYSTEM = SystemMessage(content=(
    "너는 온라인 강의 플랫폼 '모두런'의 학습 콘텐츠 담당자다. "
    "노트로 복습 문항을 만들 수 있느냐는 질문을 받으면 반드시 도구로 노트를 조회하고 "
    "출제 가능 여부를 판정한 뒤, 조회된 사실만 근거로 답한다. 추측하지 않는다."
))

# llm을 호출하는 agent 노드를 정의합니다. 
def agent(state: MessagesState):
    """LLM을 부르는 노드. 응답에 tool_calls가 있으면 도구 실행이 필요하다는 뜻이다."""
    return {"messages": [llm_with_tools.invoke([SYSTEM] + state["messages"])]}
```

`llm.bind_tools(TOOLS)`를 사용하여 모델에게 도구 목록을 붙입니다. 이렇게 만든 모델을 호출하면 응답 메시지에 `tool_calls` 라는 필드가 채워질 수 있습니다. 여기에 "어떤 도구를 어떤 인자로 불러 달라"는 요청이 담깁니다. 주의할 점은 `bind_tools`는 도구를 실행해 주지 않습니다. 도구 실행은 ToolNode가 진행합니다.

```python
# 그래프 정의
builder = StateGraph(MessagesState)
builder.add_node("agent", agent)
# ToolNode 추가
builder.add_node("tools", ToolNode(TOOLS))

builder.add_edge(START, "agent")
builder.add_conditional_edges("agent", tools_condition)   # tools 또는 END
builder.add_edge("tools", "agent")                         # 도구 결과를 들고 다시 LLM에게

tool_agent = builder.compile()
```

`ToolNode(TOOLS)`는 도구 실행을 대신해 주는 미리 만들어진 노드입니다. `tool_calls` 필드를 통해 전달된 도구를 실제로 실행하고, 결과를 `ToolMessage`로 만들어 메시지 목록에 붙여 줍니다. 여러 도구를 한꺼번에 요청받으면 모두 실행합니다.

조건부 엣지의 `tools_condition`은 미리 만들어진 라우터 함수입니다. 마지막 메시지에 `tool_calls`가 있으면 "tools"를, 없으면 END 를 반환합니다. 즉 "모델이 도구를 더 부르려 하면 도구 노드로, 최종 답을 냈으면 종료"라는 판단을 대신해 줍니다.

```python
# 그래프 시각화해보기
g = tool_agent.get_graph()
display(Image(g.draw_mermaid_png()))
```

이렇게 하면 agent가 도구를 호출하면 도구를 실행해서 다시 에이전트에게 전달하고, 에이전트가 도구를 더 호출하면 다시 도구로, 그렇지 않으면 종료하는 워크플로가 만들어졌습니다. 한번 실행을 해볼까요?

**실습: bind_tools + ToolNode + tools_condition로 ReAct 루프 만들기**

```python
question = "N01 노트로 복습 문항을 만들 수 있나요?"
result = tool_agent.invoke({"messages": [HumanMessage(content=question)]})

print("=== 메시지 이력 전체 ===")
for m in result["messages"]:
    kind = type(m).__name__
    calls = [c["name"] + str(c["args"]) for c in (getattr(m, "tool_calls", None) or [])]
    body = (m.content or "").strip()
    print(f"[{kind}] {body[:160]}")
    if calls:
        print(f"    ↳ 도구 요청: {calls}")
```

이 여섯 개의 메시지를 하나씩 읽어서 루프 실행을 관찰해보겠습니다.

**1~2번 메시지.** 사용자 질문을 받은 모델은 답변 텍스트 대신 tool_calls 에 lookup_note(note_id='N01') 요청만 내놓았습니다. 모델이 "이 질문에 답하려면 노트의 분량을 알아야 하는데 나는 모른다"고 판단한 것입니다. 이때 tools_condition 이 tool_calls 가 있음을 보고 "tools" 를 반환해 ToolNode 로 갔습니다.

**3번 메시지.** ToolNode 가 실제로 파이썬 함수를 실행하고, 그 반환 문자열을 ToolMessage 로 메시지 목록에 붙였습니다. 이제 상태에는 조회된 사실이 들어 있습니다.

**4번 메시지.** tools → agent 엣지를 따라 모델이 다시 호출되었습니다. 모델은 방금 붙은 ToolMessage 를 읽고 208자, 4문장이라는 값을 뽑아내 두 번째 도구의 인자로 넣었습니다. 개발자가 "조회한 뒤 판정하라"고 코드로 순서를 박아 넣지 않았는데도, 모델이 관찰 결과를 근거로 다음 행동을 스스로 정했습니다.

**5~6번 메시지.** 두번째 tool 사용을 통해 출제 가능 판정이 붙고, 모델이 마지막으로 호출되었습니다. 이번에는 더 부를 도구가 없으므로 tool_calls 가 비어 있고 최종 답변 텍스트가 나왔습니다. tools_condition 이 END 를 반환해 그래프가 종료되었습니다.

조회한 뒤 판정하라고 지시하지 않았는데도 LLM은 도구의 실행 결과를 보고 **다음 행동을 스스로 판단**하였습니다. 이처럼 LLM이 관찰과 행동을 반복해서 복잡한 문제를 해결할 수 있도록 하는 시스템을 **ReAct(Reasoning + Acting) 루프**라고 부릅니다. 이 과정에서 모델 내부 지식에만 의존하지 않고 도구를 호출하면서 정확한 기준선(200자, 3문장)을 파악하거나 외부 정보를 실시간으로 조회하게 만들어서 할루시네이션을 줄이는 효과를 기대할 수 있습니다.

![도구 사용 루프(ReAct) — agent와 tools 사이를 왕복하며, 더 부를 도구가 없을 때 종료된다](langgraph-react-tool-loop-diagram.jpg)<span class="img-caption">그림: 도구 사용 루프(ReAct) — agent와 tools 사이를 왕복하며, 더 부를 도구가 없을 때 종료된다</span>

### 프리빌트 에이전트

방금 우리가 조립한 agent ↔ tools 루프는 워낙 자주 쓰여서 라이브러리에 이미 준비되어 있습니다. `langchain.agents.create_agent`를 쓰면 같은 그래프를 한 줄로 만들 수 있습니다.

**실습: create_agent 프리빌트와 비교하기**

```python
from langchain.agents import create_agent

prebuilt = create_agent(
    model=ChatOpenAI(model="gpt-4o-mini", temperature=0),
    tools=TOOLS,
    system_prompt=SYSTEM.content,
)

out = prebuilt.invoke({"messages": [{"role": "user", "content": "N03 노트는 출제 가능한가요?"}]})
print("최종 답변:", out["messages"][-1].content)
print("\n호출된 도구 순서:", [c["name"] for m in out["messages"]
                              for c in (getattr(m, "tool_calls", None) or [])])
print("\n프리빌트가 만든 그래프 구조:")
display(Image(prebuilt.get_graph().draw_mermaid_png()))
```

프리빌트가 만든 그래프가 우리가 손으로 조립한 것과 사실상 같은 모양임을 확인할 수 있습니다. 그러면 왜 처음부터 이걸 쓰지 않고 직접 만들었을까요?

실무에서는 도구 호출 횟수에 제한을 두거나, 특정 도구를 부르기 전에 사람의 승인을 받는 등 다양한 형태의 워크플로를 구현하고 싶은 경우가 있습니다. 이런 경우들은 프리빌트 에이전트로 구현하기 어렵습니다. agent ↔ tools 라는 간단한 도구 루프 하나가 필요할 때는 create_agent를, 흐름의 통제가 필요할 때는 직접 그래프를 그리는 것이 유리한 선택입니다.

> [!question]+ 객관식 퀴즈
> Q. `llm.bind_tools(TOOLS)`로 만든 모델을 호출했더니 응답의 `content`는 비어 있고 `tool_calls`에 도구 요청만 담겨 있었다. 이 시점에서 도구는 실행되었는가?
>
> 1. 실행되었다. bind_tools가 도구를 자동으로 실행하고 결과까지 채워 준다
> 2. 실행되지 않았다. 모델은 '이 도구를 이렇게 불러 달라'고 요청만 했고, 실제 실행은 ToolNode 같은 코드가 해야 한다
> 3. 실행되었지만 결과는 tool_calls 안에 숨겨져 있어 직접 꺼내야 한다
> 4. 실행 여부는 모델에 따라 다르다. gpt-4o-mini는 자동 실행하고 다른 모델은 하지 않는다

> [!question]+ 주관식 퀴즈
> Q. 실습 결과에서 모델은 `lookup_note`를 먼저 부르고, 그 결과에 담긴 글자 수와 문장 수를 인자로 `check_quiz_eligibility`를 이어서 불렀습니다. 우리는 코드에 이 순서를 명시하지 않았는데 어떻게 이런 순서가 나왔을까요? 그리고 이 순서를 더 확실하게 지키도록 만들려면 무엇을 손볼 수 있을까요?

### 🚀 더 해보기 — 도구를 다루는 감각 키우기

**1. 독스트링을 망가뜨려 보기**

- 접근 힌트: lookup_note 의 독스트링을 """조회.""" 한 단어로 줄이고 같은 질문을 다시 던져 보세요(이때 parse_docstring=True 는 Args: 가 없으면 오류를 내므로 함께 지워야 합니다). 모델이 도구를 부르지 않거나 엉뚱한 인자를 넣는지 관찰하세요.
- 성공하면: 독스트링이 곧 LLM에게 전달되는 사양서라는 것을 몸으로 확인합니다. 앞으로 도구가 안 불릴 때 프롬프트보다 독스트링을 먼저 점검하는 습관이 생깁니다.

**2. 계산 도구를 추가하기**

- 접근 힌트: estimate_study_time(char_count) 처럼 노트의 예상 학습 시간을 분 단위로 계산하는 도구를 만들어 TOOLS 에 추가하고, "N01 노트를 읽는 데 몇 분 걸리나요?"를 물어 보세요. 분당 500자 같은 상수를 두고 나눗셈만 하면 됩니다.
- 성공하면: LLM이 약한 산술 계산을 코드에 맡기는 패턴을 익힙니다. 같은 질문을 도구 없이 물어본 결과와 비교하면 차이가 뚜렷합니다.

**3. 도구 호출 횟수에 상한 걸기**

- 접근 힌트: agent 노드에서 상태의 ToolMessage 개수를 세고, 5개를 넘으면 도구 없는 llm 으로 호출해 강제로 마무리하게 바꿔 보세요. sum(1 for m in state["messages"] if type(m).__name__ == "ToolMessage") 로 셀 수 있습니다.
- 성공하면: 자율적인 루프에 안전장치를 붙이는 방법을 배웁니다. 실무에서 비용 폭주와 무한 루프를 막는 가장 흔한 장치입니다.

**4. 사람의 승인을 끼워 넣기**

- 접근 힌트: 판정만 하는 대신 실제로 퀴즈를 학습자에게 공개하는 publish_quiz 도구를 만들고, ToolNode 앞에 input() 으로 확인을 받는 노드를 넣어 보세요. LangGraph의 interrupt 기능을 검색해 보면 정석적인 방법도 찾을 수 있습니다.
- 성공하면: 되돌리기 어려운 도구를 다루는 human-in-the-loop 설계를 체험합니다. 공개는 한 번 나가면 되돌릴 수 없으므로, 실서비스에 에이전트를 붙일 때 거의 항상 필요한 장치입니다.

### 📚 도구 사용 심화 자료

- [LangChain — Tools 개념 문서](https://docs.langchain.com/oss/python/langchain/tools)
- [LangGraph — ToolNode API 레퍼런스](https://reference.langchain.com/python/langgraph.prebuilt/tool_node/ToolNode)
- [LangChain — create_agent 문서](https://docs.langchain.com/oss/python/langchain/agents)
- [ReAct 논문 — Synergizing Reasoning and Acting in Language Models](https://arxiv.org/abs/2210.03629)

---

## 전체 커리큘럼 구성표

|  강좌 번호   | 강좌명                                     |
| :------: | :--------------------------------------- |
|  **1강**  | [[AT01-01 LangGraph로 에이전트 워크플로 설계하기]]           |
|  **2강**  | [[AT01-02 에이전트의 재료 — 도구, 지식 증강, 메모리, 그리고 프레임워크]]           |
|  **3강**  | [[AT01-03 LangGraph의 세 가지 재료 — State, Node, Edge]]           |
|  **4강**  | [[AT01-04 조건부 엣지로 분기 만들고 시각화 하기]]           |
|  **5강**  | **5강. 에이전트에게 도구(Tool)를 쥐여주기 (현재)**           |
|  **6강**  | [[AT01-06 패턴 ① 프롬프트 체이닝 (Prompt Chaining)]]           |
|  **7강**  | [[AT01-07 패턴 ② 라우팅 (Routing)]]           |
|  **8강**  | [[AT01-08 패턴 ③ 병렬화 (Parallelization)]]           |
|  **9강**  | [[AT01-09 패턴 ④ 오케스트레이터-워커 (Orchestrator-Worker)]]           |
|  **10강**  | [[AT01-10 패턴 ⑤ 평가자-최적화자 (Evaluator-Optimizer)]]           |
|  **11강**  | [[AT01-11 어떤 패턴을 고를 것인가 — 선택 기준과 LangSmith 관측]]           |
|  **12강**  | [[AT01-12 정리]]           |
