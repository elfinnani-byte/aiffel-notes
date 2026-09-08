# [AI Agent 파헤치기] - 3. 업무 기록 MCP 만들기

---

## 업무 기록 MCP 만들기

### 기록을 가져오는 작은 기능

앞에서 살펴본 연결 구조를 이제 우리 팀의 파일에 적용해 보겠습니다. 가상의 제품팀에는 두 주의 업무 기록이 있습니다. 팀원이 매주 기록을 복사해 AI에게 전달할 수도 있지만, 보고서를 요청할 때마다 파일을 골라 붙여 넣어야 합니다. 지난주 자료를 잘못 전달하면 글이 아무리 매끄러워도 이번 주 보고서가 되지 않습니다.

이번에 만들 도구는 주 이름을 받으면 해당 주의 기록을 읽어 줍니다. 학생이 AI와 보고서를 작성하기에 앞서, 필요한 원자료를 가져오는 통로부터 준비하는 것입니다. 업무가 완료됐는지, 팀장에게 무엇을 강조할지는 뒤에서 다룹니다. 같은 원자료를 보고서에도 쓰고 일정 확인에도 쓸 수 있도록 조회 기능의 역할을 작게 잡겠습니다.

```text
입력: week-one
  ↓
첫째 주 자료 선택 → 파일 읽기 → 기본 모양 확인
  ↓
반환: 주차·팀·기준일·업무 기록 전체
```

이 흐름에서 직접 만드는 부분은 가운데의 파일 조회입니다. MCP 메시지를 받아 함수를 실행하고 결과를 보내는 통신 작업은 SDK가 맡습니다. **SDK(Software Development Kit)** 는 특정 기능을 개발할 때 사용하는 도구와 코드의 묶음입니다. 여기서는 MCP를 처음부터 구현하는 대신, 공식 Python SDK에 우리의 조회 함수를 등록합니다.

### 먼저 읽어 볼 원자료

