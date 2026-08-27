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

---

## Memory(메모리)의 역할 및 의미

Claude Code의 **memory(메모리)** 는 세션마다 새 컨텍스트 윈도우로 시작하는 Claude가 **세션을 넘어 지식을 이어가도록 해주는 장치**입니다.

> 각 Claude Code 세션은 **매번 빈 컨텍스트 윈도우**로 시작합니다. 즉 이전 대화를 기억하지 못합니다.
> memory는 이 단절을 메워, "매번 다시 설명해야 하는 일"을 없애줍니다.

### 두 가지 메모리 시스템

memory는 크게 **두 가지 상호 보완적인 시스템**으로 나뉩니다. 둘 다 **매 대화 시작 시 자동 로드**되며, Claude는 이를 강제 설정이 아니라 **참고 컨텍스트**로 취급합니다.

| 구분 | **CLAUDE.md** | **자동 메모리(Auto Memory)** |
| --- | --- | --- |
| **작성자** | 👤 사용자(사람)가 작성 | 🤖 Claude가 스스로 작성 |
| **내용** | 지침·규칙 (해야 할 것) | 학습·패턴 (알아낸 것) |
| **성격** | "이렇게 일해라"는 **지시** | "이 프로젝트는 이렇더라"는 **메모** |
| **예시** | 코딩 표준, 빌드 명령, 아키텍처 규칙 | 빌드 명령 발견, 디버깅 인사이트, 선호도 |
| **저장 위치** | `CLAUDE.md` 파일 (git 공유 가능) | `~/.claude/projects/<project>/memory/` (로컬) |
| **로드 시점** | 모든 세션 시작 시 **전체** 로드 | 모든 세션 시작 시 `MEMORY.md`의 앞 200줄/25KB 로드 |

### 1) CLAUDE.md — 사람이 주는 "지속 지침"

Claude에게 매 세션 지속적으로 적용할 규칙을 담는 마크다운 파일입니다. **범위(scope)별로 계층**을 가지며, 넓은 범위 → 좁은 범위 순으로 로드됩니다.

| 범위 | 위치 | 용도 |
| --- | --- | --- |
| **관리 정책(조직)** | `/etc/claude-code/CLAUDE.md` 등 | 회사 표준·보안 정책 (개별 제외 불가) |
| **사용자** | `~/.claude/CLAUDE.md` | 모든 프로젝트 공통 개인 선호 |
| **프로젝트** | `./CLAUDE.md` 또는 `./.claude/CLAUDE.md` | 팀 공유 (git 커밋) |
| **로컬** | `./CLAUDE.local.md` | 개인용, `.gitignore` 대상 |

**추가 팁**

- `/init` 명령으로 코드베이스를 분석한 시작용 CLAUDE.md를 자동 생성
- `@경로/파일` 구문으로 다른 파일(README, package.json 등)을 **import** 가능 (최대 4홉)
- **200줄 이하 · 구체적 · 구조화**가 준수율을 높이는 핵심 (예: "코드를 잘 포맷" ❌ → "2칸 들여쓰기 사용" ✅)
- 대규모 프로젝트는 `.claude/rules/` 로 주제별 분리, `paths:` frontmatter로 **특정 파일에만 적용되는 규칙** 지정 가능
- `AGENTS.md`는 직접 읽지 않으므로, 필요하면 CLAUDE.md에서 `@AGENTS.md`로 가져오거나 심볼릭 링크

### 2) 자동 메모리 — Claude가 스스로 쌓는 "학습 노트"

사용자가 아무것도 안 해도 Claude가 작업 중 알아낸 것(빌드 명령, 디버깅 인사이트, 코드 스타일 선호 등)을 **스스로 노트로 저장**합니다.

