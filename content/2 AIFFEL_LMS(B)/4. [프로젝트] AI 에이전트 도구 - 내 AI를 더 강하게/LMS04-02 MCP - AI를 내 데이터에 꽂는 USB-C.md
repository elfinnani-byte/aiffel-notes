# [4.프로젝트 AI 에이전트 도구 - 내 AI를 더 강하게] - 2. MCP - AI를 내 데이터에 꽂는 USB-C

---

## MCP - AI를 내 데이터·도구에 꽂는 'USB-C'

### 문제: AI는 '내 것'을 모릅니다

아무리 똑똑한 AI도 기본적으로는 **자기가 학습한 것만** 압니다. 내 데이터베이스, 내 파일, 내 캘린더, 우리 회사 문서는 모릅니다. 그때그때 복사해서 붙여넣을 수도 있지만, 연결할 도구가 많아지면 **"AI 3개 × 도구 5개 = 연결 15개"** 를 일일이 만들어야 하는 지옥이 됩니다. (이걸 M×N 문제라고 부릅니다.)

### MCP = "AI를 위한 USB-C 포트"

**MCP(Model Context Protocol)** 는 이 문제를 푸는 **표준 규격**입니다. Anthropic이 2024년 말에 공개했고, 지금은 업계 표준이 되었습니다.

