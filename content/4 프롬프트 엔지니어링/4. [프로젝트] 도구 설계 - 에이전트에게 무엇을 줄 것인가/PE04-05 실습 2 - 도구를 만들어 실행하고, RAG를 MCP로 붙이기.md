# [프롬프트 엔지니어링] - 5. 실습 2 - 도구를 만들어 실행하고, RAG를 MCP로 붙이기

---

## 실습 2 - 도구를 만들어 실행하고, RAG를 MCP로 붙이기

재고 7개라는 답은 누가 만들었을까요. 모델의 요청, 프로그램의 실행, 결과가 답에 쓰인 순간을 실행 원문에서 한 단계씩 가려냅니다.

### 요청과 실행을 분리해 관찰합니다

아래는 2026-08-27에 실측한 환경입니다. `qwen3.5:2b` 로컬 서버를 `ollama serve`로 실행하고, 온도는 0, `reasoning=False`로 빈 생각 블록을 고정했습니다. 실행 명령은 `uv run --with langchain --with langchain-ollama python 파일명`입니다. 먼저 파일을 만들고 그대로 실행합니다.

```python
import asyncio, json
from typing import Any


async def main():
    from langchain_core.tools import tool
    from langchain_core.messages import ToolMessage
    from langchain_ollama import ChatOllama


    @tool
    def get_inventory(sku: str, warehouse_id: str) -> dict[str, Any]:
        """승인된 창고에서 특정 상품의 현재 가용 재고를 읽기 전용으로 조회합니다.
        sku에는 상품 SKU를, warehouse_id에는 SEOUL-01 또는 BUSAN-02만 넣습니다.
        수량을 변경하지 않습니다. INVALID_SKU이면 SKU를 확인해 달라고 물어야 합니다."""
        if warehouse_id not in ("SEOUL-01", "BUSAN-02"):
            return {"error": "WAREHOUSE_NOT_ALLOWED"}
        return {"sku": sku, "warehouse_id": warehouse_id,
                "available_quantity": 7, "checked_at": "2026-08-27T01:55:00+09:00"}


    llm = ChatOllama(model="qwen3.5:2b", temperature=0, reasoning=False)
    llm_with_tools = llm.bind_tools([get_inventory])


    r1 = await llm_with_tools.ainvoke("BUSAN-02 창고의 SKU A-17 재고를 알려 줘")
    print("[1] 모델의 요청:", [(tc["name"], tc["args"]) for tc in r1.tool_calls])


    result = get_inventory.invoke(r1.tool_calls[0]["args"])
    print("[2] 하네스의 실행 결과:", json.dumps(result, ensure_ascii=False))


    r2 = await llm_with_tools.ainvoke(
        [r1, ToolMessage(content=json.dumps(result, ensure_ascii=False),
                         tool_call_id=r1.tool_calls[0]["id"])])
    print("[3] 결과를 읽은 최종 답:", r2.content[:120])


asyncio.run(main())
```

`@tool`은 함수의 이름·인자·설명을 모델이 호출할 수 있는 도구 계약으로 만들고, `bind_tools([get_inventory])`는 그 계약을 모델에 연결합니다. `ToolMessage`는 실제 도구 반환값을 이전 도구 호출의 `id`와 함께 모델에 돌려주는 메시지이며, `json.dumps(..., ensure_ascii=False)`는 그 반환값의 한글을 그대로 JSON 문자열로 만듭니다. 이 예제는 비동기 `main()` 안에서 모델 요청과 도구 결과 재전달을 기다리므로 `await`와 `ainvoke()`를 쓰며, `bind_tools()` 자체는 동기 `invoke()`도 지원합니다.

실측 출력은 다음과 같습니다.

```text
[1] 모델의 요청: [('get_inventory', {'sku': 'A-17', 'warehouse_id': 'BUSAN-02'})]
[2] 하네스의 실행 결과: {"sku": "A-17", "warehouse_id": "BUSAN-02", "available_quantity": 7, "checked_at": "2026-08-27T01:55:00+09:00"}
[3] 결과를 읽은 최종 답: 승인된 창고 (BUSAN-02) 에서 상품 A-17 의 현재 가용 재고는 7개입니다.
```

[1]은 모델의 소관입니다. 모델은 `get_inventory`와 인자를 요청할 뿐, 함수를 실행하지 않습니다. [2]는 하네스의 소관입니다. 하네스가 요청 인자를 꺼내 실제 함수를 호출하고 반환값을 받습니다. [3]은 다시 모델의 소관입니다. 도구 결과를 메시지로 받은 모델이 그 값에 근거하여 사용자 답을 만듭니다. 따라서 [1]의 호출이 그럴듯해도 [2]가 없으면 재고를 읽었다고 말할 수 없고, [2]가 성공해도 [3]이 수량을 바꾸어 말하면 결과 사용은 실패입니다.