[실습 자료 묶음](https://github.com/SunCreation/mcp-skill-practice/releases/download/v0.1.0/mcp-skill-practice.zip)을 내려받아 압축을 풉니다. Claude Code나 OpenCode에서 압축을 푼 `mcp-skill-practice` 폴더를 작업 폴더로 엽니다. 여러 실습 자료가 함께 있지만, 지금 살펴볼 파일과 환경 준비 뒤 생길 폴더는 다음과 같습니다.

```text
mcp-skill-practice/
  pyproject.toml
  uv.lock
  .python-version
  requirements.txt          # 이전 설치 방식의 호환 참고 자료
  .venv/                    # 환경 준비 때 자동 생성
  weekly_records_server.py
  data/
    week-one.json
    week-two.json
```

`data/week-one.json`은 현재 실습 폴더 안의 `data` 폴더에 있는 파일을 뜻합니다. 코드 파일과 `data`를 서로 다른 곳으로 옮기지 말고 위 관계를 유지해 주세요. 곧 작성할 코드는 자신의 옆에 있는 `data`를 찾습니다.

자료를 열어 보면 메모 문장 대신 이름과 값이 짝을 이루고 있습니다. 이런 표현을 **JSON**이라고 합니다. 다음은 첫째 주 파일에서 팀 정보와 `TASK-102`만 골라 보여 준 발췌입니다. 실제 파일의 업무는 네 개이며, 실습 파일을 아래 한 항목으로 덮어쓰지 않습니다.

```json
{
  "week": "week-one",
  "team": "가상 제품팀",
  "period_start": "2026-08-31",
  "report_date": "2026-09-04",
  "records": [
    {
      "id": "TASK-102",
      "title": "로그인 안내 문구 개선",
      "owner": "지우",
      "status": "개발 완료",
      "due_date": "2026-09-07",
      "deployed_at": null,
      "verification": "미실시",
      "blocker": null,
      "note": "코드 작성은 끝났으며 다음 배포를 기다린다."
    }
  ]
}
```

중괄호 `{}` 안에는 `"week": "week-one"`처럼 이름과 값이 들어 있습니다. 이름을 **키**, 그에 연결된 내용을 **값**이라고 부릅니다. `records`의 값은 대괄호 `[]`로 묶인 목록입니다. 한 주에는 업무가 여러 개이므로 업무 한 개씩을 객체로 만들고 목록에 넣었습니다. 보고서 기준일은 `report_date`, 특정 업무를 다시 찾을 때 쓸 식별자는 `id`입니다. 날짜는 가상 자료의 기준일이므로 컴퓨터의 오늘 날짜로 바꾸지 않습니다.

`null`은 해당 자리에 값이 없다는 표현입니다. `deployed_at`이 `null`이면 이 자료에 배포일이 들어 있지 않다는 뜻입니다. JSON을 Python으로 읽으면 null은 `None`이 됩니다. 두 표현은 서로 다른 자료 표현 방식에서 같은 '값 없음'을 나타냅니다.

여기에는 흥미로운 기록이 있습니다. `TASK-102`의 상태는 '개발 완료'인데 배포일은 없고 확인도 '미실시'입니다. 보고서에서 이것을 어떻게 표현할지는 팀의 기준을 알아야 정할 수 있겠지요. 지금 도구가 할 일은 이 차이를 지우지 않고 그대로 가져오는 것입니다. 도구가 임의로 '완료 한 건'으로 요약하면 다음 단계에서 필요한 근거가 사라집니다.

원본에서 `TASK-102`를 찾아 상태와 `note`를 직접 읽어 보세요. 다음 절에서 실제 도구를 호출했을 때 다시 확인할 기준이 됩니다.

### 실습 폴더의 Python 환경

코드를 실행하려면 Python과 MCP 패키지가 필요합니다. **패키지**는 다른 개발자가 만들어 배포한 기능 묶음입니다. Python에 기본으로 들어 있는 파일 읽기 기능만으로는 MCP 서버를 만들 수 없으므로 MCP 패키지를 추가합니다. 이번에는 Python 실행 환경과 패키지를 함께 관리하는 **uv**를 사용합니다. 첨부 폴더는 이미 uv 프로젝트로 준비되어 있으므로 새 프로젝트를 만드는 `uv init`이나 패키지를 추가하는 `uv add`를 다시 실행할 필요는 없습니다.

실습 폴더의 세 파일은 서로 다른 질문에 답합니다. `pyproject.toml`은 '우리 코드에 무엇이 필요한가'를 선언합니다. 이 파일의 `dependencies`에 들어 있는 `mcp[cli]==2.1.1`은 우리가 직접 사용하는 **직접 의존성**입니다. `cli`는 터미널에서 `mcp` 명령을 사용할 추가 기능을 포함한다는 뜻이고, `==2.1.1`은 MCP 버전을 고정합니다. 같은 파일의 `requires-python = ">=3.10"`은 프로젝트가 요구하는 Python 버전의 하한입니다.

`uv.lock`은 '그 조건을 만족하는 패키지들을 어떤 버전으로 조합할 것인가'를 기록합니다. MCP 자체도 다른 패키지를 필요로 합니다. 우리가 직접 이름을 적지 않았지만 MCP를 실행하려고 함께 설치되는 패키지가 **간접 의존성**입니다. 잠금 파일에는 직접·간접 의존성의 확정 버전과 설치에 필요한 정보가 들어 있습니다. 운영체제나 Python 버전에 따라 필요한 항목이 다를 수 있으므로, uv는 그중 현재 환경에 맞는 항목을 선택합니다. 학생이 `uv.lock`을 손으로 수정하지 않고 첨부된 파일을 사용하는 이유입니다. [uv 프로젝트 파일 설명](https://docs.astral.sh/uv/concepts/projects/layout/)

`.python-version`에는 `3.13`이 들어 있습니다. 프로젝트가 허용하는 범위가 '3.10 이상'이라면, 이 파일은 이번 실습에서 선택할 Python을 '3.13 계열'로 맞춥니다. 패키지 버전을 기록한 `uv.lock`과 Python 버전을 선택하는 파일의 역할을 구분해 주세요. `3.13`은 `3.13.x`의 마지막 숫자까지 고정한 표기는 아닙니다. uv는 조건에 맞는 Python을 찾아 사용하고, 필요한 Python이 없으면 기본 설정에서 자동으로 내려받을 수 있습니다. [uv의 Python 관리 안내](https://docs.astral.sh/uv/guides/install-python/)

**가상환경**은 프로젝트마다 설치할 패키지를 분리하는 공간입니다. 별도 컴퓨터를 만드는 것은 아닙니다. 예를 들어 다른 프로젝트가 MCP SDK 1.x로 작성되어 있어도, 이번 프로젝트의 2.1.1을 별도 환경에 두면 서로 다른 코드를 각자 맞는 환경에서 실행할 수 있습니다. uv는 기본 설정에서 `pyproject.toml` 옆에 `.venv`를 자동으로 만듭니다. 자료를 넣은 `data`와 실행에 필요한 패키지를 둔 `.venv`는 서로 다른 역할의 폴더입니다. [uv 프로젝트 환경 설명](https://docs.astral.sh/uv/concepts/projects/layout/#the-project-environment)

첨부 `pyproject.toml`의 `[tool.uv]` 아래에는 `package = false`도 있습니다. 이는 이번 실습 코드 자체를 설치 가능한 패키지로 빌드하지 않고, 필요한 외부 패키지와 실행 환경을 관리하겠다는 설정입니다. 서버는 뒤에서 `weekly_records_server.py` 파일로 실행합니다. 함께 남겨 둔 `requirements.txt`는 이전 설치 방식과의 호환을 위한 참고 자료이며, 이 교안의 환경 준비 기준은 `pyproject.toml`과 `uv.lock`입니다. [uv의 프로젝트 패키징 설명](https://docs.astral.sh/uv/concepts/projects/config/#project-packaging)

터미널은 프로그램에 명령을 글로 전달하는 창입니다. 학생이 모든 설치 명령을 외울 필요는 없습니다. 실습 폴더를 연 Claude Code 또는 OpenCode에 먼저 짧게 요청해 보세요.

> 이 폴더를 uv로 실행할 수 있게 준비해 줘. Python과 MCP 버전도 확인해 줘.

uv가 설치되어 있지 않다면 "내 운영체제에 uv를 설치하고 실습 환경을 준비해 줘"라고 요청하면 됩니다. 설치 방법은 uv 공식 설치 안내를 기준으로 합니다.

> [!info]+ **수동으로 uv 설치와 Python 환경 준비하기**
> 먼저 터미널에서 `uv --version`을 실행합니다. uv 버전이 나오면 바로 실습 폴더로 이동합니다. 명령을 찾지 못하면 공식 설치 안내에서 운영체제에 맞는 방법을 선택합니다. 공식 독립 설치 프로그램을 사용하는 명령은 다음과 같습니다.
> 
> macOS·Linux:
>```
>curl -LsSf https://astral.sh/uv/install.sh | sh
>```
>
>>Windows PowerShell:
>```
>powershell -ExecutionPolicy ByPass -c "irm https://astral.sh/uv/install.ps1 | iex"
>```
>
>설치 후 터미널을 새로 열고 `uv --version`이 표시되는지 확인합니다. 이제 현재 위치를 압축을 푼 실습 폴더로 옮깁니다. 경로는 실제 위치로 바꾸며, 폴더 이름에 공백이 있을 수 있으므로 따옴표로 감쌉니다.
>
>macOS·Linux:
>```
>cd "/압축을/푼/mcp-skill-practice"
>```
>>
>Windows PowerShell:
>```
>cd "C:\압축을\푼\mcp-skill-practice"
>```
>
>이후 환경 준비와 확인 명령은 두 운영체제에서 같습니다.
>```
>uv sync --locked
>uv run --locked python --version
>uv run --locked python -c "import sys; print(sys.executable)"
>uv run --locked python -c "from importlib.metadata import version; print(version('mcp'))"
>```
>`uv sync`는 프로젝트의 잠금 파일을 기준으로 필요한 패키지를 가상환경에 맞추는 명령입니다. `.venv`가 없으면 만들고, 현재 실습에 필요한 의존성을 설치합니다. `--locked`는 프로젝트 선언과 잠금 파일이 맞는지 확인하되 잠금 파일은 변경하지 않도록 합니다. 둘이 맞지 않거나 잠금 파일이 없으면 오류로 멈춥니다. 이때 옵션을 빼고 계속하기보다 첨부된 `pyproject.toml`과 `uv.lock`을 함께 사용하고 있는지 확인합니다.
>
>`uv run`은 뒤에 적은 명령을 이 프로젝트의 환경에서 실행합니다. 실행 전에 필요한 환경도 확인하므로, 별도로 가상환경을 활성화하는 명령을 외우지 않아도 됩니다. 여기에도 `--locked`를 붙여 실행 과정에서 잠금 파일이 바뀌지 않게 합니다. `--locked`는 설치를 금지하거나 인터넷 연결을 없애는 옵션이 아닙니다. 필요한 패키지가 없으면 잠금 파일에 맞춰 설치할 수 있습니다. [uv의 잠금 파일과 환경 동기화 설명](https://docs.astral.sh/uv/concepts/projects/sync/)
>
>마지막 세 명령은 각각 Python 버전, 실제 Python 실행 파일의 위치, 설치된 MCP 버전을 출력합니다. `-c`는 뒤의 짧은 Python 코드를 실행하라는 뜻이고, `importlib.metadata`는 설치된 패키지 정보를 읽는 Python 기본 기능입니다. 결과에서 Python은 `3.13.x`, 실행 경로는 실습 폴더의 `.venv` 아래, MCP 버전은 `2.1.1`인지 확인합니다.

설치가 끝났다는 답변에서 **실행한 Python의 위치**를 눈여겨보세요. 보통 macOS·Linux에서는 실습 폴더의 `.venv/bin/python`, Windows에서는 `.venv\Scripts\python.exe`를 가리킵니다. 앞에서 설명한 가상환경과 실제 실행 파일의 위치가 연결되는지 확인하는 과정입니다.

이것은 다음 절의 연결 문제와도 이어집니다. `.venv`에 패키지를 설치해 놓고 호스트가 컴퓨터의 다른 Python을 실행하면 '설치했는데도 mcp를 찾을 수 없다'는 오류가 날 수 있습니다. 그때는 재설치부터 반복하기보다 **설치한 프로젝트와 호스트가 실행하는 프로젝트가 같은지** 확인합니다. 다음 절에서는 호스트도 실습 폴더를 지정해 `uv run --locked`로 실행하게 하여 이 환경을 사용합니다. AI에게 준비를 맡겨도 이 관계를 알고 있으면 무엇을 확인해 달라고 요청할지 판단할 수 있습니다.

### AI와 구현하고 코드 따라가기

첨부 자료에는 완성 예제 `weekly_records_server.py`가 있습니다. 먼저 내용을 열어 두고 다음과 같이 요청해 보세요. 완성 예제를 읽으며 구현 원리를 익혀도 좋고, 예제를 별도로 보관하고 직접 생성한 코드와 비교해도 좋습니다.

> 주를 고르면 이 폴더의 업무 기록을 읽어 주는 MCP를 만들어 줘. 보고서는 아직 만들지 말자.

> [!info]+ **수동으로 서버 파일 작성하기**
> 
> 실습 폴더의 `weekly_records_server.py`에 다음 코드를 저장합니다. 첨부된 파일과 같은 완성 예제입니다.
>
>```python
>"""고정된 주간 업무 기록을 읽는 MCP 서버. 보고서 판정은 하지 않습니다."""
>
>import json
>from pathlib import Path
>from typing import Any
>
>from mcp.server import MCPServer
>
>mcp = MCPServer("weekly-records")
>DATA_DIR = Path(__file__).with_name("data")
>WEEK_FILES = {
 >   "week-one": "week-one.json",
 >   "week-two": "week-two.json",
>}
>
>
>def load_weekly_records(week: str) -> dict[str, Any]:
 >   """허용한 주차의 예시 파일만 읽습니다. 파일 경로를 입력받지 않습니다."""
 >   selected_week = week.strip().lower()
 >   if selected_week not in WEEK_FILES:
 >       raise ValueError("week는 week-one 또는 week-two를 사용하세요.")
>
 >   with (DATA_DIR / WEEK_FILES[selected_week]).open(encoding="utf-8") as file:
 >       data = json.load(file)
>
 >   if not isinstance(data, dict) or not isinstance(data.get("records"), list):
 >       raise ValueError("주간 데이터는 records 목록이 있는 객체여야 합니다.")
 >   if data.get("week") != selected_week:
 >       raise ValueError("요청한 주차와 데이터의 week가 다릅니다.")
 >   return data
>
>
>@mcp.tool()
>def get_weekly_records(week: str) -> dict[str, Any]:
 >   """week-one 또는 week-two의 가상 팀 업무 기록을 읽습니다.
>
 >   반환값은 주차, 팀 이름, 기간 시작일, 보고 기준일, 업무 기록 목록입니다.
 >   파일을 수정하지 않으며 완료·진행·위험 판정이나 보고서 작성은 하지 않습니다.
 >   """
 >   return load_weekly_records(week)

AI가 만든 코드가 위 코드와 글자까지 같을 필요는 없습니다. 대신 주차가 어떤 파일을 가리키는지, 없는 주차를 받으면 어떻게 하는지, 원본 기록이 그대로 돌아오는지를 살펴봅니다. 이 판단을 할 수 있도록 완성 예제에서 `week-one` 한 건이 지나가는 길을 따라가 보겠습니다.

### 주차를 파일로 바꾸는 표

먼저 사용할 기능을 가져오고 자료의 위치를 정합니다.

```python
import json
from pathlib import Path
from typing import Any

DATA_DIR = Path(__file__).with_name("data")
WEEK_FILES = {
    "week-one": "week-one.json",
    "week-two": "week-two.json",
}
```

`import`는 다른 곳에 정의된 기능을 이 파일에서 사용할 수 있게 가져오는 문법입니다. `json`은 JSON을 Python 자료로 읽는 데, `Path`는 파일 위치를 다루는 데 사용합니다. `Any`는 조금 뒤에 볼 반환값의 자료형 설명에 등장합니다. 세 기능은 Python에 포함되어 있습니다. MCP 관련 기능은 별도로 설치한 패키지에서 가져옵니다.

`WEEK_FILES`는 주차와 파일 이름을 연결한 딕셔너리입니다. JSON 객체처럼 키로 값을 찾는 Python 자료 구조입니다. `WEEK_FILES["week-one"]`을 읽으면 `"week-one.json"`을 얻습니다. 도구를 쓰는 쪽은 실제 폴더 구조를 몰라도 `week-one`이라고 요청하면 됩니다.

두 주차를 표에 적어 두었으므로 아무 파일 경로나 받는 조회 기능이 되지 않습니다. 예를 들어 `week-three`를 요청했을 때 비슷한 이름의 파일을 추측해서 열지 않고, 허용한 주차인지 먼저 검사할 수 있습니다. 이후 실제 서비스를 만든다면 주차 목록을 어디서 얻을지 다시 설계하겠지만, 지금은 입력과 결과의 관계를 확실히 관찰할 수 있는 두 파일로 시작합니다.

`Path(__file__)`에서 `__file__`은 이 **Python 파일의 경로**입니다. `.with_name("data")`는 그 경로의 마지막 이름인 `weekly_records_server.py`를 `data`로 바꿉니다. 예를 들어 코드가 `/Users/student/mcp-skill-practice/weekly_records_server.py`에 있다면 자료 위치는 `/Users/student/mcp-skill-practice/data`가 됩니다.

이 기준을 쓰는 이유는 호스트가 터미널의 현재 폴더와 다른 곳에서 서버를 실행할 수 있기 때문입니다. 단순히 `data/week-one.json`을 열면 실행 당시의 작업 폴더를 기준으로 찾을 수 있습니다. 코드 파일의 위치를 기준으로 삼으면 호스트의 시작 위치가 달라도 코드 옆의 자료를 찾도록 만들 수 있습니다. `Path`와 경로 결합 방식은 [Python pathlib 문서](https://docs.python.org/3/library/pathlib.html)에서 확인할 수 있습니다.

### 입력을 확인하는 함수

```python
def load_weekly_records(week: str) -> dict[str, Any]:
    selected_week = week.strip().lower()
    if selected_week not in WEEK_FILES:
        raise ValueError("week는 week-one 또는 week-two를 사용하세요.")
```

함수는 이름을 붙여 다시 실행할 수 있는 작업 묶음입니다. `def`로 함수를 정의하고, 괄호 안의 `week`로 입력을 받습니다. `load_weekly_records("week-one")`이라고 호출하면 이 함수 안에서 `week`는 `"week-one"`입니다. 콜론 뒤에서 들여쓴 줄들이 함수의 작업이며, 더 깊이 들여쓴 `raise`는 `if` 조건이 맞을 때 실행됩니다.

`week: str`은 입력이 문자열이라는 **타입 힌트**입니다. `-> dict[str, Any]`는 문자열 키를 가진 딕셔너리를 반환하며 그 값의 종류는 다양할 수 있다는 설명입니다. 반환 자료에는 `team`이라는 문자열도 있고 `records`라는 목록도 있으므로 값의 타입을 하나로 제한하지 않은 것입니다. 타입 힌트는 코드의 입력과 출력을 읽는 사람과 도구에 알려 줍니다. **일반 Python 함수에 힌트를 적었다고 모든 호출의 타입을 Python이 자동 검사하는 것은 아닙니다.** MCP를 통해 들어오는 입력을 검사하는 단계는 뒤에서 별도로 연결하겠습니다.

`strip()`은 문자열 양끝의 공백을 제거하고, `lower()`는 영문을 소문자로 바꿉니다. 따라서 `" WEEK-ONE "`은 `"week-one"`이 됩니다. 이런 처리를 정규화라고 합니다. 뜻은 같은 입력을 비교하기 쉬운 한 형태로 맞추는 것입니다. `week-three`를 `week-one`으로 바꾸는 것처럼 내용을 추측하지는 않습니다.

그다음 `if selected_week not in WEEK_FILES`로 표에 없는 주차인지 확인합니다. 없는 주차라면 `raise ValueError(...)`로 작업을 중단하고 값이 적절하지 않다는 오류를 알립니다. 기록이 없는 상황을 정상적인 빈 보고서처럼 넘기지 않으므로, 호출한 쪽은 입력을 확인해 다시 요청할 수 있습니다.

### 파일을 읽고 원본을 돌려주기

주차가 맞으면 다음 부분으로 넘어갑니다.

```python
with (DATA_DIR / WEEK_FILES[selected_week]).open(encoding="utf-8") as file:
    data = json.load(file)
```

`WEEK_FILES[selected_week]`는 방금 선택한 주차의 파일 이름입니다. `DATA_DIR / 파일이름`에서 `/`는 숫자를 나누는 연산이 아니라, `Path` 객체에 파일 이름을 이어 붙이는 연산입니다. `week-one`을 받았다면 최종적으로 `data/week-one.json`의 위치를 만들게 됩니다.

`open()`은 그 파일을 엽니다. `encoding="utf-8"`은 한글을 포함한 문자를 UTF-8 방식으로 읽겠다는 뜻입니다. `with ... as file:`은 열린 파일을 `file`이라는 이름으로 쓰는 범위를 만들고, 해당 블록을 벗어날 때 파일을 닫도록 처리합니다. 파일을 열어 둔 채 잊어버리지 않게 하는 문법입니다.

`json.load(file)`은 파일에 적힌 JSON 텍스트를 Python에서 다룰 수 있는 자료로 바꿉니다. 바깥 객체는 딕셔너리, `records`의 배열은 목록, `null`은 `None`이 됩니다. 이 시점의 `data`는 화면에 출력한 글이 아니라 코드가 키로 값을 찾을 수 있는 자료입니다.

읽기가 끝났다고 바로 반환하지는 않습니다. 파일이 요청과 맞는 기본 구조인지 검사합니다.

```python
if not isinstance(data, dict) or not isinstance(data.get("records"), list):
    raise ValueError("주간 데이터는 records 목록이 있는 객체여야 합니다.")
if data.get("week") != selected_week:
    raise ValueError("요청한 주차와 데이터의 week가 다릅니다.")
return data
```

`isinstance(data, dict)`는 `data`가 딕셔너리인지 확인합니다. `data.get("records")`는 `records`에 연결된 값을 읽으며, 키가 없으면 기본적으로 `None`을 돌려줍니다. 그 값이 목록인지도 확인하므로, 파일에 업무 목록이 빠졌거나 전혀 다른 모양으로 저장됐을 때 정상 자료로 넘기지 않습니다. `or`의 앞 조건이 참이면 뒤 조건은 평가하지 않으므로, 딕셔너리가 아닌 자료에 `.get()`을 호출하지도 않습니다.

다음 검사는 파일의 `week`와 요청한 주차가 같은지 확인합니다. 파일 이름은 첫째 주인데 내부가 둘째 주 자료라면 보고서가 잘못될 수 있기 때문입니다. 지금 검사는 바깥 자료 구조와 주차에 집중합니다. 모든 업무의 날짜 형식이나 기록의 사실 여부까지 확인하는 코드는 아닙니다. 나중에 외부 입력을 받는 서비스로 바꾼다면 어떤 검사가 더 필요한지 따로 생각해 볼 수 있습니다.

마지막 `return data`가 호출한 쪽에 결과를 돌려줍니다. `print(data)`로 화면에 글을 쓰는 것과 다릅니다. MCP SDK가 반환 자료를 받아 호스트에 전할 수 있으려면 함수의 **반환값**이 필요합니다. 원자료의 네 기록은 여기서 요약되거나 분류되지 않습니다.

### Python 함수를 도구로 공개하기

지금까지 본 `load_weekly_records`는 보통의 Python 함수입니다. 같은 파일에 있다고 모든 함수가 AI에게 공개되지는 않습니다. 도구로 제공할 함수를 SDK에 등록하는 부분을 보겠습니다.

```python
from mcp.server import MCPServer

mcp = MCPServer("weekly-records")

@mcp.tool()
def get_weekly_records(week: str) -> dict[str, Any]:
    """week-one 또는 week-two의 가상 팀 업무 기록을 읽습니다.

    반환값은 주차, 팀 이름, 기간 시작일, 보고 기준일, 업무 기록 목록입니다.
    파일을 수정하지 않으며 완료·진행·위험 판정이나 보고서 작성은 하지 않습니다.
    """
    return load_weekly_records(week)
```

`MCPServer("weekly-records")`는 도구들을 제공할 서버 객체를 만듭니다. 객체는 관련 상태와 기능을 함께 다룰 수 있는 코드 단위이며, 여기서는 만들어진 서버를 `mcp`라는 변수에 담습니다. `weekly-records`는 서버 이름이고, `get_weekly_records`는 그 서버가 제공하는 도구 이름입니다.

함수 위의 `@mcp.tool()`은 **데코레이터** 문법입니다. 바로 아래 함수를 서버의 도구로 등록합니다. `load_weekly_records`에는 이 표시가 없으므로 외부에 도구로 공개되지 않습니다. 공개 함수는 요청을 받아 실제 파일 읽기를 담당한 함수를 호출합니다. 이렇게 나누면 파일 조회 규칙을 MCP 통신과 별도로 읽고 시험하기 쉽습니다.

```text
MCP에 공개된 get_weekly_records(week)
                 ↓
파일 조회 함수 load_weekly_records(week)
                 ↓
data/week-one.json → Python 딕셔너리 → MCP 응답
```

함수 안의 삼중 따옴표 문자열은 **docstring**, 즉 함수 설명문입니다. SDK는 함수 이름과 설명, 타입 힌트를 이용해 호스트에 제공할 도구 정의를 만듭니다. 앞 절에서 본 도구 안내가 코드에서는 이 부분과 연결됩니다. MCP Python SDK 2.1.1의 서버 작성 방식은 [공식 SDK 예제](https://github.com/modelcontextprotocol/python-sdk/tree/v2.1.1)를 기준으로 합니다.

호스트가 받을 입력 규칙에서 핵심만 읽으면 다음과 같습니다. 이는 이해를 위해 부가 필드를 생략한 스키마 발췌입니다.

```json
{
  "type": "object",
  "properties": {
    "week": { "type": "string" }
  },
  "required": ["week"]
}
```

스키마는 자료가 어떤 모양이어야 하는지 정한 규칙입니다. 위 규칙은 입력 객체에 `week`가 꼭 있어야 하고 그 값은 문자열이어야 한다고 알려 줍니다. 학생이 이 JSON을 따로 작성하는 것이 아니라 SDK가 함수 정보를 바탕으로 만듭니다. 그래서 `{"week": 1}`처럼 숫자를 보내거나 `week`를 생략하면 MCP SDK의 입력 처리에서 오류가 납니다. 반면 `"week-three"`는 문자열이라는 조건에는 맞지만, 함수 안의 `WEEK_FILES` 검사에서 거절됩니다. **입력의 모양 검사와 우리가 정한 허용 값 검사는 서로 다른 역할**입니다.

도구 설명이 필요한 이유도 여기서 드러납니다. 모델에게 `week`라는 문자열만 알려 주면 어떤 값을 써야 할지 부족합니다. 설명문이 `week-one`과 `week-two`, 반환되는 정보, 조회 기능의 범위를 알려 줍니다. 나중에 AI가 도구를 엉뚱하게 사용한다면 실행 코드뿐 아니라 모델이 읽는 이 안내도 살펴볼 수 있습니다.

### 다음 호출에서 확인할 결과

이제 `week-one`이 들어와 원본 자료가 돌아오는 경로를 코드에서 따라왔습니다. AI가 작성한 버전에서도 이 흐름을 찾아보세요. 구현이 다르다면 차이가 결과에 어떤 영향을 주는지 설명해 달라고 요청할 수 있습니다. 예를 들어 조회 도구 안에 보고서 작성까지 들어 있다면, 지금 만들기로 한 기능의 범위를 다시 알려 주면 됩니다.

다음 절에서 호스트를 연결한 뒤에는 다음 입력으로 결과를 관찰하겠습니다.

| 입력                       | 확인할 결과                                             | 관련 코드         |
| :----------------------- | :------------------------------------------------- | :------------ |
| `week-one`               | 기준일 2026-09-04, TASK-101부터 TASK-104까지 네 기록이 원본과 일치 | 파일 읽기와 반환     |
| `WEEK-ONE`               | 첫째 주의 같은 자료 반환                                     | 공백 제거와 소문자 변환 |
| `week-three`             | 기록 대신 오류 결과                                        | 허용 주차 검사      |
| `week` 값에 숫자 전달 또는 입력 생략 | 입력 검증 오류                                           | SDK의 스키마 검사   |

실패한 요청의 문구는 호스트와 SDK의 오류 표시 방식에 따라 달라질 수 있습니다. 화면에 함수의 한국어 문장이 그대로 나오지 않더라도, 어떤 입력이 거절됐는지부터 확인하고 필요하면 서버 로그에서 원인을 찾습니다. 첨부 자료의 `weekly-records-verification.md`에는 SDK 클라이언트로 수행한 기존 검증 결과가 있습니다. 이제 학생의 호스트에서도 직접 호출해 같은 원자료를 받는지 확인할 차례입니다.

한 가지 실행 방식도 짚고 넘어가겠습니다. **첨부된 완성 예제** `weekly_records_server.py`에는 서버를 직접 시작하는 코드가 없으므로 `uv run --locked python weekly_records_server.py`로 실행하면 함수와 서버 객체를 정의한 뒤 끝납니다. 다음 절에서는 `mcp run`이 이 파일의 서버 객체를 찾아 실행하도록 호스트에 등록합니다. 별도 터미널에서 서버를 계속 켜 놓은 뒤 연결하는 방식이 아니라, 호스트가 필요한 서버 프로세스를 시작하는 방식으로 진행합니다.

AI가 생성한 다른 구현을 사용한다면 시작 방식도 확인해 주세요. 생성본에는 서버를 직접 시작하는 부분, 즉 **실행 진입점**이 들어 있을 수 있습니다. 그런 파일은 Python으로 직접 실행하는 방식이 맞을 수 있습니다. 다음 절의 수동 설정은 첨부 완성 예제 기준입니다. 생성본은 AI와 실제 파일 이름·공개 도구 이름·서버 시작 부분을 함께 읽고, 그 코드에 맞는 실행 명령을 호스트에 등록합니다. "이 서버는 어떻게 시작하는지 코드에서 찾아 설명해 줘" 정도로 짧게 요청해도 좋습니다.

또한 stdio 연결에서는 표준 출력이 MCP 메시지가 오가는 통로입니다. 디버깅을 위해 넣은 `print()`도 같은 통로에 글을 쓰므로 통신을 방해할 수 있습니다. 서버에서 진단 내용을 남길 때는 표준 오류로 보내는 로그를 사용합니다. 위 환경 준비에서 터미널에 Python 경로를 출력한 것은 서버 통신 밖에서 실행한 독립 명령이므로 목적과 위치가 다릅니다.

코드 파일은 준비됐지만 호스트에 아직 이 서버를 등록하지 않았습니다. 다음에는 **어떤 프로그램으로 이 파일을 실행할지** 알려 주고, 처음 골라 두었던 `TASK-102`의 기록이 실제 도구 응답으로 돌아오는지 살펴보겠습니다.

---

## 전체 커리큘럼 구성표

|  강좌 번호   | 강좌명                                     |
| :------: | :--------------------------------------- |
|  **1강**  | [[AA05-01 에이전트와 도구 연결]]           |
|  **2강**  | [[AA05-02 MCP 연결 구조와 메시지의 흐름]]           |
|  **3강**  | **3강. 업무 기록 MCP 만들기 (현재)**           |
|  **4강**  | [[AA05-04 호스트 연결과 첫 도구 호출]]           |
|  **5강**  | [[AA05-05 많은 도구와 문맥 부담]]           |
|  **6강**  | [[AA05-06 Claude Code의 Tool Search]]           |
|  **7강**  | [[AA05-07 OpenCode의 Code Mode]]           |
|  **8강**  | [[AA05-08 Skill의 역할과 업무 지식]]           |
|  **9강**  | [[AA05-09 팀 보고서 첫 작성]]           |
|  **10강**  | [[AA05-10 완성한 업무 방식을 Skill로 남기기]]           |
|  **11강**  | [[AA05-11 새 업무 기록에서 Skill 재사용하기]]           |
|  **12강**  | [[AA05-12 Colab MCP 연결과 실행]]           |
|  **13강**  | [[AA05-13 Google Workspace 자료를 읽는 인증과 연결]]           |
|  **14강**  | [[AA05-14 자기 과업에 맞게 확장하기]]           |
|  **15강**  | [[AA05-15 실행 경험과 Skill 개선 연구]]           |
