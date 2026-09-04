# [AI Agent 파헤치기] - 7. 멀티에이전트 — 한 Gateway에 여러 역할

---

## 멀티에이전트 — 한 Gateway에 여러 역할

지금까지는 에이전트 하나를 다뤘습니다. 실무에서는 역할이 갈립니다. 문의를 받는 역할, 코드를 고치는 역할, 자료를 조사하는 역할이 서로 다른 시간에 다른 일을 합니다. OpenClaw는 이 차이를 에이전트를 여러 개 띄우는 것으로 풉니다. Gateway 프로세스 하나 안에 서로 격리된 에이전트 여럿을 두고, 들어온 대화를 담당 역할에게 나릅니다. ([docs/concepts/multi-agent](https://docs.openclaw.ai/concepts/multi-agent))

이 절에서는 설치 없이 공식 문서의 설정 예시를 읽으며 이 구조를 이해합니다. 눈여겨 볼 그림은 하나입니다. 채널에서 메시지가 들어오면 binding이 받을 에이전트를 고르고, 고른 에이전트는 자기 작업 폴더와 자기 기록 안에서만 움직입니다. 챗봇을 여러 개 만드는 일이 아니라, 대화를 역할에게 보내는 작은 라우터를 설계하는 일입니다.

### binding은 채널 계정을 역할에 연결합니다

binding(바인딩)은 채널 계정을 특정 에이전트에 연결하는 규칙입니다. 채널 계정은 슬랙 작업공간 하나, 와츠앱 번호 하나 같이 채널이 구분하는 단위입니다. 예를 들어 디스코드의 support 계정에서 온 메시지는 항상 support 에이전트로 가게 만드는 규칙이 binding입니다. ([docs/concepts/agent-bindings](https://docs.openclaw.ai/concepts/agent-bindings))

공식 문서의 설정 예시를 함께 읽습니다. 에이전트 둘과 binding 하나를 정의한 가장 짧은 형태입니다.

```json5
{
  agents: {
    entries: {
      main: {
        default: true,
        workspace: "~/.openclaw/workspace",
      },
      support: {
        workspace: "~/.openclaw/workspace-support",
      },
    },
  },
  bindings: [
    {
      channel: "discord",
      account: "support",
      agent: "support",
    },
  ],
}
```

main에는 `default: true`가 붙었습니다. 어떤 binding에도 해당하지 않는 대화가 도착하면 기본 에이전트가 받습니다. 기본 에이전트를 정하지 않으면, 갈 곳이 없어진 메시지가 조용히 사라지는 사고가 날 수 있습니다. 여러 역할을 두기로 했다면 어느 에이전트가 남은 대화를 받을지 반드시 정해 둡니다. ([docs/concepts/agent-bindings](https://docs.openclaw.ai/concepts/agent-bindings))

binding이 하는 일은 메시지를 받을 에이전트를 고르는 것까지입니다. 앞 절에서 본 페어링 승인이나 허용 목록처럼 누가 말을 걸 수 있는지를 정하는 규칙은 채널 설정이 따로 가집니다. 라우터가 갈 길을 정할 뿐, 문지방의 기준을 대신 정하지 않습니다.

### 에이전트마다 분리되는 것

에이전트 하나가 감싸는 범위를 페르소나 하나의 전체라고 부릅니다. 역할의 성격, 쓰는 규칙, 작업 폴더, 대화 기록이 한 단위로 묶입니다. 공식 문서는 에이전트마다 분리되는 것으로 작업 폴더, 상태 폴더, 대화 기록 저장소를 적습니다. ([docs/concepts/multi-agent](https://docs.openclaw.ai/concepts/multi-agent))

- workspace(작업 폴더)는 에이전트가 기본으로 일하는 폴더입니다. main과 support가 각자 다른 폴더를 쓰므로 만지는 파일이 처음부터 갈립니다.
- 상태 폴더는 인증 프로필과 에이전트별 설정이 사는 곳입니다. 대화 기록은 에이전트마다 따로인 SQLite 파일로 남습니다. 기본 위치는 `~/.openclaw/agents/<agentId>/agent/openclaw-agent.sqlite`입니다.
- 기억과 기록은 기본으로 섞이지 않습니다. 한 에이전트의 기본 메모리는 다른 에이전트의 기록을 검색하지 않습니다. 함께 써야 하는 자료는 두 에이전트가 모두 읽는 공유 문서를 명시적으로 둡니다.

서로 다른 두 파이썬 프로젝트가 폴더를 나눠 쓰는 것과 같은 이치입니다. 연구 담당과 코딩 담당이 한 테이블에서 일하면 서류가 섞입니다. 폴더를 나누고 필요한 문서만 복사해 두는 규칙이 있어야 서로의 기록을 실수로 건드리지 않습니다.

주의할 경계도 문서는 분명히 적습니다. 상태 폴더를 에이전트끼리 재사용하면 인증과 세션 상태가 충돌합니다. 작업 폴더는 기본 작업 경로일 뿐 단단한 격리 장치가 아니므로, 폴더 밖의 경로를 막아야 한다면 sandbox를 따로 두어야 합니다. ([docs/concepts/agent-workspace](https://docs.openclaw.ai/concepts/agent-workspace))

### delegate는 자기 계정으로 대신 행동합니다

역할이 늘어나면 사람을 대신해 일하는 에이전트가 필요해집니다. delegate(**델리게이트**)는 조직 안에서 자기 이메일 주소와 표시 이름을 가지고 일하는 에이전트입니다. 공식 문서는 정체성을 한 문장으로 못 박습니다. 사람으로 가장하지 않고, 자기 자격 증명으로, 정해진 권한 안에서 대신 행동합니다. 내 계정을 빌려주는 자동화가 아니라, 권한 범위를 정해 둔 보조 구성원을 추가하는 설계입니다.

권한은 작은 것에서 시작합니다. 문서가 권하는 첫 단계는 읽고 초안만 만들고 사람의 검토를 받는 등급입니다. 아무것도 승인 없이는 밖으로 나가지 않습니다. 발송과 일정 생성처럼 밖으로 나가는 행동은 다음 등급에서 delegate 자신의 이름으로 이뤄집니다. 넓힐 때는 도구 제한, sandbox, 감사 기록을 먼저 갖추는 순서를 지킵니다. ([docs/concepts/delegate-architecture](https://docs.openclaw.ai/concepts/delegate-architecture))

delegate 문서가 다루는 권한은 조직의 이메일과 캘린더입니다. 넓은 위임 열쇠 하나가 새면 모든 사서함과 캘린더가 열리는 구조라 공식 문서 스스로 최소 권한을 권합니다. 수업에서는 실제 메일 권한을 연결하지 않고, 역할과 승인 규칙을 설계하는 것까지만 다룹니다.

### standing orders와 전문가 레인

역할이 여럿이면 맡긴 일을 언제 어떻게 하는지 규칙이 필요합니다. standing orders(**스탠딩 오더**)는 항상 적용되는 지시입니다. 무엇을 해도 되는지, 무엇은 승인을 받아야 하는지, 무엇이 보이면 멈추는지를 에이전트에게 미리 적어 두는 규칙입니다. 허용, 승인, 중단의 세 칸을 채우는 셈이고, 앞 절에서 만든 권한 기준 메모와 같은 문양입니다. ([docs/automation/standing-orders](https://docs.openclaw.ai/automation/standing-orders))

병렬 작업은 에이전트를 많이 만드는 일이 아니라 역할 설계가 먼저입니다. 공식 문서의 병렬 전문가 레인은 연구, 제작, 검토처럼 역할이 겹치지 않게 나누고, 동시에 돌릴 개수를 정해 두는 구성을 권합니다. 아래는 예시 설정의 해당 부분입니다.

```json5
{
  agents: {
    defaults: {
      maxConcurrent: 4,
      subagents: { maxConcurrent: 8, delegationMode: "prefer" },
    },
  },
  messages: {
    queue: { mode: "collect", cap: 20, drop: "summarize" },
  },
}
```

동시 실행 수에 한도를 두고, 몰리는 메시지는 모아 두었다가 요약해 넘기는 규칙까지가 설계 대상입니다. 에이전트를 늘리는 일보다 역할 겹침과 용량을 먼저 정하는 편이 사고를 줄입니다. ([docs/concepts/parallel-specialist-lanes](https://docs.openclaw.ai/concepts/parallel-specialist-lanes))

에이전트끼리 직접 대화하는 길은 기본으로 닫혀 있습니다. 켜려면 명시적 활성화와 허용 목록이 필요합니다. 필요한 상대만, 필요한 범위만 지정하는 설계입니다.

```json5
{
  tools: {
    agentToAgent: {
      enabled: false,
      allow: ["home", "work"],
    },
  },
}
```

명령줄에서 특정 에이전트에게 단일 실행을 시키는 `openclaw agent`도 별개의 도구입니다. `--agent`, `--session-id`처럼 대상을 지정해 한 번의 실행을 요청할 뿐, 에이전트 사이의 대화를 여는 장치가 아닙니다. 역할 간 전달은 서로의 대화를 모두 보는 것이 아니라, 누구에게 무엇을 넘길지 명시하는 작업입니다. ([docs/concepts/multi-agent](https://docs.openclaw.ai/concepts/multi-agent), [docs/tools/agent-send](https://docs.openclaw.ai/tools/agent-send))

### 역할이 늘면 누가 얼마나 쓰는지가 궁금해집니다

여기까지 읽은 구조를 한 줄로 정리합니다. Gateway가 문을 지키고, binding이 대화를 역할에게 나르고, 각 에이전트는 자기 폴더와 자기 기록 안에서 정해진 권한만 움직입니다. 역할이 이렇게 나뉘면 다음 질문이 자연스럽게 따라옵니다. 역할마다 토큰을 얼마나 쓰는지, 비용이 어떻게 쌓이는지, 지금 연결이 살아 있는지를 어디서 보는지입니다. 다음 절에서 이 질문을 직접 확인합니다.

---

## 전체 커리큘럼 구성표

|  강좌 번호   | 강좌명                                     |
| :------: | :--------------------------------------- |
|  **1강**  | [[AA04-01 OpenClaw는 무엇이고 어디서 왔나]]           |
|  **2강**  | [[AA04-02 OpenClaw 나도 써보자]]           |
|  **3강**  | [[AA04-03 세션 기록 읽기]]           |
|  **4강**  | [[AA04-04 텔레그램으로 부리기]]           |
|  **5강**  | [[AA04-05 보안 - OpenClaw가 지키는 것들]]           |
|  **6강**  | [[AA04-06 pi - 내 에이전트 툴의 뼈대]]           |
|  **7강**  | **7강. 멀티에이전트 — 한 Gateway에 여러 역할 (현재)**           |
|  **8강**  | [[AA04-08 모니터링 - 사용량과 상태 보기]]           |
|  **9강**  | [[AA04-09 자동화 - 반복되는 일을 예약하기]]           |
|  **10강**  | [[AA04-10 확장 - 손끝 늘리기, 기능 투어]]           |