### 실행 원문을 트레이스로 남깁니다

**트레이스(trace), 요청부터 최종 답까지의 실행 기록을** 남기면 나중의 수정이 어느 단계에 영향을 주었는지 되짚을 수 있습니다. `tool-lab` 폴더를 만든 뒤 `trace-01.txt`에 요청 문장, [1]의 도구 이름과 인자, [2]의 실행 결과, [3]의 최종 답을 순서대로 저장합니다. 모델명, 온도, 실행 명령도 파일 맨 위에 함께 적습니다. 통과나 실패라는 판정만 남기지 말고, 그 판정을 뒷받침하는 원문을 보존합니다. 이전 기록을 덮어쓰면 호출 변화와 실행 변화의 근거가 사라집니다.

### 경계 사례는 한 곳만 고쳐 다시 실행합니다

요청을 "부산 3번 창고의 A-17 재고를 알려 줘"로 바꾸어 실행합니다. 존재하지 않는 창고 코드로 함수가 호출되면 하네스는 `{"error": "WAREHOUSE_NOT_ALLOWED"}`를 돌려줍니다. 그 뒤 모델이 허용된 창고 코드를 다시 물어야 하는지 [3]에서 관찰합니다. 수량을 추정하거나 오류를 무시했다면 그 문장도 그대로 기록합니다.

문제가 있으면 설명 문장, 스키마 enum, 실패 뒤 행동 가운데 하나만 고칩니다. 예를 들어 실패 뒤 행동만 고쳐 "허용된 창고 코드를 물어본다"를 명시한 뒤 같은 요청으로 재실행하고, 요청·인자·실행 결과·최종 답을 `tool-lab/trace-02.txt`에 저장합니다. 두 트레이스를 나란히 놓고 호출 자체가 달라졌는지, 오류 반환은 같은지, 최종 답만 달라졌는지 비교합니다. 설명과 enum을 함께 바꾸거나 실패 처리까지 동시에 고치면 어느 변화가 효과를 냈는지 알 수 없습니다. 한 번에 한 곳만 바꾼 기록이 쌓여야 새 실패와 재등장을 가려냅니다.

### 자기 도구 대신 남이 만든 도구를 붙여 봅니다

지금까지는 자기 함수를 도구로 만들었습니다. 실무의 도구 연결은 남이 만든 MCP 서버를 내 에이전트에 등록하는 일이며, Claude Code와 opencode 모두 같은 설정 파일로 이를 다룹니다. 에이전트 입장에서는 빌트인 도구이든 MCP로 붙은 도구이든 모델에게 주입되는 도구 정의라는 점이 같습니다.

실습은 문서 검색 MCP 서버 하나를 두 도구에 각각 등록하고 같은 질문을 던지는 것입니다. 먼저 무엇을 붙일지부터 정합니다.

**Qdrant**는 문서 조각을 벡터로 보관하고 의미가 비슷한 조각을 찾아 돌려주는 오픈소스 검색 엔진입니다. 이번 실습에서는 Qdrant 본체에 더해 **mcp-server-qdrant**, 즉 Qdrant를 MCP 도구로 노출해 주는 연결 서버를 씁니다. Qdrant는 저장소이고, mcp-server-qdrant는 그 저장소를 에이전트의 도구 목록에 올려 주는 창구입니다. 창구가 도구 두 개를 내놓습니다. `qdrant-store`는 문장을 받아 벡터로 바꿔 저장하고, `qdrant-find`는 질문과 비슷한 문장을 찾아 돌려줍니다. 문장을 벡터로 바꾸는 임베딩 모델은 fastembed가 기본 제공하는 것을 로컬에서 쓰므로, 이 실습에는 회원 가입도 API 키도 필요 없습니다.

설치는 별도 절차가 없습니다. 연결 서버가 `uvx mcp-server-qdrant` 한 줄로 알아서 내려받아 실행하므로, 에이전트 설정에 이 실행 줄을 적는 것만으로 설치와 실행이 함께 끝납니다. `uvx`는 파이썬 실행 도구 uv의 명령으로, 패키지 이름을 받아 임시 환경을 만들어 실행해 주는 역할입니다. 저장 위치를 정하는 세 가지 환경 변수만 미리 이해하면 됩니다. `QDRANT_URL`은 저장소 주소인데 `:memory:`로 두면 서버 프로세스 메모리에만 저장되어 세션이 닫히면 사라집니다. `QDRANT_COLLECTION`은 문서 조각을 모아 두는 묶음의 이름입니다. `EMBEDDER_PROVIDER`와 `FASTEMBED_MODEL`은 문장을 벡터로 바꿀 모델의 지정입니다.

먼저 Claude Code는 프로젝트 루트의 `.mcp.json`에 서버를 등록합니다.