- 저장소: `~/.claude/projects/<project>/memory/` — `MEMORY.md`(인덱스) + 주제별 파일(`debugging.md` 등)
- `MEMORY.md`의 앞 **200줄/25KB만** 세션 시작 시 로드, 나머지 주제 파일은 필요할 때 읽음
- **컴퓨터 로컬** 저장 — 같은 git 저장소의 worktree/하위 디렉토리는 공유하지만, 다른 컴퓨터·클라우드로는 공유되지 않음
- 기본 활성화. `/memory` 토글 또는 `autoMemoryEnabled: false`(또는 `CLAUDE_CODE_DISABLE_AUTO_MEMORY=1`)로 끌 수 있음
- UI에 **"Writing memory" / "Recalled memory"** 표시로 동작 확인
- 서브에이전트도 자체 자동 메모리 유지 가능

### memory의 한계 (중요)

> CLAUDE.md 내용은 시스템 프롬프트가 아니라 **시스템 프롬프트 뒤의 사용자 메시지**로 전달됩니다.
> 따라서 Claude가 읽고 따르려 하지만 **엄격한 강제는 아닙니다.**

- **반드시 강제**해야 하는 규칙(커밋 전 린트 실행 등)은 memory가 아니라 **Hooks**로 구현
- 도구/명령/경로를 하드 차단하려면 **관리 설정(`permissions.deny`)** 사용
- `/compact`(압축) 후에도 **프로젝트 루트 CLAUDE.md는 다시 로드**되어 생존 (단, 대화로만 준 지침은 사라짐)

### 관리 명령어

- **`/memory`** — 현재 로드된 CLAUDE.md·규칙 파일 목록 확인, 자동 메모리 폴더 열기/편집, 자동 메모리 토글
- **`/init`** — 프로젝트 CLAUDE.md 생성
- Claude에게 "이건 CLAUDE.md에 추가해줘" 또는 "~를 기억해줘"라고 말하면 각각 CLAUDE.md/자동 메모리에 저장

### 한 줄 요약

> **memory = "세션이 바뀌어도 유지되는 지식"**.
> **CLAUDE.md**는 *사람이 주는 규칙*, **자동 메모리**는 *Claude가 스스로 쌓는 학습 노트*.
> 둘 다 매 세션 자동 로드되지만 강제가 아닌 컨텍스트이므로, 꼭 강제할 규칙은 Hooks/설정으로 처리합니다.

### 참고 링크