USB-C를 떠올려 보세요. 예전엔 기기마다 충전기가 달랐지만, USB-C 하나로 통일되니 아무 기기나 아무 충전기에 꽂힙니다. **MCP는 'AI를 위한 USB-C'** 입니다 - 도구를 한 번 MCP 규격으로 만들어 두면, MCP를 지원하는 **아무 AI에나 꽂힙니다.** (출처: [modelcontextprotocol.io, "What is MCP"](https://modelcontextprotocol.io/docs/getting-started/intro); [IBM, "What is Model Context Protocol?"](https://www.ibm.com/think/topics/model-context-protocol))

무엇을 할 수 있냐면 -

- AI가 **내 데이터베이스를 직접 조회**해서 답하기
- **Figma 디자인**을 읽어서 그대로 화면 만들기
- 내 **캘린더·Notion·Slack**에 접근해 개인 비서처럼 굴기

### 역사 이야기: MCP는 어쩌다 '표준'이 됐나

MCP가 정말 흥미로운 건 **경쟁사들이 같은 규격에 손을 얹었다**는 점입니다. 앞서 본 'M×N 통합 지옥'을 풀 공용 규격이 필요했고, Anthropic이 2024년 11월 공개했는데, 이듬해 **OpenAI(2025년 3월)와 구글(2025년 4월)이 잇따라 채택**했습니다. 그리고 2025년 말에는 여러 회사가 함께 관리하는 중립 재단(리눅스 재단 산하 Agentic AI Foundation)으로 넘어갔습니다. 경쟁하는 회사들이 같은 규격을 함께 떠받치는 건 흔치 않은 일입니다. (출처: [Anthropic, "Introducing the Model Context Protocol," 2024](https://www.anthropic.com/news/model-context-protocol); [TechCrunch, "OpenAI adopts rival Anthropic's standard for connecting AI models to data," 2025](https://techcrunch.com/2025/03/26/openai-adopts-rival-anthropics-standard-for-connecting-ai-models-to-data/))

그래서 오늘 배우는 MCP는 특정 회사 기능이 아니라 **업계 공용 약속**입니다 - 한 번 익히면 도구가 바뀌어도 계속 쓸 수 있다는 뜻이죠.

### MCP로 실제로 뭘 할까 - 사례 모음

MCP가 추상적으로 느껴질 수 있으니, 사람들이 실제로 연결해 쓰는 예를 봅시다.

- **디자인 → 코드**: Figma를 MCP로 연결하면, AI가 디자인을 **읽어서 그대로 화면**을 만듭니다.
- **내 문서와 대화**: Notion을 연결해 "이번 주 할 일 정리해 줘", "이 문서 요약해 줘"를 채팅 안에서.
- **웹을 데이터로**: 웹사이트를 훑어 **상품·가격·이미지**를 정리해 오기(스크래핑)를 자연어로.
- **글을 소리로**: 긴 글을 음성으로 바꾸기.
- **일정 자동화**: "다음 주 화요일 회의 잡아 줘" → 캘린더에 초대장.
- **사내 업무**: 회사 DB·내부 문서·ERP에 붙여 IT 지원·채용·계약 업무를 돕기.

Anthropic이 Google Drive·Slack·GitHub·Postgres 같은 **기성 MCP 서버**를 공개해 뒀고, 개발 도구들(Cursor·Zed·Replit 등)도 MCP를 받아들였습니다.

흥미로운 현실 - 공개적으로 널리 쓰이는 MCP 서버는 아직 손에 꼽지만, **회사 내부용** MCP(사내 DB·문서 연결)가 가장 값지다는 평가입니다. 남에게 공개하지 않아도, **내 데이터에 AI를 붙이는 것** 만으로 큰 힘이 되니까요. (출처: [Merge, "5 real-world Model Context Protocol integration examples"](https://www.merge.dev/blog/mcp-integration-examples); [Pragmatic Engineer, "Building MCP servers in the real world"](https://newsletter.pragmaticengineer.com/p/mcp-deepdive))

### 실전 MCP - 사람들이 실제로 많이 붙이는 것들

MCP 서버는 이제 '앱스토어'처럼 한곳(공식 레지스트리)에서 찾습니다. 여러분이 "내 작업엔 뭘 붙일까" 감을 잡도록 대표적인 것만 추렸습니다.

| MCP 서버 | 한 줄 용도 | 이런 사람에게 |
| :--- | :--- | :--- |
| **파일시스템** | 내 폴더의 파일을 읽고 쓰기 | 문서·자료 정리하는 누구나 |
| **GitHub** | 저장소 이슈·코드 읽고 커밋·검색 | 프로젝트를 GitHub에 올리는 사람 |
| **브라우저 자동화(Playwright)** | AI가 실제 웹브라우저를 열어 클릭·입력·캡처 반복 | 웹작업·테스트 |
| **Slack** | 채널 내역 검색·메시지 보내기 | 팀 소통 자동화 |
| **Notion** | 노션 페이지·DB 읽고 쓰기 | 노션으로 지식 관리하는 사람 |
| **데이터베이스** | DB에 자연어로 물어 조회(읽기 전용 권장) | 앞서 배운 내 서비스 DB와 연결 |
| **구글 드라이브·캘린더** | 문서 검색·일정 확인/생성 | 문서·일정 자동화 |

(출처: 공식 MCP 레지스트리 [https://registry.modelcontextprotocol.io](https://registry.modelcontextprotocol.io))

딱 하나 조심 - 공개된 MCP 서버는 수천 개 이상으로 급증했고 그중엔 검증 안 된 것도 많습니다. **모르는 서버를 함부로 붙이지 말고, 회사가 직접 낸 공식 서버 위주로** 시작하세요. AI에게 내 데이터·계정 접근 권한을 주는 일이니까요.

### 개념 정리: MCP가 건네는 세 가지 - 도구·자원·프롬프트

MCP로 연결하면 AI가 세 종류를 쓸 수 있게 됩니다. 이름만 알아 두면 됩니다.

- **도구(Tools)** - AI가 **실행**하는 기능 (예: 검색하기, DB 조회하기)
- **자원(Resources)** - AI가 **읽어 들이는** 데이터 (예: 파일, 문서, 기록)
- **프롬프트(Prompts)** - 미리 만들어 둔 **요청 템플릿**

이게 왜 좋을까요? ① 연결을 매번 새로 안 만들어도 되고, ② AI가 **실제 최신 데이터**를 보니 **엉뚱한 답(환각)이 줄고**, ③ 학습 시점 이후의 최신 정보도 쓸 수 있습니다. (출처: [modelcontextprotocol.io](https://modelcontextprotocol.io/docs/getting-started/intro); [Descope, "What Is the Model Context Protocol (MCP) and How It Works"](https://www.descope.com/learn/post/mcp))

MCP는 결국 이렇게 다가옵니다 - "AI야, **우리 서비스 DB** 보고 답해 줘"가 가능해지는 것. 문법을 몰라도, 이미 만들어진 MCP 연결을 **켜기만** 하면 됩니다.

![[day09-mcp-structure-diagram.png]]
*출처: 모두의연구소 AIFFEL_LMS 강의자료. MCP 구조 인포그래픽: Host의 MCP Client가 MCP Server와 표준 방식으로 대화하고, 서버가 세 종류의 기능(도구·자원·프롬프트)을 제공합니다.*

### MCP의 약속: 연결 하나를 다시 만들지 않기

MCP에서 **host**는 AI를 담은 앱(예: 코딩 도구)이고, 그 안의 **client**가 연결을 관리합니다. **server**는 Notion·파일 시스템·DB 같은 바깥 기능을 MCP 방식으로 내놓습니다. server가 제공하는 것은 실행 가능한 **tools**, 읽을 수 있는 **resources**, 재사용할 **prompts**입니다. 한 서버가 무엇을 허용하는지는 서버가 명시하고, host는 그 목록을 AI에게 보여 줍니다.

표준이 없으면 AI 앱 M개와 데이터 서비스 N개가 각각 전용 연동을 만들어야 하므로 연결이 **M×N개**가 됩니다. 공통 규약을 따르면 앱은 MCP client를 한 번 구현하고, 서비스는 MCP server를 한 번 구현해 조합할 수 있어 대략 **M+N개**의 구현으로 줄어듭니다. USB-C 비유가 여기서 나옵니다. 플러그 모양이 같다고 모든 기기가 같은 권한을 갖는 것은 아닌 것처럼, MCP도 연결 규칙이지 자동 안전 보장은 아닙니다.

특히 tool은 실제 행동을 할 수 있습니다. 처음에는 "읽기" 도구부터 연결하고, 쓰기·삭제·배포처럼 되돌리기 어려운 권한은 승인 과정을 둡니다. AI가 최신 데이터를 읽는다고 해서 해석까지 자동으로 맞는 것은 아니므로, 중요한 결과는 원본과 함께 확인합니다.

---

## 전체 커리큘럼 구성표

|  강좌 번호   | 강좌명                                     |
| :------: | :--------------------------------------- |
|  **1강**  | [[LMS04-01 어시스턴트에서 에이전트로 - 도구의 정체]]    |
|  **2강**  | **2강. MCP - AI를 내 데이터에 꽂는 USB-C (현재)**  |
|  **3강**  | [[LMS04-03 스킬 - 반복 작업을 AI에게 외워 두기]]     |
|  **4강**  | [[LMS04-04 하네스 종류 - 어떤 AI 코딩 도구를 고를까]]  |
|  **5강**  | [[LMS04-05 스킬 너머 - 확장과 요즘 흐름]]          |
|  **6강**  | [[LMS04-06 실습 - 어제 만든 Todo 앱을 'AI가 아는' 앱으로]] |
|  **7강**  | [[LMS04-07 내 워크플로 강화하기와 과제]]            |
|  **8강**  | [[LMS04-08 프로젝트 제출]]                    |