```json
{
  "mcpServers": {
    "qdrant": {
      "command": "uvx",
      "args": ["mcp-server-qdrant"],
      "env": {
        "QDRANT_URL": ":memory:",
        "QDRANT_COLLECTION": "policies",
        "EMBEDDER_PROVIDER": "fastembed",
        "FASTEMBED_MODEL": "BAAI/bge-small-en-v1.5"
      }
    }
  }
}
```

`mcpServers` 키 아래 각 항목이 MCP 서버 하나이고, `command`와 `args`는 서버를 띄우는 실행 줄, `env`는 그 서버가 읽을 환경 변수입니다. 파일을 저장한 뒤 `claude`를 실행하면 서버가 뜨며, `/mcp` 명령으로 등록된 서버와 도구 목록을 확인합니다. 등록된 도구의 설명 문장은 세션 시작 시 모델의 컨텍스트 창에 자동으로 주입됩니다. 이어서 같은 세션에서 문서를 저장하고 검색하도록 요청하면, 모델은 `qdrant-store`와 `qdrant-find` 중에서 스스로 도구를 골라 호출 요청을 씁니다.

opencode는 같은 서버를 자신의 설정 파일에 등록합니다. 프로젝트 루트에 `opencode.json`을 만들고 아래처럼 씁니다.

```json
{
  "mcp": {
    "qdrant": {
      "type": "local",
      "command": ["uvx", "mcp-server-qdrant"],
      "environment": {
        "QDRANT_URL": ":memory:",
        "QDRANT_COLLECTION": "policies",
        "EMBEDDER_PROVIDER": "fastembed",
        "FASTEMBED_MODEL": "BAAI/bge-small-en-v1.5"
      }
    }
  }
}
```

키 이름은 다르지만 내용은 같습니다. 누가 도구를 실행하는지(`command`), 무엇을 검색 대상으로 삼는지(`environment`)를 에이전트의 설정이 정의합니다. 설정을 저장한 뒤 opencode를 실행해 같은 질문을 던지면, 앞의 실습과 똑같은 Qdrant 도구가 Claude와 opencode 두 에이전트에서 같은 방식으로 주입되고 호출되는 것을 관찰할 수 있습니다.

아래 도식에서 밑줄 친 키 이름의 차이와 두 설정이 같은 서버로 향하는 경로를 함께 확인합니다.

![[Pasted image 20260827140050.png]]

실물 서버 랙의 케이블처럼 설정 파일의 표기는 달라도 양쪽 연결은 하나의 서버 실행 지점으로 모입니다.

![여러 케이블이 연결된 서버 랙](servers-in-a-rack.jpg)<span class="img-caption">출처: [Wikimedia Commons — Servers in a Rack.jpg](https://commons.wikimedia.org/wiki/File:Servers_in_a_Rack.jpg) · 라이선스: CC BY-SA 3.0 · 저작자: Abigor</span>

관찰 기록은 세 가지를 남깁니다. 첫째, 도구 정의가 두 에이전트에서 동일하게 주입되는지 — 각 세션에서 모델이 도구 설명을 읽고 골랐는지 호출 로그로 확인합니다. 둘째, 같은 질문에 두 에이전트가 같은 도구를 골랐는지, 골랐다면 인자를 같게 채웠는지입니다. 이 실습을 실측한 환경에서는 qwen이 아닌 각 에이전트의 기본 모델이 `qdrant-store`로 문장 세 개를 저장하고 `qdrant-find`로 검색을 요청했습니다. 셋째, 인메모리 저장은 세션이 닫히면 사라지므로 관찰 기록은 세션 안에서 `trace-mcp-01.txt`로 남겨야 합니다. 이 기록을 과제의 근거 문서 후보로 가져가고, 정책 문서를 찾아 넣고 질문을 바꾸는 일은 과제 설계의 범위입니다.

---

## 전체 커리큘럼 구성표

|  강좌 번호   | 강좌명                                     |
| :------: | :--------------------------------------- |
|  **1강**  | [[PE04-01 실패의 세 순간과 도구 호출의 작동 모델]]           |
|  **2강**  | [[PE04-02 도구 설명은 모델이 읽는 지시다]]           |
|  **3강**  | [[PE04-03 실습 1 - 도구 정의서와 실패 사례 리디자인]]           |
|  **4강**  | [[PE04-04 최소 하네스 - 여섯 개의 설계 지점]]           |
|  **5강**  | **5강. 실습 2 - 도구를 만들어 실행하고, RAG를 MCP로 붙이기 (현재)**           |
|  **6강**  | [[PE04-06 권한과 신뢰 경계 - 계약의 안전 층]]           |
|  **7강**  | [[PE04-07 과제 - 서비스 가이드 챗봇 설계]]           |
|  **8강**  | [[PE04-08 도구를 쓰는 평가 - 전후 판정과 회귀]]           |
|  **9강**  | [[PE04-09 안티패턴 회수와 관통 질문의 답]]           |
