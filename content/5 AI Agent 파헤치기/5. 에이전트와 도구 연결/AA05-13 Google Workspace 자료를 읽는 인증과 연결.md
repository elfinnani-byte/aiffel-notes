# [AI Agent 파헤치기] - 13. Google Workspace 자료를 읽는 인증과 연결

---

## Google Workspace 자료를 읽는 인증과 연결

앞 절에서는 Colab에 계산을 남기고 다시 실행하는 과정을 살펴봤습니다. 이번에는 계산에 필요한 원자료가 팀의 공유 표에 있는 상황입니다. 금요일마다 동료가 업데이트한 Google Sheets를 내려받아 실습 폴더에 복사한다면, 복사한 뒤 바뀐 기록은 보고서에 반영되지 않겠지요. 필요한 순간에 표를 직접 읽을 수 있다면 이 전달 단계를 바꿀 수 있습니다.

브라우저에서는 이미 표가 잘 보입니다. 그런데 Python 프로그램에서 같은 표를 읽으려면 준비가 더 필요합니다. 브라우저에 로그인한 사실이 새로 실행한 프로그램으로 자동 전달되지는 않기 때문입니다. 프로그램은 어떤 앱이, 어느 사용자의 동의를 받아, 어떤 작업을 하려는지 Google에 알려야 합니다. 이 절에서는 그 준비를 직접 해 보고, 표의 첫 부분이 반환되는 순간까지 따라갑니다.

Google Docs·Sheets·Drive·Gmail·Calendar는 Google Workspace의 문서·표·파일·메일·일정 서비스입니다. 내 계정의 비공개 자료를 MCP나 Skill의 스크립트로 다루려면 Google이 허용한 접근 경로가 필요합니다. Skill 문서를 작성하는 것 자체에는 credential이 필요하지 않습니다. 그 Skill이 사용하는 MCP·CLI·스크립트가 Google API를 호출할 때 인증이 필요합니다.

이 실습은 프로젝트와 앱 준비 → 최초 로그인 → 프로그램의 API 읽기 → MCP 등록 → 에이전트의 실제 호출 순서입니다. Colab MCP와는 별개의 인증 경로입니다. 앞에서 Colab 노트북 연결을 승인한 것은 그 연결에 대한 승인입니다. 이제 만들 Google API 앱이 표를 읽는 데 필요한 동의까지 대신하지는 않습니다.

이 관계를 먼저 한 줄로 놓고 봅시다.

```text
로컬 프로그램 → Google 로그인·동의 화면 → 학생의 계정 선택·동의
             ← 인증 결과를 받아 토큰 저장 ←
로컬 프로그램 → 토큰을 첨부해 Sheets API 호출 → 표 데이터 반환
```

Google 계정의 비밀번호는 Google 로그인 화면에 입력합니다. 로컬 프로그램은 동의 결과를 바탕으로 API를 요청할 수 있는 토큰을 받습니다. 뒤에서 나오는 여러 설정 화면은 이 흐름의 서로 다른 부분을 준비하는 화면입니다.

화면 메뉴가 작게 보이면 각 캡처를 확대해 확인합니다. 실제 콘솔은 언어와 업데이트 시점에 따라 메뉴 위치가 조금 다를 수 있습니다.

### 인증 정보와 토큰의 차이

**Credential(인증 정보)**은 앱이나 사용자를 식별·인증하는 데 쓰는 정보입니다. OAuth는 비밀번호를 도구에 넘기는 대신, Google 화면에서 허용할 접근을 선택하는 절차입니다. 클라이언트 ID는 '어떤 앱인가'를 나타내고, **접근 토큰(access token)**은 동의한 범위에서 API를 호출할 때 사용합니다. **갱신 토큰(refresh token)**은 새 접근 토큰을 받는 데 쓰일 수 있는 값입니다. **Scope(범위)**는 '표 읽기', '일정 읽기'처럼 허용할 작업의 범위입니다.

| 필요한 일 | 인증 방식의 출발점 |
| :--- | :--- |
| 내가 로그인해 내 자료를 다루기 | 사용자 동의를 받는 OAuth 클라이언트 |
| 프로그램 전용 계정에 공유한 자료 다루기 | 서비스 계정; 공유한 파일 등의 접근 범위를 별도로 설정 |
| 지원되는 공개 데이터 조회 | API 키를 지원하는 API인지 먼저 확인 |