- [Claude가 프로젝트를 기억하는 방법 — 공식 문서(한국어)](https://code.claude.com/docs/ko/memory)

---

## Claude Agent SDK

**Claude Agent SDK**(구 *Claude Code SDK*)는 **Claude Code를 라이브러리로 사용해 직접 AI 에이전트를 만들 수 있게 해주는 개발 키트**입니다. 터미널에서 사람이 쓰는 Claude Code가 아니라, **여러분의 코드(프로그램) 안에서 Claude Code의 에이전트 엔진을 돌리는 것**이 핵심입니다.

> Claude Code를 움직이는 것과 **동일한 도구·에이전트 루프(agent loop)·컨텍스트 관리**를 Python/TypeScript로 프로그래밍할 수 있게 노출한 라이브러리.

여기서 **에이전트(agent)** 란 스스로 단계를 계획하고, 파일을 읽고, 명령을 실행하고, 코드를 수정하는 **도구(tool)를 호출하며** 작업을 완수하는 애플리케이션을 뜻합니다. 이 "도구 호출 루프"를 직접 구현하지 않아도 SDK가 대신 돌려줍니다.

### 다른 Claude 도구와의 차이

| 목적 | 사용할 것 | 이유 |
| --- | --- | --- |
| 도구 루프를 직접 구현하지 않고 **에이전트를 만들고 싶다** | **Agent SDK** | 내 프로세스 안에서 에이전트 루프를 돌리는 라이브러리 (Python/TS) |
| 터미널에서 대화형 개발·일회성 작업 | **Claude Code CLI** | 일상적 대화형 사용에 최적 |
| API를 직접 호출하고 **도구 루프를 직접 구현** | **Client SDK** (Anthropic API) | Claude Code가 아닌 API 직접 접근 |
| 샌드박스·세션 인프라 없이 **장기/비동기 에이전트** 운영 | **Managed Agents** | Anthropic이 에이전트·샌드박스를 호스팅하는 별도 제품 |

> 💡 SDK는 **Python·TypeScript 전용**입니다. 다른 언어에서 같은 에이전트 루프를 쓰려면 CLI를 `-p` + `--output-format json`으로 **서브프로세스로 실행**(headless)하면 됩니다.

### 제공하는 기능 (Claude Code의 강점을 그대로)

| 기능 | 설명 |
| --- | --- |
| **내장 도구(Built-in tools)** | 파일 읽기/쓰기/수정, 명령 실행, 웹 검색 |
| **Hooks** | 에이전트 생명주기의 특정 지점에서 커스텀 코드 실행 |
| **Subagents** | 집중된 하위 작업을 위한 전문 에이전트 생성 |
| **MCP** | Model Context Protocol로 외부 도구·데이터 소스 연결 |
| **Permissions** | 어떤 도구를 자동 실행/승인 필요로 할지 제어 |
| **Sessions** | 교환 간 컨텍스트 유지, 재개(resume)·분기(fork) 가능 |
| **Skills / Commands / Memory** | 프로젝트의 `.claude/`, `~/.claude/`에서 Claude Code와 동일하게 자동 로드 |
| **Plugins** | skills·agents·hooks·MCP 서버를 묶어 로컬 경로로 로드 |

### 시작 방법

```bash
# Python
pip install claude-agent-sdk

# TypeScript
npm install @anthropic-ai/claude-agent-sdk
```

- **인증**: API 키 방식 사용 (Anthropic Console에서 발급). Agent SDK로 만든 제품에는 **claude.ai 로그인/요율 사용이 사전 승인 없이는 허용되지 않음** — 반드시 API 키 인증 사용
- 핵심 진입점은 **`query`** (프롬프트를 주면 에이전트 루프가 돌며 도구를 호출하고 결과 메시지를 스트리밍)

```python
import anyio
from claude_agent_sdk import query

async def main():
    async for message in query(prompt="이 저장소의 버그를 찾아 고쳐줘"):
        print(message)

anyio.run(main)
```

### 활용 사례

- **자동 버그 수정 에이전트** — 코드베이스를 탐색해 버그를 찾고 고침 (공식 Quickstart 예제)
- **코드 리뷰/리팩토링 봇**, CI 파이프라인에 통합된 에이전트
- **SRE·운영 자동화**, 데이터 파이프라인 에이전트
- **서브에이전트 오케스트레이션** — 여러 전문 에이전트를 조율하는 복잡한 워크플로우("하네스 엔지니어링")

### 브랜딩 주의사항 (파트너용)

SDK로 만든 제품에 Claude 브랜딩은 선택 사항이며, **"Claude Agent" / "Powered by Claude"는 허용**되지만 **"Claude Code" / "Claude Code Agent" 표기는 금지**입니다. 제품은 자체 브랜드를 유지해야 합니다.

### 핵심 정리

> **Claude Agent SDK = "Claude Code를 라이브러리로"**.
> CLI가 사람이 터미널에서 쓰는 도구라면, SDK는 **내 프로그램이 Claude Code 엔진(도구·에이전트 루프·컨텍스트·MCP·서브에이전트·Hooks)을 직접 호출**해 커스텀 AI 에이전트를 만드는 방식입니다. Python·TypeScript를 지원하며, API 키로 인증합니다.

### 참고 링크

- [Agent SDK 개요 — 공식 문서](https://code.claude.com/docs/en/agent-sdk/overview)
- [Quickstart](https://code.claude.com/docs/en/agent-sdk/quickstart)
- [TypeScript SDK (GitHub)](https://github.com/anthropics/claude-agent-sdk-typescript) · [Python SDK (GitHub)](https://github.com/anthropics/claude-agent-sdk-python)
- [예제 에이전트 모음 (GitHub)](https://github.com/anthropics/claude-agent-sdk-demos)
