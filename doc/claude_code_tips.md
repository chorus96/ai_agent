# Claude Code 팁 정리

> Claude Code를 사용하면서 알아두면 좋은 개념·팁을 정리한 문서입니다.
> 최종 정리일: 2026-08-27

---

## Plugin vs MCP 비교

Claude Code에서 **Plugin**과 **MCP**는 자주 혼동되지만, 사실 **서로 다른 층위(layer)** 의 개념입니다. 한쪽은 "포장/배포 방식"이고, 다른 한쪽은 "외부 연결 방식"입니다.

### 한 줄 요약

- **MCP (Model Context Protocol)**: Claude를 **외부 도구·데이터·API에 연결하는 표준 프로토콜** — *무엇에 연결할 것인가*
- **Plugin**: 슬래시 커맨드·서브에이전트·Hooks·Skills·**MCP 서버 설정까지** 하나로 묶어 배포하는 **최상위 패키지** — *어떻게 묶어서 배포할 것인가*

즉 **MCP는 Plugin 안에 담길 수 있는 구성 요소 중 하나**입니다. 둘은 경쟁 관계가 아니라 포함 관계에 가깝습니다.

### 상세 비교

| 항목 | MCP | Plugin |
| --- | --- | --- |
| **본질** | 외부 시스템 연결 **프로토콜(표준)** | 확장 기능 **배포 패키지(번들)** |
| **주 목적** | Claude에 새로운 **도구/데이터 소스** 제공 (DB, GitHub, Slack, 파일시스템 등) | 여러 확장 요소를 한 번에 **설치/공유/버전관리** |
| **담는 내용** | 도구(tools), 리소스(resources), 프롬프트 | 슬래시 커맨드 + 서브에이전트 + Hooks + Skills + **MCP 서버** + 설정 |
| **실행 형태** | 별도 프로세스로 도는 **MCP 서버**(stdio/HTTP/SSE) | 설치되는 파일·설정의 묶음 (자체 프로세스 아님) |
| **연결 대상** | 주로 **외부** 서비스/시스템 | 주로 **Claude Code 내부** 동작 확장 |
| **설정 위치** | `.mcp.json`, `claude mcp add`, 사용자/프로젝트 설정 | 플러그인 마켓플레이스 / `/plugin` 로 설치 |
| **범위** | 단일 기능(연결) | 여러 기능을 아우르는 상위 개념 |
| **비유** | "외부 협력사와 통신하는 규격" | "여러 도구를 담은 설치형 앱 패키지" |

### 관계도

```
Plugin (최상위 패키지)
├── Slash Commands   (/명령)
├── Subagents        (서브에이전트)
├── Hooks            (이벤트 자동화)
├── Skills           (기술 매뉴얼)
├── MCP 서버 설정  ← MCP가 여기에 포함될 수 있음
└── 기타 설정
```

### 언제 무엇을 쓰나

- **"Claude가 우리 회사 DB / Jira / 사내 API에 접근하게 하고 싶다"** → **MCP** (해당 시스템의 MCP 서버 연결)
- **"슬래시 커맨드 + 코드리뷰 서브에이전트 + 린트 Hook + 필요한 MCP 연결을 팀 전체에 한 번에 배포하고 싶다"** → **Plugin** 으로 묶어서 배포
- 실무에서는 보통 **Plugin으로 배포하면서, 그 안에 필요한 MCP 서버 설정을 포함**시키는 형태가 가장 일반적입니다.

### 핵심 정리

> MCP는 **"연결(what to connect to)"**, Plugin은 **"배포(how to package & distribute)"**.
> MCP는 Plugin의 구성 요소가 될 수 있으므로, 둘은 대체재가 아니라 **함께 쓰이는 상하위 관계**입니다.

### 참고 링크

- [MCP를 통해 Claude Code를 도구에 연결하기 — 공식 문서(한국어)](https://code.claude.com/docs/ko/mcp)
- [Claude code — plugin, commands, skills, agents, hooks 에 대하여 — Velog](https://velog.io/@ghdtjrrl94/Claude-code-plugin-commands-skills-agents-hooks-%EC%97%90-%EB%8C%80%ED%95%98%EC%97%AC)