API 키 하나로 내 Gmail이나 비공개 문서 전체에 접근할 수 있는 것은 아닙니다. 기본 실습은 내 컴퓨터에서 실행하는 도구 + 내 계정의 OAuth 동의로 진행합니다. 연결 서비스가 이미 'Google로 로그인'을 제공한다면 자체 credential 발급이 불필요할 수도 있습니다. 먼저 사용할 MCP의 README나 Skill의 연결 안내에서 요구하는 방식을 확인합니다. [Google 인증 정보 선택 안내](https://developers.google.com/workspace/guides/create-credentials)

서비스 계정도 이름에 '계정'이 들어가지만 사람이 로그인하는 개인 계정과는 다릅니다. 예를 들어 매일 실행하는 서버에 전용 계정을 마련하고 그 계정에 연습용 표를 공유하는 구성이 가능합니다. 이 경우에는 '내가 로그인해서 볼 수 있는 표'와 '프로그램 전용 계정에 공유된 표'가 달라집니다. 여기서는 내 계정으로 로그인하는 사용자 OAuth 한 가지를 실제로 구현합니다.

### 개인 계정과 실습 도구

이번 캡처는 개인 Google 계정, 별도 실습 프로젝트, Google Sheets API, Desktop app, spreadsheets.readonly 조합으로 진행했습니다. 개인 Gmail 계정도 이 API를 사용할 수 있습니다. 서비스 이름에 Workspace가 들어간다고 유료 회사 계정이 필수인 것은 아닙니다.

Claude Code나 OpenCode에 처음에는 목적부터 짧게 알려 줍니다.

> 이 Google Sheets 도구를 연결하고 싶어. 안내 문서를 읽고 내가 준비할 인증 절차부터 알려 줘.

에이전트가 안내할 재료는 [실습 ZIP](https://github.com/SunCreation/mcp-skill-practice/releases/download/v0.1.0/mcp-skill-practice.zip)의 GOOGLE_AUTH_README.md입니다. 이 절의 설정은 그 안의 두 Python 프로그램을 위한 것입니다. 이미 사용하는 다른 Google MCP가 있다면 그 도구의 README를 대신 읽게 합니다. 이름이 모두 Google MCP여도 Desktop 앱, 웹앱, 서비스 계정 중 지원하는 방식이나 토큰 저장 형식이 다를 수 있습니다.

이번에는 '연습용 스프레드시트의 제목과 첫 세 줄 읽기'로 시작합니다. 사용할 MCP가 요구하는 API, OAuth 앱 종류, 인증 파일 경로 설정, 최초 로그인 명령을 안내에서 찾습니다. 앱 종류를 추측하지 않습니다. 아래는 Desktop app을 지원하는 도구의 경로입니다. Web application을 요구하는 도구라면 그 안내에 나온 redirect URI(로그인 후 결과를 돌려받는 주소)를 정확히 등록해야 합니다.

### 실습 프로젝트와 API

[Google Cloud Console](https://console.cloud.google.com/)을 열고 오른쪽 위 계정에서 이번 실습에 사용할 개인 계정인지 먼저 확인합니다. 회사 계정으로 열려 있다면 개인 계정으로 전환한 뒤 진행합니다. 기존 캡처의 이름이나 이메일을 입력하는 것이 아니라 자신의 계정으로 만드는 작업입니다.

Cloud 프로젝트는 API 설정과 앱 인증 정보를 묶어 관리하는 단위입니다. 내 컴퓨터의 실습 폴더나 Google Drive의 폴더와는 다른 곳에 있습니다. 실습 폴더에는 Python 파일을 두고, Cloud 프로젝트에는 '이 프로그램이 사용할 Google 기능과 앱 설정'을 둔다고 연결해 보세요. Google 표 자체를 이 프로젝트로 옮기지는 않습니다.

상단 프로젝트 선택기에서 실습용 프로젝트를 선택하거나 New Project로 새로 만듭니다. 새 프로젝트를 만드는 경우 아래 화면에서 진행합니다.

![실습용 Google Cloud 프로젝트 만들기](google-workspace-project.jpg)<span class="img-caption">프로젝트 생성 화면: Project name에 구분하기 쉬운 이름을 입력하고 왼쪽 아래 Create를 누른다. 실습에서는 Workspace OAuth Practice처럼 목적을 나타내는 이름을 사용한다. 캡처 안의 이름은 촬영 당시 예시다. 프로젝트 ID는 전역에서 고유해야 하므로 화면 예제와 달라도 된다.</span>

생성을 마친 뒤 상단 프로젝트 선택기에 방금 만든 이름이 표시되는지 확인합니다. 프로젝트를 여러 개 사용하는 경우 이 확인이 도움이 됩니다. 엉뚱한 프로젝트에서 API를 켠 뒤 다른 프로젝트의 인증 JSON을 받으면 설정은 각각 존재해도 함께 작동하지 않기 때문입니다.

선택한 프로젝트에서 APIs & Services → Library를 열고 Google Sheets API를 찾아 Enable(사용 설정)합니다. Library는 프로젝트에서 사용할 Google API를 선택하는 목록입니다. 이 작업은 '우리 프로그램에서 표 API를 사용할 준비'를 하는 단계이고, 어느 사용자의 어느 표를 읽을지는 아직 정하지 않았습니다.

![Google Sheets API 사용 설정](google-workspace-enable-api.jpg)<span class="img-caption">상단의 실습 프로젝트를 확인하고 Enable을 누른다. API를 켜는 것과 내 표 접근을 허용하는 OAuth 동의는 별개다.</span>

| 하려는 작업 | 확인할 API |
| :--- | :--- |
| 표의 셀 읽기 | Google Sheets API |
| 문서 본문 읽기 | Google Docs API |
| 파일 이름 검색·목록 조회 | Google Drive API |
| 메일 읽기 | Gmail API |
| 일정 읽기 | Google Calendar API |

Docs/Sheets 도구가 파일을 찾기 위해 Drive API도 사용할 수 있으므로 도구 문서의 요구 사항을 함께 봅니다. API 사용 설정은 이 프로젝트에서 API를 쓸 준비이며, 내 문서 접근에 대한 사용자 동의와는 별개입니다. [Sheets Python 시작 안내](https://developers.google.com/workspace/sheets/api/quickstart/python)

### 동의 화면과 테스트 사용자

이제 사용자가 로그인할 때 보게 될 앱의 소개를 준비합니다. Google Auth platform → Branding을 열고 처음이라면 Get started를 선택합니다. App name에는 Sheets Practice처럼 나중에 알아볼 수 있는 이름을 넣습니다. 지원 이메일은 동의 화면을 본 사용자가 문의할 연락처이고, 연락 이메일은 앱 설정과 관련된 안내를 받을 주소입니다. 혼자 시험하는 앱에서는 자신의 주소로 준비할 수 있습니다.

Audience는 이 앱을 누가 이용할지 정하는 곳입니다. 개인 Google 계정 실습은 External을 선택합니다. External은 곧바로 앱을 공개 게시한다는 뜻이 아닙니다. 여기서는 Testing 상태로 두고, Test users에 등록한 계정으로 개발 중인 앱을 시험합니다. Internal은 Google Workspace 또는 Cloud Identity 조직 내부를 대상으로 하는 선택이며, 조직 조건을 충족할 때 사용할 수 있습니다. 개인 계정 실습에서는 회사용 Internal 안내를 그대로 따라가지 않습니다.

![OAuth 앱 이름과 지원 이메일 설정](google-workspace-branding.jpg)<span class="img-caption">App name은 나중에 로그인 동의 화면에 나타난다. 지원 이메일과 연락 이메일은 자기 주소로 입력한다.</span>

![개인 Google 계정의 External 선택](google-workspace-external.jpg)<span class="img-caption">개인 계정에서는 Internal을 선택할 수 없다. External의 Testing으로 시작하고 다음 단계에서 자기 계정을 Test users에 추가한다. 프로젝트 설정 마지막에는 Google API 사용자 데이터 정책을 읽고 동의하는 단계가 있다.</span>

아래 Test users에는 나중에 브라우저 로그인에서 선택할 개인 이메일을 추가합니다. 이 목록은 앱을 시험할 사용자를 정합니다. Cloud 프로젝트 설정을 편집할 사람이나 표 파일을 공유받을 사람을 정하는 목록과는 다릅니다. 프로젝트를 편집할 수 있어도 다른 계정으로 로그인하면 테스트 사용자 제한에 걸릴 수 있습니다. [동의 화면과 scope 설정](https://developers.google.com/workspace/guides/configure-oauth-consent)

Audience의 Test users → Add users에서 실제 로그인할 자기 이메일을 입력하고 Save합니다. 화면에는 1 user (1 test, 0 other) 집계가 보입니다. 위쪽에 Branding 미완료 안내가 보이면 Branding에서 앱 이름·지원 이메일·연락 이메일 등의 누락 정보를 확인합니다. 이 실습에서는 Testing을 유지한 채 이후 실제 인증과 읽기에 성공했습니다. 공개 배포를 위해 임의의 홈페이지나 개인정보처리방침 주소를 넣지 않습니다.

![Testing 상태와 등록한 테스트 사용자](google-workspace-test-user.jpg)<span class="img-caption">Test users에 실제 로그인할 개인 이메일을 등록한 화면. 1 user (1 test, 0 other) 집계를 확인한다.</span>

### 읽기 범위와 파일 접근

이제 Data Access에서 요청할 scope를 정합니다. scope는 API를 통해 어떤 종류의 작업을 하도록 요청하는지를 나타냅니다. 이번 코드가 요청하는 값은 다음 하나입니다.

```python
SCOPES = ["https://www.googleapis.com/auth/spreadsheets.readonly"]
```

끝의 readonly는 읽기 전용이라는 뜻입니다. 이 문자열은 사람이 보기 좋은 이름 대신 Google이 구별하는 권한 식별자이므로 정확한 값을 사용합니다. 표에 내용을 쓰는 기능이 필요해지면 코드의 기능뿐 아니라 요청할 권한도 다시 검토해야 합니다.

Scope와 파일 공유 권한은 서로 다른 조건입니다. 내 계정이 '표 읽기'를 동의했다고 해도 공유받지 않은 다른 사람의 비공개 표까지 읽을 수는 없습니다. 반대로 내 계정에 편집 권한이 있는 표라도 이번 읽기 전용 토큰으로 수정 API를 호출할 수는 없습니다. 어떤 계정이 어떤 파일에 접근할 수 있는지와, 그 계정이 이 프로그램에 어떤 작업을 허용했는지를 함께 확인합니다.

콘솔 Data Access의 선언, Python의 SCOPES, 실제 로그인 동의 화면을 이어서 봅니다. 콘솔에서 범위를 저장했다는 사실만으로 프로그램의 요청이나 기존 토큰의 권한이 자동으로 바뀌지는 않습니다.

Data Access → Add or remove scopes에서 https://www.googleapis.com/auth/spreadsheets.readonly를 선택합니다. 목록에서 찾기 어렵다면 Manually add scopes에 전체 문자열을 입력하고 Add to table → Update → Save 순서로 저장합니다. See all your Google Sheets spreadsheets는 접근 가능한 표 전체에 대한 읽기 권한이며, 예제 표 하나만 허용하는 범위가 아닙니다.

![Sheets 읽기 범위 저장 화면](google-workspace-scopes.jpg)<span class="img-caption">Data Access에서 spreadsheets.readonly 범위를 추가하고 저장하는 화면.</span>

### OAuth 클라이언트와 인증 JSON

Google Auth platform → Clients → Create client → Desktop app을 선택하고 이름을 정합니다. 생성 후 JSON을 다운로드합니다. 이름을 credentials.json으로 바꾸라는 도구는 그 안내를 따릅니다. 프로젝트 코드 밖의 개인 설정 폴더에 보관하고 경로를 기억합니다. 이 JSON은 앱 설정이며, 아직 사용자 로그인이 끝나 생성되는 token 파일과는 다릅니다. [Desktop app 클라이언트 생성](https://developers.google.com/workspace/guides/create-credentials#desktop-app)

로컬 프로그램이 브라우저 로그인 결과를 돌려받는 실습이므로 Desktop app을 선택했습니다. Web application용 redirect URI 입력란을 찾아 임의로 채우지 않습니다. 클라이언트 이름은 콘솔에서 구분하는 이름입니다.

![Desktop app 클라이언트 생성 화면](google-workspace-desktop-client.jpg)<span class="img-caption">Clients → Create client에서 Desktop app을 선택하고 이름을 정하는 화면.</span>

생성 직후 Download JSON을 눌러 저장합니다. 이 그림은 비밀값이 있는 영역을 제외하고 실제 버튼 부분만 캡처했습니다. 촬영 당시 콘솔은 창을 닫으면 client secret을 다시 보거나 다운로드할 수 없다고 안내했습니다. 학생도 생성 창에 표시된 보관 안내를 읽고 다운로드 완료를 확인합니다.

![OAuth 생성 창의 Download JSON 버튼 부분](google-workspace-download-json.jpg)<span class="img-caption">비밀값 영역을 제외하고 Download JSON 버튼 부분만 캡처한 화면.</span>

이제 이름이 비슷한 두 파일을 구분해 두면 다음 단계가 쉬워집니다.

| 파일 | 만들어지는 순간 | 이 실습에서의 역할 |
| :--- | :--- | :--- |
| credentials.json | Cloud 콘솔에서 앱 클라이언트를 만들고 다운로드 | 어떤 앱이 로그인을 요청하는지 식별하는 설정 |
| token.json | 프로그램 실행 후 사용자가 로그인하고 동의 | 동의한 사용자로 API를 요청하고 인증을 갱신하는 정보 |

Credential은 인증에 쓰는 정보를 넓게 부르는 말입니다. 모든 도구에서 반드시 credentials.json이라는 파일 하나를 의미하는 것은 아닙니다. 이번 프로그램이 이 이름을 사용하므로 파일명까지 맞춰 보는 것입니다.

이번 예제에서는 파일을 ~/.config/google-workspace/credentials.json에 보관합니다. ~는 자기 사용자 홈 폴더입니다. 이후 인증 프로그램은 같은 폴더에 token.json을 생성합니다. 둘 다 실습 ZIP이나 Git 저장소에 넣지 않습니다.

다운로드한 내용이나 토큰을 에이전트 대화에 붙여 넣지 않습니다. 도구에는 파일을 읽을 경로를 설정합니다. 과제에는 파일 내용 대신 '어느 설정 항목에 경로를 지정했는가'만 적습니다. 저장소 안에 둘 수밖에 없다면 .gitignore로 버전 관리에서 제외하고 이미 추적되고 있지 않은지도 확인합니다. .gitignore는 Git이 새 파일을 추적하지 않도록 지정하는 목록이며, 이미 올라간 비밀값을 지워 주지는 않습니다.

에이전트에는 다운로드한 파일의 경로를 알려 주고 요청합니다.

> 인증 JSON은 준비했어. 이 도구가 사용할 경로를 설정해 줘. 내용은 출력하지 말아 줘.

> [!info]+ **수동으로 설정하기 — 인증 JSON 위치**
> 홈 폴더에 .config/google-workspace 폴더를 만들고 다운로드한 JSON을 credentials.json이라는 이름으로 옮깁니다. 아래 첫 줄은 보관 폴더를 만들고, 둘째 줄은 예시 다운로드 파일을 옮깁니다. 둘째 줄의 파일 경로는 실제 다운로드 위치로 바꿉니다. 이미 같은 이름의 인증 파일을 사용 중이라면 다른 보관 폴더를 정하고 경로 설정으로 구분합니다.
>
> ```bash
> mkdir -p ~/.config/google-workspace
> mv "/실제/다운로드한/파일.json" ~/.config/google-workspace/credentials.json
> ```
>
> 기본 위치를 쓰면 별도 환경변수 설정이 필요 없습니다. 다른 위치를 쓰는 경우 CREDENTIALS_PATH는 앱 JSON, TOKEN_PATH는 로그인 결과를 저장할 파일 경로입니다. 환경변수는 실행할 프로그램에 외부에서 전달하는 이름·값 설정입니다. 이 두 이름은 첨부 코드의 약속이며 모든 Google 도구의 공통 이름은 아닙니다.
>
> ```bash
> export CREDENTIALS_PATH="/내/개인설정/google/credentials.json"
> export TOKEN_PATH="/내/개인설정/google/token.json"
> ```

### 최초 로그인과 API 읽기

실습 ZIP에는 GOOGLE_AUTH_README.md, 최초 인증용 google_auth_check.py, 읽기 MCP 서버 google_sheet_reader.py도 들어 있습니다. 이 예제는 spreadsheets.readonly만 요청하고, 인증 확인 프로그램은 Google의 공개 예제 표를 읽습니다. 표에서 A·B·C는 열 이름이고 1·2·3은 행 번호입니다. A1:C3은 A열 1행부터 C열 3행까지를 뜻하며 헤더가 첫 행이면 헤더도 포함합니다. MCP 서버는 사용자가 지정한 표의 제목과 첫 시트 A1:C3을 읽습니다. 지금은 이 프로그램으로 최초 로그인과 API 읽기를 확인하고, 그다음 MCP 서버를 등록합니다. 기존 Google MCP를 사용한다면 그 MCP의 scope·파일 형식을 다시 확인해야 하며, 이 예제의 token 파일을 무조건 재사용하지 않습니다.

> 이 폴더의 Google 인증 안내를 읽고 첫 로그인을 도와줘. 로그인 뒤 예제 표가 읽히는지도 확인하자.

에이전트에게 GOOGLE_AUTH_README.md와 인증 파일의 경로를 알려 줍니다. 계정 선택과 권한 동의는 학생이 브라우저에서 확인합니다.

> [!info]+ **수동으로 설정하기 — 최초 인증과 API 읽기**
> 첨부 예제는 앞 절에서 작업한 mcp-skill-practice 폴더에서 실행합니다. 이 폴더에는 이미 pyproject.toml, uv.lock, .python-version이 들어 있습니다. pyproject.toml에는 필요한 라이브러리와 Google용 선택 묶음이 선언되어 있고, uv.lock에는 설치할 버전이 기록되어 있습니다. .python-version은 이 실습에서 사용할 Python 3.13을 지정합니다. 새 프로젝트를 만드는 uv init은 실행하지 않습니다.
>
> **Extra(선택 의존성 묶음)**는 특정 기능을 사용할 때 기본 의존성에 더할 라이브러리 묶음입니다. 이 실습의 google extra는 기본 MCP 환경에 Google API와 인증용 라이브러리를 추가합니다. --extra google을 지정하면 이 묶음까지 설치 대상으로 선택합니다. 별도의 Google 프로젝트나 가상환경을 새로 만드는 옵션은 아닙니다.
>
> 실습 폴더에서 다음 두 명령을 실행합니다. 앞 절의 .venv가 있으면 같은 환경을 사용하고, 없으면 uv sync가 만듭니다.
>
> ```bash
> uv sync --locked --extra google
> uv run --locked --extra google python google_auth_check.py
> ```
>
> 첫 명령은 uv.lock에 기록된 버전으로 기본 MCP와 Google용 의존성을 준비합니다. 둘째 명령은 그 프로젝트 환경에서 인증 프로그램을 실행합니다. --locked는 잠금 파일을 임의로 바꾸지 않도록 하며, pyproject.toml과 맞지 않으면 오류로 알려 줍니다. 실행 명령에도 --extra google을 명시하면 필요한 Google 라이브러리를 실행 전에 확인하고 준비할 수 있습니다.
>
> 첨부한 requirements.txt와 requirements-google.txt는 기존 설치 방식과의 호환을 위한 참고 파일입니다. 이 본문의 설치와 실행 기준은 pyproject.toml과 uv.lock입니다. 에이전트에게 이번 설치와 실행이 이 실습 폴더의 .venv를 사용하는지 확인하게 합니다. 다른 실습의 가상환경이 활성화되어 있다면 현재 폴더의 환경에 맞춘 뒤 진행합니다.

이 명령을 실행하면 인증 JSON을 읽고 기본 브라우저를 엽니다. 아직 로그인한 적이 없으므로 프로그램은 지금 이 순간에 사용자 동의를 요청합니다. 브라우저에 계정이 여러 개 있다면 콘솔의 Test users에 등록한 개인 계정을 고릅니다.

프로그램은 로그인 결과를 받기 위해 잠시 localhost 주소에서 기다립니다. localhost는 지금 프로그램을 실행하는 자기 컴퓨터를 가리킵니다. 이 예제는 그 컴퓨터 안에서만 결과를 받고, 사용하지 않는 포트를 골라 잠시 엽니다. 포트는 같은 컴퓨터 안의 여러 통신 대상을 구분하는 번호입니다. 학생이 별도의 로그인 웹사이트를 인터넷에 배포하는 작업은 아닙니다.

로그인 후에는 이 프로그램이 Google 공식 예제 표를 직접 읽습니다. MCP를 먼저 등록하지 않은 이유도 여기 있습니다. 이 단계의 결과를 보면 Google 인증과 API가 동작하는지 확인할 수 있고, 이후 호스트 연결에 문제가 생겨도 어느 단계까지 성공했는지 구분할 수 있습니다.

도구 문서의 로그인 명령 또는 최초 실행을 진행합니다. 열린 Google 화면에서 계정, 앱 이름, 요청 권한을 확인하고 직접 동의합니다. 인증이 완료되면 도구가 token 파일이나 자체 저장소에 인증 결과를 보관할 수 있습니다. JSON 다운로드만으로 로그인까지 끝났다고 보지 않습니다. 아래 동의 화면을 거치면 인증 프로그램이 공개 예제 표를 직접 읽습니다. 이 시점에는 호스트의 MCP 등록이 필요하지 않습니다.

![실습 앱의 Google 계정 선택](google-workspace-account.jpg)<span class="img-caption">콘솔에서 정한 앱 이름과 사용할 계정을 확인하는 화면. 새 실습 앱 이름은 Sheets Practice처럼 정하면 된다. 캡처 안의 이름은 촬영 당시 예시다.</span>

![직접 만든 테스트 앱의 미검증 안내](google-workspace-testing-warning.jpg)<span class="img-caption">직접 만든 Testing 앱의 안내. 앱 이름·개발자·계정을 확인한 이 실습에서는 Continue로 진행했다. 출처를 모르는 다른 앱의 경고를 무조건 넘기는 절차가 아니다.</span>

![Google Sheets 읽기 권한 동의 화면](google-workspace-consent.jpg)<span class="img-caption">권한 설명이 표 읽기인지 확인한 뒤 Continue를 누르는 화면. 콘솔에 범위를 저장했더라도 실제 실행 프로그램이 다른 권한을 요청할 수 있으므로 이 화면을 다시 확인한다.</span>

![API로 읽은 공개 예제 표의 실제 화면](google-workspace-sheet-result.jpg)<span class="img-caption">Google 공식 시작 예제의 Example Spreadsheet. 화면의 제목과 왼쪽 위 A1:C3을 아래 실제 반환값과 대조한다. 사용자 개인 자료를 읽은 예제가 아니다.</span>

실제 인증 후 실행한 프로그램의 반환값(2026-09-06):

```json
{
  "title": "Example Spreadsheet",
  "sheet": "Class Data",
  "range": "'Class Data'!A1:C3",
  "values": [
    ["Student Name", "Gender", "Class Level"],
    ["Alexandra", "Female", "4. Senior"],
    ["Andrew", "Male", "1. Freshman"]
  ]
}
```

완료 판단: 이번 환경에서는 Google 동의 후 localhost 완료 페이지가 Chrome에서 ERR_BLOCKED_BY_CLIENT로 보였지만, 로컬 프로그램은 인증 결과를 받아 위 API 응답을 출력하고 정상 종료했습니다. 이 메시지 하나로 성공·실패를 단정하지 않습니다. 먼저 터미널의 종료 상태와 실제 읽기 결과를 확인하고, 실패했다면 같은 프로그램의 인증 안내를 따릅니다. token 파일의 내용은 출력하지 않습니다.

### 토큰을 받아 다시 사용하는 코드

방금 브라우저에서 누른 동의가 코드의 어느 부분과 이어지는지 봅시다. 전체 구현은 실습 ZIP의 google_auth_check.py에 있습니다. 핵심 흐름은 인증 파일 경로 확인 → 기존 토큰 읽기 또는 최초 로그인 → 표 읽기입니다. 아래 코드는 실제 파일에서 발췌했습니다.

먼저 프로그램은 파일 위치와 실행 옵션을 읽고 다음과 같이 경로를 선택합니다.

```python
creds = (load_token(token_path) if token_path.exists() and not args.reauth else
         authenticate(credential_path, token_path, no_browser=args.no_browser,
                      auth_url_file=url_path, timeout=args.timeout))
```

token_path.exists()는 저장된 토큰 파일이 있는지 확인합니다. 토큰이 있고 재인증을 요청하지 않았다면 load_token()으로 읽습니다. 처음 실행해서 토큰이 없거나 --reauth를 선택했다면 authenticate()로 로그인합니다. 여기서 creds는 이후 Google API에 사용할 인증 객체입니다. 파일이 있다는 사실과 그 인증이 유효하다는 사실은 다르므로 실제 유효성 확인은 load_token() 안에서 계속합니다.

처음 로그인할 때는 다음 부분이 실행됩니다.

```python
flow = PrivateURLFlow.from_client_secrets_file(str(credentials_path), SCOPES)
flow.auth_url_file = auth_url_file
try:
    creds = flow.run_local_server(
        host="localhost", bind_addr="127.0.0.1", port=0,
        open_browser=not no_browser,
        authorization_prompt_message="",
        success_message="Authentication complete. You can close this tab.",
        timeout_seconds=timeout,
        prompt="consent",
    )
    check_scopes(creds)
    save_private(token_path, creds.to_json())
    return creds
finally:
    if flow.wrote_url and auth_url_file is not None:
        auth_url_file.unlink(missing_ok=True)
```

from_client_secrets_file()이 콘솔에서 다운로드한 앱 설정 JSON과 요청할 SCOPES를 읽습니다. 이 줄에서 Google 계정 로그인이 끝나는 것은 아닙니다. run_local_server()가 브라우저의 로그인·동의 흐름을 시작하고 그 결과를 받아 creds를 만듭니다. port=0은 사용할 수 있는 포트를 시스템이 고르도록 하는 값입니다. prompt="consent"는 사용자 동의 화면을 요청합니다.

반환된 인증을 check_scopes()로 확인하고, creds.to_json()으로 저장할 수 있는 형태로 바꿔 token.json에 기록합니다. 이 예제의 save_private()는 개인 설정 폴더에 임시 파일을 만든 뒤 교체하며 토큰 파일을 다른 사용자에게 공개하지 않도록 저장합니다. 토큰 자체를 보고서나 에이전트 답변에 출력할 이유는 없습니다.

PrivateURLFlow는 Google 라이브러리의 InstalledAppFlow를 확장한 클래스입니다. 표준 OAuth 절차를 새로 구현한 것이 아니라, 자동으로 브라우저를 열지 않는 실행에서도 로그인 URL을 별도 파일로 전달할 수 있도록 보조 동작을 더했습니다. 일반 실행에서는 기본 브라우저가 열립니다. auth_url_file 관련 줄과 마지막 파일 삭제는 이 보조 실행 경로를 처리하는 부분입니다.

다음번에 실행할 때는 다음 함수가 저장된 인증을 읽습니다.

```python
def load_token(path: Path) -> Credentials:
    if not path.is_file():
        raise ValueError("인증 토큰이 없습니다. google_auth_check.py를 먼저 실행하세요.")
    creds = Credentials.from_authorized_user_file(str(path))
    check_scopes(creds)
    if not creds.valid:
        if not creds.refresh_token:
            raise ValueError("재인증이 필요합니다. google_auth_check.py --reauth를 실행하세요.")
        creds.refresh(Request())
        check_scopes(creds)
        save_private(path, creds.to_json())
    return creds
```

Credentials.from_authorized_user_file()은 사용자 동의 뒤 만들어진 토큰 JSON을 읽습니다. 앞의 from_client_secrets_file()이 읽는 앱 JSON과 대상이 다릅니다. 두 함수가 어떤 파일을 읽는지 연결하면 인증 파일 이름이 비슷해도 헷갈리지 않습니다.

접근 토큰은 API 요청 때 쓰이며 유효 기간이 있습니다. 저장된 접근 토큰을 더 사용할 수 없더라도 갱신 토큰이 유효하면 creds.refresh(Request())가 Google에 새 접근 토큰을 요청합니다. 그래서 매번 표를 읽을 때마다 로그인할 필요가 줄어듭니다. 새 인증 결과를 다시 저장한 뒤 API 호출로 이어집니다. 갱신 토큰이 없거나 만료·철회된 경우에는 재로그인이 필요합니다. load_token() 자체는 브라우저를 열지 않으므로 실패하면 인증 프로그램에서 재인증합니다. [Google OAuth와 토큰 갱신](https://developers.google.com/identity/protocols/oauth2)

이 함수가 처음과 갱신 뒤 모두 호출하는 check_scopes()도 역할이 있습니다. 다른 도구에서 받은 넓은 권한의 토큰을 파일명만 바꿔 가져와 놓고 '읽기 전용 실습'이라고 생각하는 상황을 발견하려고 저장된 범위를 실습 범위와 대조합니다. 예제는 파일을 읽을 때 scopes를 새 값으로 덮어쓰지 않고, 원래 저장된 범위를 확인합니다.

### 표의 제목과 셀을 읽는 코드

이제 인증 결과를 이용해 데이터를 읽습니다. 스프레드시트 URL에는 문서 전체를 구분하는 ID가 있습니다. 다음 공개 예제에서는 /d/ 뒤에서 /edit 앞까지가 ID입니다. 뒤에 붙는 gid는 문서 안의 특정 시트를 가리키는 값으로 문서 ID와 다릅니다.

```text
https://docs.google.com/spreadsheets/d/1BxiMVs0XRA5nFMdKvBdBZjgmUUqptlbs74OgvE2upms/edit
                                      └ 문서 ID: /d/ 뒤부터 /edit 앞까지
```

한 스프레드시트 파일 안에는 여러 시트 탭이 있을 수 있습니다. A1:C3은 그 시트의 A열 1행부터 C열 3행까지, 최대 아홉 칸을 뜻합니다. 이번 프로그램은 첫 시트 이름을 알아낸 뒤 이 범위를 읽습니다. 이 범위는 코드가 정한 조회 범위입니다. 앞에서 동의한 spreadsheets.readonly 권한이 아홉 칸으로 제한되는 것은 아닙니다. [Sheets 권한 범위](https://developers.google.com/workspace/sheets/api/scopes)

실제 read_header() 함수는 다음과 같습니다.

```python
def read_header(creds: Credentials, spreadsheet_id: str) -> dict:
    if not re.fullmatch(r"[A-Za-z0-9_-]+", spreadsheet_id):
        raise ValueError("표 URL 전체가 아닌 spreadsheet_id를 입력하세요.")
    service = build("sheets", "v4", credentials=creds, cache_discovery=False)
    try:
        metadata = service.spreadsheets().get(
            spreadsheetId=spreadsheet_id,
            fields="properties(title),sheets(properties(title))",
        ).execute()
        sheets = metadata.get("sheets", [])
        if not sheets:
            raise ValueError("읽을 시트가 없습니다.")
        title = sheets[0]["properties"]["title"]
        selected_range = "'" + title.replace("'", "''") + "'!A1:C3"
        result = service.spreadsheets().values().get(
            spreadsheetId=spreadsheet_id, range=selected_range,
        ).execute()
        return {"title": metadata["properties"]["title"], "sheet": title,
                "range": result.get("range", selected_range),
                "values": result.get("values", [])}
    finally:
        service.close()
```

첫 검사인 re.fullmatch()는 입력 전체가 ID에 사용할 수 있는 문자로만 이루어져 있는지 확인합니다. URL을 그대로 넣으면 / 같은 문자가 있으므로 여기서 막습니다. 실제 문서가 존재하고 접근 가능한지는 이후 Google API 응답에서 확인합니다.

build("sheets", "v4", credentials=creds, ...)는 Sheets API v4를 호출할 클라이언트를 준비합니다. 여기서의 클라이언트는 앞서 배운 MCP 클라이언트와 별개의 Google API 호출용 Python 객체입니다. 이 프로그램이 Google과 통신할 때 사용할 인증을 credentials=creds로 전달합니다.

데이터를 요청하는 곳은 두 군데입니다. 첫 get(...).execute()는 문서 제목과 시트 제목을 가져옵니다. fields는 필요한 응답 항목을 지정하므로 모든 메타데이터를 가져오지 않아도 됩니다. 메타데이터는 셀의 실제 업무 기록이 아니라 문서 제목이나 시트 목록처럼 자료를 설명하는 정보입니다.

첫 시트의 이름을 찾은 뒤 두 번째 values().get(...).execute()에서 셀 값을 읽습니다. selected_range가 'Class Data'!A1:C3이라면 Class Data 시트의 왼쪽 위 아홉 칸을 요청하는 것입니다. 작은따옴표와 replace()는 시트 이름에 공백이나 작은따옴표가 있어도 올바른 범위 문자열이 되도록 처리합니다.

마지막 딕셔너리는 학생과 에이전트가 읽을 결과 형식입니다. 문서 제목 title, 시트 이름 sheet, 실제 범위 range, 행별 값 values를 묶습니다. values의 안쪽 목록 하나가 한 행입니다. Sheets API는 뒤쪽의 빈 행·열을 생략할 수 있으므로 모든 요청 범위가 항상 같은 크기의 목록으로 반환된다고 가정하지 않습니다. 마지막의 finally는 읽기에 성공하거나 실패해도 API 클라이언트를 닫습니다. [셀 값 응답 형식](https://developers.google.com/workspace/sheets/api/reference/rest/v4/spreadsheets.values#ValueRange)

### 읽기 함수를 MCP로 공개

지금까지는 학생이 인증 프로그램을 실행했습니다. 이제 AI가 필요할 때 같은 기능을 요청할 수 있도록 MCP 도구로 공개합니다. google_sheet_reader.py의 전체 코드는 다음과 같습니다.

```python
"""미리 인증한 읽기 전용 토큰으로 지정한 표의 첫 시트 A1:C3만 읽습니다."""
from mcp.server import MCPServer
from mcp.server.mcpserver.exceptions import ToolError

from google_auth_check import configured_path, load_token, read_header

mcp = MCPServer("google-sheets-reader")


@mcp.tool()
def read_sheet_header(spreadsheet_id: str) -> dict:
    """지정한 Google 표의 제목과 첫 시트 A1:C3을 읽습니다. 생성·수정하지 않습니다.

    spreadsheet_id는 표 URL의 /d/와 /edit 사이 ID입니다.
    인증이 안 되어 있으면 터미널에서 google_auth_check.py를 먼저 실행하세요.
    """
    try:
        token = load_token(configured_path("TOKEN_PATH", "token.json"))
        return read_header(token, spreadsheet_id)
    except Exception as error:
        raise ToolError(
            "Google 읽기 실패. 인증 확인 스크립트와 지정 표의 접근 권한을 확인하세요 "
            f"({type(error).__name__})."
        ) from None
```

새로 생긴 핵심은 @mcp.tool()입니다. 앞서 만든 read_header()를 그대로 이용하되, read_sheet_header라는 이름과 입력 spreadsheet_id를 호스트가 발견하고 호출할 수 있게 등록합니다. 함수 설명은 도구를 고르는 모델에게 어떤 표를 얼마나 읽는지 알려 줍니다. 도구에 URL이 아니라 ID가 필요하다는 안내도 여기에 들어 있습니다.

함수 안에서는 먼저 TOKEN_PATH에서 토큰을 읽고 read_header()에 전달합니다. MCP 호출 도중 브라우저에서 첫 로그인을 기다리게 하지 않고, 준비된 인증으로 바로 읽는 구조입니다. 필요한 재로그인은 인증 프로그램이 맡습니다. 예외가 나면 ToolError를 통해 호스트가 읽을 수 있는 오류를 돌려줍니다. Google API의 긴 응답 원문을 그대로 대화에 흘리는 대신, 인증 확인 프로그램과 해당 표의 접근 권한을 확인할 단서를 남깁니다.

앞 단계에서 로그인과 API 읽기를 마쳤다면 에이전트에게 파일 경로와 실습 폴더를 알려주고 요청합니다.

> 로그인하고 예제 표 읽기까지 확인했어. 이제 이 폴더의 Google Sheets 도구를 MCP로 연결해 줘.

> [!info]+ **수동으로 설정하기 — Google Sheets 읽기 MCP**
> 앞 단계의 uv sync --locked --extra google 준비와 google_auth_check.py 인증을 마친 상태에서 등록합니다. 아래 /내/실습폴더는 압축을 푼 실제 절대 경로로 바꿉니다. Claude Code 명령의 경로를 감싼 큰따옴표는 공백이 있는 폴더 이름도 하나의 경로로 전달하도록 유지합니다.
>
> Claude Code
>
> ```bash
> claude mcp add --transport stdio --scope project google-sheets-reader -- uv --directory "/내/실습폴더" run --locked --extra google mcp run google_sheet_reader.py --transport stdio
> ```
>
> OpenCode V1 — 앞에서 확인한 설치 버전이 V1이면 기존 mcp 객체에 추가합니다.
>
> ```json
> {
>   "mcp": {
>     "google-sheets-reader": {
>       "type": "local",
>       "command": ["uv", "--directory", "/내/실습폴더", "run", "--locked", "--extra", "google", "mcp", "run", "google_sheet_reader.py", "--transport", "stdio"],
>       "enabled": true
>     }
>   }
> }
> ```
>
> OpenCode V2 — 설치 버전이 V2라면 위 V1 객체 대신 다음처럼 mcp.servers 아래에 서버를 추가합니다. 기존 다른 서버 설정은 함께 유지합니다. V2에는 enabled 필드가 없으며, disabled를 생략하면 기본값 false로 연결합니다. codemode도 생략하면 기본값 true이므로 앞에서 배운 Code Mode 경로로 도구를 제공합니다.
>
> ```json
> {
>   "mcp": {
>     "servers": {
>       "google-sheets-reader": {
>         "type": "local",
>         "command": ["uv", "--directory", "/내/실습폴더", "run", "--locked", "--extra", "google", "mcp", "run", "google_sheet_reader.py", "--transport", "stdio"]
>       }
>     }
>   }
> }
> ```
>
> --directory는 서버를 실행할 실습 폴더를 지정합니다. 세 설정 모두 그 폴더의 프로젝트 환경에서 실행하며, --locked --extra google로 잠금 파일과 Google용 의존성을 확인합니다. 서버를 실행하는 명령은 같고, 호스트가 설정을 읽는 위치와 켜기·끄기 항목이 달라진 것입니다. Google 사용자 OAuth는 앞에서 실행한 Python 프로그램이 처리합니다. 이 로컬 MCP 설정에 원격 MCP용 OAuth 항목을 추가하는 단계는 아닙니다.
>
> 기본 인증 파일 위치는 ~/.config/google-workspace/입니다. 위치를 바꿨다면 최초 인증 프로그램에는 CREDENTIALS_PATH와 TOKEN_PATH를 지정하고, MCP 서버에는 TOKEN_PATH를 지정합니다. Claude Code는 서버 설정의 env, OpenCode는 environment에 넣습니다. 아래는 환경변수 부분의 예시이며 전체 서버 설정은 아닙니다.
>
> ```json
> {
>   "TOKEN_PATH": "/내/개인설정/google/token.json"
> }
> ```
>
> 이 변수 이름은 첨부한 실습 코드 기준입니다. 다른 MCP에서는 해당 도구의 문서를 따릅니다. 인증 파일 내용을 설정에 복사하지 않습니다.

에이전트가 GOOGLE_AUTH_README.md를 읽고 현재 호스트에 맞게 설정하도록 합니다. 학생은 선택한 서버 파일과 인증 파일 위치가 맞는지 확인합니다. MCP마다 경로 설정 이름과 토큰 형식이 다르므로 다른 MCP로 바꿀 때는 그 도구의 안내를 다시 읽게 합니다. Skill에서는 인증 정보를 직접 담지 않고, 연결된 MCP나 스크립트를 이용하는 절차를 안내합니다.

### 호스트의 호출과 원본 대조

MCP 실제 검증 기록: 같은 인증으로 SDK 클라이언트에서 read_sheet_header를 호출하여 위 표 제목·범위·내용을 다시 확인했습니다. 이 기록은 Claude Code·OpenCode 모델의 도구 선택까지 검증한 기록은 아니며, 학생은 아래 단계에서 자기 호스트의 실제 호출을 확인합니다.

호스트에서 연결 상태를 확인한 뒤 표 URL과 함께 요청합니다.

> 이 표의 제목과 처음 세 줄을 읽어 줘. 화면과 결과를 비교하자.

표 URL을 함께 주고 호출 기록을 살펴봅니다. 연결 목록에 google-sheets-reader가 보이면 서버와 연결된 것입니다. 이어 read_sheet_header가 선택되었는지, spreadsheet_id에 URL 속 ID가 전달되었는지, 반환된 제목과 행이 원본과 같은지 확인합니다. 연결 표시·실제 도구 호출·원본 대조가 각각 다른 관찰입니다.

학생이 자신의 표 URL을 주면 제목·내용도 그 표에 맞게 달라집니다. 공개 예제를 다시 호출한 경우 title은 문서 상단의 Example Spreadsheet, sheet는 아래쪽 탭의 Class Data에 대응합니다. values의 첫 목록은 헤더이고 두 번째와 세 번째 목록은 화면의 2·3행입니다. 요청에 '처음 세 줄'을 썼다고 헤더 다음 세 명의 데이터가 오는 것은 아닙니다. 이 도구는 헤더를 포함한 1–3행을 읽도록 작성했습니다. 표를 읽는 기능을 연결한 것이므로, 이 순서만 적었다고 별도의 Skill이 필요한 것은 아닙니다. 앞에서 만든 Skill이 Google 자료를 필요로 한다면 이 MCP를 자료를 읽는 수단으로 사용할 수 있습니다. 인증 JSON이나 토큰은 Skill 본문에 넣지 않습니다.

실제 표 화면과 결과를 비교합니다. 읽기 전용 scope라도 접근 가능한 파일이 한 개로 제한된다는 뜻은 아니므로, 도구가 허용받은 실제 범위를 별도로 확인합니다. [Google OAuth 처리 흐름](https://developers.google.com/identity/protocols/oauth2)

### 팀 업무 자료로 연결 확장

공개 예제 표는 인증과 읽기 경로를 확인하기 위한 자료입니다. 반환된 학생 명단을 앞의 주간 보고서 데이터로 사용하는 것은 아닙니다. 팀 보고서와 연결하려면 앞서 사용한 week-one·week-two의 업무 기록을 연습용 Google 표에 옮기고, 열 이름과 값의 의미를 정하는 단계가 더 필요합니다.

예를 들어 업무 ID, 주차, 진행 기록, 배포 상태, 운영 확인 기록이 각각 어디에 들어가는지 먼저 정합니다. 그다음 필요한 주차의 행을 읽어 기존 weekly-records 결과와 같은 의미의 데이터로 돌려주도록 도구를 바꿀 수 있습니다. 이때 현재의 A1:C3만으로 모든 업무를 가져올 수 있는지 학생이 판단해야 합니다. 이번 인증 성공은 그 확장에 필요한 연결 기반을 확인한 것입니다.

> 우리 주간 기록을 이 연습용 표에서 읽고 싶어. 지금 도구에서 무엇을 바꾸면 될지 같이 살펴보자.

에이전트와 바꿀 열·조회 범위를 정한 뒤, 한 업무가 원본 JSON과 같은 의미로 반환되는지 먼저 비교해 보세요. 보고서의 완료·진행·위험 기준은 계속 Skill이 제공할 수 있습니다. 데이터의 위치를 Google 표로 바꿨다고 팀 규칙까지 새로 만들어야 하는 것은 아닙니다. 반대로 데이터 연결 설명만 적은 문서를 억지로 별도 Skill로 만들 필요도 없습니다.

### 바이브코딩으로 인증을 준비할 때

> 이 Google Sheets 도구를 연결하고 싶어. 안내 문서를 보고 내가 해야 할 인증 준비를 순서대로 알려 줘.

> 인증 파일은 준비했어. 파일 내용은 출력하지 말고 경로 설정을 도와줘.

에이전트에게 모든 설정값을 미리 장문으로 지정할 필요는 없습니다. 학생은 사용할 서비스와 작업 범위를 고르고, 에이전트는 그 도구의 안내를 읽어 필요한 준비를 설명하게 합니다. Google 계정 로그인과 권한 동의는 학생이 화면에서 직접 확인합니다.

### 연결이 멈춘 단계 찾기

예를 들어 로그인은 끝났는데 표를 읽지 못했다면, 앱 이름부터 다시 만드는 것보다 '어느 계정이 동의했고 그 계정이 해당 표를 볼 수 있는가'를 먼저 확인하는 편이 원인에 가깝습니다. API 읽기는 되는데 호스트에서만 실패한다면 두 프로그램이 같은 TOKEN_PATH와 가상환경을 사용하는지도 살펴봅니다.

| 오류·현상 | 확인 순서 |
| :--- | :--- |
| access_denied / 테스트 사용자 제한 | 동의 화면에서 취소·거부했는지, 로그인 계정과 Test users 등록, 조직 정책 |
| redirect_uri_mismatch | 도구가 요구하는 앱 종류와 등록한 반환 주소 |
| API가 꺼져 있다는 오류 | credential을 만든 같은 프로젝트에서 API 사용 설정 |
| insufficient permissions | 요청 scope, 실제 동의한 범위, 대상 파일 접근 권한 |
| invalid_grant / 재로그인 필요 | 도구의 재인증 안내 확인; 토큰 만료·철회 가능성 |

Sheets·Drive 등의 권한을 요청하는 External 앱이 Testing 상태이면 갱신 토큰은 7일 후 만료됩니다. 이름·이메일·프로필 등 기본 사용자 정보 범위만 요청한 경우는 예외입니다. 수업 후 재로그인이 필요한 상황을 코드 오류로만 보지 않습니다. 권한을 바꿨다면 기존 인증에 새 권한이 자동 반영되는지 가정하지 말고 도구의 재인증 안내를 따릅니다. [Google 토큰 만료 설명](https://developers.google.com/identity/protocols/oauth2#expiration)

연결 기록에는 선택한 API와 계정 종류, 앱 종류, 동의한 작업 범위, 실제 호스트 호출의 입력과 읽기 결과를 남깁니다. 인증 JSON·토큰 원문은 제출하지 않습니다. 계정 정책 때문에 진행이 멈췄다면 어느 단계에서 어떤 오류가 발생했는지 적습니다. 원인을 모른 채 모든 설정을 처음부터 다시 만드는 대신, 위 오류 표에서 해당 단계를 찾아볼 수 있습니다.

이제 로컬 파일, 계산을 실행할 노트북, 팀의 공유 표라는 서로 다른 위치를 살펴봤습니다. 다음에는 자신의 과업에 필요한 자료와 판단 기준을 고르고, 오늘 만든 연결과 Skill 중 무엇을 재사용할지 정합니다.

---

## 전체 커리큘럼 구성표

|  강좌 번호   | 강좌명                                     |
| :------: | :--------------------------------------- |
|  **1강**  | [[AA05-01 에이전트와 도구 연결]]           |
|  **2강**  | [[AA05-02 MCP 연결 구조와 메시지의 흐름]]           |
|  **3강**  | [[AA05-03 업무 기록 MCP 만들기]]           |
|  **4강**  | [[AA05-04 호스트 연결과 첫 도구 호출]]           |
|  **5강**  | [[AA05-05 많은 도구와 문맥 부담]]           |
|  **6강**  | [[AA05-06 Claude Code의 Tool Search]]           |
|  **7강**  | [[AA05-07 OpenCode의 Code Mode]]           |
|  **8강**  | [[AA05-08 Skill의 역할과 업무 지식]]           |
|  **9강**  | [[AA05-09 팀 보고서 첫 작성]]           |
|  **10강**  | [[AA05-10 완성한 업무 방식을 Skill로 남기기]]           |
|  **11강**  | [[AA05-11 새 업무 기록에서 Skill 재사용하기]]           |
|  **12강**  | [[AA05-12 Colab MCP 연결과 실행]]           |
|  **13강**  | **13강. Google Workspace 자료를 읽는 인증과 연결 (현재)**           |
|  **14강**  | [[AA05-14 자기 과업에 맞게 확장하기]]           |
|  **15강**  | [[AA05-15 실행 경험과 Skill 개선 연구]]           |
