# Claude Code 팁 정리

> Claude Code를 사용하면서 알아두면 좋은 개념·팁을 정리한 문서입니다.
> 최종 정리일: 2026-08-27

## 목차

1. [Plugin vs MCP 비교](#plugin-vs-mcp-비교)
2. [Memory(메모리)의 역할 및 의미](#memory메모리의-역할-및-의미)
3. [Claude Agent SDK](#claude-agent-sdk) — 개념·기능·인증(사내 LLM 포함)·오픈소스 여부
4. [Subagents(서브에이전트) 활용법](#subagents서브에이전트-활용법)
   - [Subagents vs Agent Teams](#subagents-vs-agent-teams)
5. [Hooks(훅) 실전 활용](#hooks훅-실전-활용)
6. [Skills 만들기](#skills-만들기)
7. [권한 모드 (Permission Modes)](#권한-모드-permission-modes)
8. [Plan Mode (계획 모드)](#plan-mode-계획-모드)
9. [Slash Commands (슬래시 명령어)](#slash-commands-슬래시-명령어)
10. [Slash Commands vs Skill 비교](#slash-commands-vs-skill-비교)
11. [Checkpoints & Rewind (체크포인트·되감기)](#checkpoints--rewind-체크포인트되감기)

> 📚 관련 문서: 한글 자료·도서·GitHub 모음은 [`claude_code.md`](./claude_code.md), CLI 명령어·플래그·슬래시 명령어 레퍼런스는 [`claude_code_cli.md`](./claude_code_cli.md) 참고.

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

- **인증**: Console API 키뿐 아니라 **사내 LLM 게이트웨이·클라우드 프로바이더(Bedrock/Vertex/Foundry)** 도 지원 (아래 [인증·프로바이더](#인증프로바이더-사내-llm-포함) 참고)
- 핵심 진입점은 **`query`** (프롬프트를 주면 에이전트 루프가 돌며 도구를 호출하고 결과 메시지를 스트리밍)

```python
import anyio
from claude_agent_sdk import query

async def main():
    async for message in query(prompt="이 저장소의 버그를 찾아 고쳐줘"):
        print(message)

anyio.run(main)
```

### 인증·프로바이더 (사내 LLM 포함)

Agent SDK는 **"Claude Code를 라이브러리로"** 돌리는 것이라, **CLI가 지원하는 프로바이더 설정을 그대로 상속**합니다. 따라서 **Anthropic Console API 키만 되는 것이 아니라**, 사내 LLM 게이트웨이나 클라우드 프로바이더로도 동작합니다.

> **오해 주의**: Agent SDK에서 금지되는 것은 딱 하나 — **claude.ai *개인 구독 로그인*(Pro/Max OAuth)으로 만든 제품을 제3자에게 제공**하는 것(사전 승인 필요)입니다. 이는 **비즈니스/약관상 제한**이지 "Console API 키만 허용"이라는 기술적 제한이 아닙니다.

| 인증 방식 | 설정 환경변수 |
| --- | --- |
| **사내 LLM 게이트웨이** | `ANTHROPIC_BASE_URL` (+ 필요 시 `ANTHROPIC_AUTH_TOKEN`) |
| Anthropic Console | `ANTHROPIC_API_KEY` |
| Amazon Bedrock | `CLAUDE_CODE_USE_BEDROCK=1` + AWS 자격증명 |
| Google Cloud Agent Platform (Vertex) | `CLAUDE_CODE_USE_VERTEX=1` + GCP 자격증명 |
| Microsoft Foundry | Foundry 설정 + API 키/Entra ID |
| 프로바이더별 게이트웨이 경유 | `ANTHROPIC_BEDROCK_BASE_URL`, `ANTHROPIC_VERTEX_BASE_URL`, `ANTHROPIC_FOUNDRY_BASE_URL`, `ANTHROPIC_AWS_BASE_URL` |
| 회사 프록시 | `HTTPS_PROXY` / `HTTP_PROXY` |

**실무 팁 — 사내 LLM을 이미 CLI로 쓰고 있다면**

- CLI에서 **`/status`** 를 실행하면 현재 세션이 쓰는 **프로바이더·base URL·프록시**를 확인할 수 있습니다 (보통 `ANTHROPIC_BASE_URL`로 사내 게이트웨이를 가리키는 방식).
- Agent SDK는 **별도 키 발급 없이**, SDK를 실행하는 프로세스 환경에 **CLI가 쓰는 것과 동일한 환경변수**를 넘겨주면 그대로 사내 LLM으로 동작합니다.

```bash
# 사내 게이트웨이로 Agent SDK 실행 (SDK 프로세스가 상속하는 환경변수)
export ANTHROPIC_BASE_URL="https://llm-gateway.mycompany.com"
export ANTHROPIC_AUTH_TOKEN="<사내 토큰>"
python my_agent.py
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
> CLI가 사람이 터미널에서 쓰는 도구라면, SDK는 **내 프로그램이 Claude Code 엔진(도구·에이전트 루프·컨텍스트·MCP·서브에이전트·Hooks)을 직접 호출**해 커스텀 AI 에이전트를 만드는 방식입니다. Python·TypeScript를 지원하며, **Console API 키·사내 게이트웨이·Bedrock/Vertex/Foundry** 로 인증할 수 있습니다.

### 오픈소스 여부 · 라이선스

**Agent SDK 자체는 오픈소스입니다 — MIT 라이선스**로 GitHub에 공개되어 있습니다.

| 구성 요소 | 라이선스 / 공개 여부 |
| --- | --- |
| Python SDK (`claude-agent-sdk-python`) | **MIT License** (© 2025 Anthropic, PBC) |
| TypeScript SDK (`claude-agent-sdk-typescript`) | **MIT License** |

단, 두 가지를 구분해야 합니다.

- **SDK 코드(오픈소스) ≠ 모델(비공개)** — SDK 라이브러리 코드는 MIT지만, 뒤에서 추론하는 **Claude 모델 자체는 오픈소스가 아닙니다**. SDK는 API(또는 Bedrock/Vertex/사내 게이트웨이)를 호출하는 **클라이언트**일 뿐이며, 모델 가중치가 열려 있는 것은 아닙니다.
- **MIT + 상용 이용약관 병행** — 코드는 MIT지만, "이용(모델 호출·서비스 제공)"은 **Anthropic 상용 이용약관(Commercial Terms of Service)** 의 적용을 받습니다. 앞서의 "claude.ai 개인 구독 로그인으로 만든 제품의 제3자 제공 금지" 같은 제약이 여기서 나옵니다.

> **정리**: SDK 소스 코드 → MIT 오픈소스 ✅ (자유 사용·수정·상용화) / Claude 모델 → 비공개(오픈웨이트 아님) / 이용 조건 → MIT + Anthropic 상용 약관 병행

### 참고 링크

- [Agent SDK 개요 — 공식 문서](https://code.claude.com/docs/en/agent-sdk/overview)
- [Python SDK LICENSE (MIT)](https://github.com/anthropics/claude-agent-sdk-python/blob/main/LICENSE)
- [Quickstart](https://code.claude.com/docs/en/agent-sdk/quickstart)
- [TypeScript SDK (GitHub)](https://github.com/anthropics/claude-agent-sdk-typescript) · [Python SDK (GitHub)](https://github.com/anthropics/claude-agent-sdk-python)
- [예제 에이전트 모음 (GitHub)](https://github.com/anthropics/claude-agent-sdk-demos)
- [엔터프라이즈 배포 개요 (프로바이더·게이트웨이)](https://code.claude.com/docs/en/third-party-integrations) · [LLM 게이트웨이](https://code.claude.com/docs/en/llm-gateway)

---

## Subagents(서브에이전트) 활용법

**서브에이전트(subagent)** 는 **특정 유형의 작업을 처리하는 특화된 AI 어시스턴트**입니다. 각 서브에이전트는 **자체 컨텍스트 윈도우**에서 실행되며, 고유한 시스템 프롬프트·도구 접근·권한을 가지고, 작업을 마치면 **요약만 주 대화로 반환**합니다.

> 부작업이 검색 결과·로그·다시 참조하지 않을 파일 내용으로 **주 대화를 넘칠 때** 서브에이전트를 사용하세요.
> 서브에이전트가 그 작업을 자기 컨텍스트에서 처리하고 요약만 돌려줍니다.

### 왜 쓰는가 (핵심 장점)

- **컨텍스트 보존** — 탐색·구현의 장황한 출력을 주 대화에서 분리
- **제약 적용** — 서브에이전트가 쓸 수 있는 도구를 제한 (예: 읽기 전용)
- **구성 재사용** — 사용자 레벨 서브에이전트로 여러 프로젝트에서 공유
- **동작 특화** — 특정 도메인용 집중 시스템 프롬프트
- **비용 제어** — Haiku 같은 더 빠르고 저렴한 모델로 라우팅

### 내장 서브에이전트

Claude가 적절할 때 자동으로 사용하는 기본 제공 에이전트가 있습니다.

| 에이전트 | 도구 | 용도 |
| --- | --- | --- |
| **Explore** | 읽기 전용 (Write/Edit 거부) | 코드베이스 검색·분석 (빠르고 저렴) |
| **Plan** | 읽기 전용 | plan mode에서 계획 수립용 연구 |
| **general-purpose** | 모든 도구 | 탐색+수정이 필요한 복잡한 다단계 작업 |

> `Explore`/`Plan`은 속도를 위해 CLAUDE.md·git 상태를 건너뜁니다. 그 외 모든 서브에이전트는 둘 다 로드합니다.

### 만드는 방법

서브에이전트는 **YAML frontmatter + Markdown 시스템 프롬프트**로 이루어진 파일입니다. Claude에게 만들어 달라고 요청하거나 직접 작성할 수 있습니다. (v2.1.198부터 `/agents`의 대화형 생성 마법사는 제거됨 — Claude에 요청하거나 `.claude/agents/`를 직접 편집)

```markdown
---
name: code-reviewer
description: Reviews code for quality and best practices. Use proactively after code changes.
tools: Read, Glob, Grep
model: sonnet
---

You are a code reviewer. When invoked, analyze the code and provide
specific, actionable feedback on quality, security, and best practices.
```

- **본문(시스템 프롬프트)** 이 서브에이전트의 동작을 규정합니다. 서브에이전트는 전체 Claude Code 시스템 프롬프트가 아니라 **이 프롬프트만** 받습니다.
- 파일을 추가/편집하면 Claude Code가 몇 초 내 감지해 **재시작 없이** 다음 위임부터 반영 (단, 해당 `agents` 디렉토리가 세션 시작 시 없었다면 최초 1회 재시작 필요)

### 저장 위치와 범위 (우선순위 높은 순)

| 위치 | 범위 | 용도 |
| --- | --- | --- |
| 관리되는 설정 | 조직 전체 | 관리자가 배포 (최우선) |
| `--agents` CLI 플래그 | 현재 세션 | JSON으로 전달, 디스크 미저장 (테스트·자동화) |
| `.claude/agents/` | 현재 프로젝트 | 팀 공유 (git 커밋) |
| `~/.claude/agents/` | 모든 프로젝트 | 개인용 |
| 플러그인 `agents/` | 플러그인 활성 위치 | 플러그인으로 배포 (최저) |

> 같은 이름이 여러 곳에 있으면 우선순위가 높은 위치가 이깁니다. 트리 전체에서 `name`은 고유하게 유지하세요.

### 주요 frontmatter 필드

`name`과 `description`만 필수입니다.

| 필드 | 설명 |
| --- | --- |
| `name` | 소문자·하이픈 고유 식별자 |
| `description` | **언제 이 서브에이전트에 위임할지** (자동 위임 판단 근거) |
| `tools` | 사용 허용 도구 (허용 목록). 생략 시 전체 상속 |
| `disallowedTools` | 거부할 도구 (거부 목록). `tools`보다 먼저 적용 |
| `model` | `sonnet`/`opus`/`haiku`/`fable`/전체 ID/`inherit` (기본: `inherit`) |
| `permissionMode` | `default`/`acceptEdits`/`auto`/`dontAsk`/`bypassPermissions`/`plan` |
| `skills` | 시작 시 컨텍스트에 미리 로드할 Skills |
| `mcpServers` | 이 서브에이전트에서 쓸 MCP 서버 |
| `hooks` | 이 서브에이전트로 범위 지정된 라이프사이클 hooks |
| `memory` | 지속 메모리 범위 (`user`/`project`/`local`) — 교차 세션 학습 |
| `isolation` | `worktree` 지정 시 격리된 git worktree에서 실행 |
| `background` | `true`면 항상 백그라운드 실행 (v2.1.198부터 기본 백그라운드) |

> ⚠️ 보안상 **플러그인 서브에이전트**에서는 `hooks`·`mcpServers`·`permissionMode`가 무시됩니다.

### 호출 방법

**1) 자동 위임** — Claude가 작업 내용과 서브에이전트의 `description`을 보고 알아서 위임합니다. 적극 위임을 유도하려면 description에 **"use proactively"** 를 넣으세요.

**2) 명시적 호출** — 자동 위임이 부족할 때:

```text
# 자연어로 이름 지정 (Claude가 위임 결정)
Use the test-runner subagent to fix failing tests

# @-mention (특정 서브에이전트 실행 보장)
@agent-code-reviewer look at the auth changes
```

**3) 세션 전체를 서브에이전트로** — 주 스레드 자체가 해당 시스템 프롬프트·도구·모델을 사용:

```bash
claude --agent code-reviewer
```

프로젝트 기본값으로 만들려면 `.claude/settings.json`에 `{"agent": "code-reviewer"}`.

### 실전 활용 패턴

- **대량 출력 격리** — 테스트 실행·로그 처리 등 컨텍스트를 많이 먹는 작업을 위임하고 요약만 회수
  > `Use a subagent to run the test suite and report only the failing tests with their error messages`
- **병렬 연구** — 서로 독립적인 조사를 여러 서브에이전트로 동시 진행 후 Claude가 종합
  > `Research the authentication, database, and API modules in parallel using separate subagents`
- **서브에이전트 체인** — 다단계 워크플로우를 순차 위임
  > `Use the code-reviewer subagent to find performance issues, then use the optimizer subagent to fix them`
- **중첩 서브에이전트** — 서브에이전트가 다시 자신의 서브에이전트를 생성 (v2.1.172+), 중간 출력은 주 대화에 도달하지 않음

### 서브에이전트 vs 주 대화 — 언제 무엇을?

| 주 대화가 적합 | 서브에이전트가 적합 |
| --- | --- |
| 잦은 왕복·반복 개선이 필요 | 주 컨텍스트에 불필요한 대량 출력 발생 |
| 여러 단계가 컨텍스트를 공유(계획→구현→테스트) | 특정 도구 제한·권한을 적용하고 싶음 |
| 빠르고 대상이 명확한 변경 | 작업이 자체 완결적이고 요약 반환 가능 |
| 지연시간이 중요 (서브에이전트는 시동 시간 필요) | — |

> 주 대화 컨텍스트에서 도는 **재사용 프롬프트/워크플로우**가 필요하면 서브에이전트보다 **Skills**를, 지속적 병렬성이 필요하면 **agent teams**를 고려하세요.

### 모범 사례

- `description`을 **구체적으로** 작성 — 자동 위임 정확도를 좌우 (필요 시 "use proactively" 포함)
- **최소 권한** — 읽기 전용 작업이면 `tools: Read, Grep, Glob`로 제한
- **모델 라우팅** — 단순·대량 작업은 `haiku`, 정교한 분석은 `sonnet`/`opus`
- 팀 공유는 `.claude/agents/`에 커밋, 개인용은 `~/.claude/agents/`
- 하나의 서브에이전트는 **하나의 역할**에 집중시키기 (코드리뷰어, 디버거, 테스트러너 등)

### 참고 링크

- [사용자 정의 subagent 만들기 — 공식 문서(한국어)](https://code.claude.com/docs/ko/sub-agents)

---

## Subagents vs Agent Teams

둘 다 작업을 **병렬화**하지만 방식이 다릅니다. 핵심 갈림길은 **"워커들이 서로 통신해야 하는가?"** 입니다.

> - **Subagents** = **한 세션 안**에서 메인 에이전트가 부리는 워커. **결과만 메인에 보고**하고 서로 대화 안 함
> - **Agent Teams** = **각자 독립 세션**인 여러 Claude 인스턴스가 **서로 직접 통신**하며 공유 작업 목록으로 **자체 조율**

### 상세 비교

| 항목 | **Subagents** | **Agent Teams** |
| --- | --- | --- |
| **구조** | 단일 세션 내 워커 | 여러 독립 Claude Code 인스턴스 |
| **컨텍스트** | 자체 컨텍스트, 결과는 **호출자에 반환** | 자체 컨텍스트, **완전히 독립** |
| **통신** | **메인에게만 결과 보고** (워커끼리 X) | **팀원끼리 직접 메시지** 전송 |
| **조율** | 메인 에이전트가 모든 작업 관리 | **공유 작업 목록**으로 자체 조율 |
| **상호작용** | 사용자는 메인만 상대 | 리더를 거치지 않고 **개별 팀원과 직접 대화** 가능 |
| **토큰 비용** | 낮음 (결과가 메인에 요약됨) | **높음** (팀원마다 별도 인스턴스) |
| **최적 용도** | 결과만 중요한 집중 작업 | 논의·협업이 필요한 복잡 작업 |
| **상태** | 정식 기능 | **실험적** (기본 비활성화) |

### 구조 차이

```
[Subagents]                      [Agent Teams]
   메인 에이전트                      팀 리더
   ├─▶ subagent A ─┐               ├─ 팀원 A ⇄ 팀원 B
   ├─▶ subagent B ─┼─▶ 결과 요약     ├─ 팀원 B ⇄ 팀원 C   ← 서로 직접 통신
   └─▶ subagent C ─┘               └─ 공유 작업 목록 + 메일박스
   (워커끼리 대화 X)                  (자체 조율)
```

### 언제 무엇을?

**Subagents** — 빠르고 집중된 워커, 결과만 필요할 때
- "테스트 실행하고 실패만 요약", "코드베이스 병렬 검색 후 종합"
- 순차 작업, 같은 파일 편집, 의존성 많은 작업에도 (팀보다) 유리

**Agent Teams** — 팀원이 발견을 공유·반박하고 스스로 조율해야 할 때
- **병렬 코드 리뷰**(보안/성능/테스트 분담), **경쟁 가설 디버깅**(서로 이론 반박), **교차 계층 작업**(프론트/백/테스트 소유), **새 모듈/기능**

### Agent Teams만의 특징

- **활성화 필요**: `CLAUDE_CODE_EXPERIMENTAL_AGENT_TEAMS=1` (settings.json `env` 또는 환경변수)
- **리더 + 팀원 + 공유 작업 목록 + 메일박스** 구조 (`~/.claude/teams/`, `~/.claude/tasks/`)
- **팀원과 직접 대화**: 에이전트 패널 ↑↓ 선택 후 Enter, 또는 분할 창(tmux/iTerm2) 모드
- **계획 승인 요구** 가능: 팀원이 계획 모드로 작업하고 리더 승인 후 구현
- **subagent 정의 재사용**: `security-reviewer` 같은 서브에이전트 유형을 팀원으로도 사용
- **권한**: 팀원은 리더 권한 상속, 팀원 프롬프트는 리더로 버블업. 팀원끼리 전달한 승인은 신뢰 안 됨(우회 방지)

### ⚠️ 주의사항

- **토큰 비용이 훨씬 큼** — 팀원 수만큼 선형 증가. 일상 작업엔 단일 세션이 경제적. **3~5명 권장**
- **실험적 제한** — in-process 팀원은 `/resume`·`/rewind`로 복원 안 됨, **세션당 팀 1개**, **중첩 팀 불가**(팀원은 자기 팀원 못 만듦), 리더 고정
- **파일 충돌 주의** — 두 팀원이 같은 파일 편집 시 덮어쓰기 → 파일 소유를 나눌 것

### 핵심 정리

> **Subagents는 "결과 보고형 위임"(경량·저비용·정식)**, **Agent Teams는 "협업형 다중 에이전트"(고비용·통신·실험적)**.
> 워커끼리 대화가 필요 없으면 **Subagents**, 서로 발견을 공유·도전하며 자체 조율해야 하면 **Agent Teams**.

### 참고 링크

- [세션 팀 조율하기(Agent Teams) — 공식 문서(한국어)](https://code.claude.com/docs/ko/agent-teams) · [Subagents](https://code.claude.com/docs/ko/sub-agents)

---

## Hooks(훅) 실전 활용

**Hooks(훅)** 는 **Claude Code 라이프사이클의 특정 지점에서 자동 실행되는 셸 명령**입니다. LLM이 "실행하기로 선택하는 것"에 의존하지 않고, **특정 작업이 항상 일어나도록 결정론적으로 보장**하는 것이 핵심입니다.

> 예: "편집 후 항상 포맷팅", "커밋 전 항상 테스트", "민감 파일은 절대 수정 금지", "입력 대기 시 데스크톱 알림".
> 이런 **강제**는 CLAUDE.md(권고)로는 안 되고 Hooks로 해야 합니다.

### 주요 이벤트

| 이벤트 | 발생 시점 | 차단 가능 |
| --- | --- | --- |
| `SessionStart` | 세션 시작/재개 시 (matcher `compact`로 압축 후) | — |
| `UserPromptSubmit` | 프롬프트 제출 직후, Claude 처리 전 | ✅ |
| `PreToolUse` | 도구 호출 실행 **전** | ✅ (거부 가능) |
| `PostToolUse` | 도구 호출 성공 **후** | ❌ (이미 실행됨) |
| `PostToolUseFailure` | 도구 호출 실패 후 | — |
| `Notification` | Claude Code가 알림을 보낼 때 | — |
| `SubagentStart` / `SubagentStop` | 서브에이전트 생성/종료 시 | — |
| `Stop` | Claude가 응답을 마칠 때마다 (중단 시엔 X) | ✅ |
| `PreCompact` / `PostCompact` | 컨텍스트 압축 전/후 | — |
| `SessionEnd` | 세션 종료 시 | — |

> 그 외 `FileChanged`, `CwdChanged`, `ConfigChange`, `PermissionRequest`, `InstructionsLoaded` 등 다양한 이벤트가 있습니다.

### 설정 형식

`settings.json`의 `hooks` 블록에 **이벤트 → matcher → 명령** 구조로 등록합니다.

```json
{
  "hooks": {
    "PostToolUse": [
      {
        "matcher": "Edit|Write",
        "hooks": [
          { "type": "command", "command": "jq -r '.tool_input.file_path' | xargs npx prettier --write" }
        ]
      }
    ]
  }
}
```

- **matcher** — 도구 이름 패턴(`Edit|Write`, `Bash` 등). 생략 시 해당 이벤트 전부에 적용. (v2.1.191+는 `Edit,Write`도 허용)
- **type** — 대부분 `command`(셸 명령). 이 외 `http`(URL POST), `mcp_tool`(MCP 도구 호출), `prompt`(LLM 판단), `agent`(다중 턴 검증) 지원
- 같은 이벤트에 여러 hook을 등록하면 **병렬 실행**되고 결과가 병합됨

### 입력과 출력 (통신 방식)

Hook은 **stdin(JSON 입력) · stdout · stderr · 종료 코드**로 Claude Code와 통신합니다.

**입력** — 이벤트 데이터가 JSON으로 stdin에 전달 (`session_id`, `cwd`, `hook_event_name`, `tool_name`, `tool_input` 등). `jq`로 파싱하는 것이 일반적.

**출력 — 종료 코드**

| 종료 코드 | 의미 |
| --- | --- |
| **0** | 이의 없음, 정상 진행. (`UserPromptSubmit`·`SessionStart`는 stdout이 **Claude 컨텍스트에 주입**됨) |
| **2** | 작업 **차단**. stderr에 쓴 이유가 Claude에게 피드백으로 전달됨 |
| 그 외 | 진행하되 `hook error` 공지 + stderr 첫 줄 표시 |

**출력 — 구조화된 JSON** (더 정밀한 제어, exit 0 + stdout에 JSON)

```json
{
  "hookSpecificOutput": {
    "hookEventName": "PreToolUse",
    "permissionDecision": "deny",
    "permissionDecisionReason": "Use rg instead of grep for better performance"
  }
}
```

- `PreToolUse`의 `permissionDecision`: `allow`(프롬프트 건너뜀) / `deny`(취소+이유) / `ask`(사용자에게 질문)
- `UserPromptSubmit`은 `hookSpecificOutput.additionalContext`로 컨텍스트 주입
- ⚠️ exit 2(차단)와 JSON을 **혼용 금지** — exit 2일 때 JSON은 무시됨

### 실전 예시

**1) 편집 후 자동 포맷팅** (위 설정 예시 참고) — `PostToolUse` + `Edit|Write` + Prettier

**2) 보호 파일 편집 차단** — `PreToolUse` + `Edit|Write`, 스크립트가 exit 2로 차단

```bash
#!/bin/bash
INPUT=$(cat)
FILE_PATH=$(echo "$INPUT" | jq -r '.tool_input.file_path // empty')
PROTECTED_PATTERNS=(".env" "package-lock.json" ".git/")
for pattern in "${PROTECTED_PATTERNS[@]}"; do
  if [[ "$FILE_PATH" == *"$pattern"* ]]; then
    echo "Blocked: $FILE_PATH matches protected pattern '$pattern'" >&2
    exit 2
  fi
done
exit 0
```

**3) 입력 대기 시 데스크톱 알림** — `Notification` 이벤트 (macOS `osascript`, Linux `notify-send` 등)

**4) 압축 후 컨텍스트 재주입** — `SessionStart` + matcher `compact`, `echo`한 텍스트가 컨텍스트에 추가

**5) 위험 명령 차단** — `PreToolUse` + `Bash`, `rm -rf`·`drop table` 등 패턴 감지 시 exit 2

### 설정 위치 (범위)

| 위치 | 범위 | 공유 |
| --- | --- | --- |
| `~/.claude/settings.json` | 모든 프로젝트 | 로컬 |
| `.claude/settings.json` | 단일 프로젝트 | git 커밋 가능 |
| `.claude/settings.local.json` | 단일 프로젝트 | gitignored |
| 관리형 정책 설정 | 조직 전체 | 관리자 제어 |
| 플러그인 `hooks/hooks.json` | 플러그인 활성 시 | 플러그인 번들 |
| Skill/Agent frontmatter | 해당 컴포넌트 활성 시 | 컴포넌트에 정의 |

> `/hooks` 명령으로 현재 등록된 hooks를 확인하고, `"disableAllHooks": true`로 전체 비활성화(관리형 설정 제외).

### 보안 · 주의사항

- ⚠️ **Hooks는 여러분의 사용자 권한으로 임의 셸 명령을 실행**합니다. 신뢰할 수 없는 설정/플러그인의 hook은 위험 — 등록 전 반드시 내용 검토
- `PostToolUse`는 도구가 **이미 실행된 뒤**라 취소 불가 (차단하려면 `PreToolUse` 사용)
- `Stop`은 매 응답 종료마다 발생(작업 완료 시점만이 아님), 사용자 중단 시엔 미발생
- 여러 hook이 같은 도구 입력을 수정하면 순서가 비결정적 — 입력 수정 hook은 하나만 두기
- 타임아웃: `command`/`http` 기본 10분, `UserPromptSubmit` 30초, `prompt` 30초, `agent` 60초 (`timeout` 필드로 조정)

### 핵심 정리

> **Hooks = "항상 일어나야 하는 일을 코드로 강제"**.
> CLAUDE.md가 *권고*라면 Hooks는 *강제*. `PreToolUse`로 사전 차단·검증, `PostToolUse`로 사후 자동화(포맷·테스트), `Notification`/`Stop`으로 알림, `SessionStart`로 컨텍스트 주입에 주로 씁니다.

### 참고 링크

- [hooks를 사용하여 작업 자동화 — 공식 가이드(한국어)](https://code.claude.com/docs/ko/hooks-guide)
- [Hooks 참조(전체 스키마) — 공식 문서(한국어)](https://code.claude.com/docs/ko/hooks)

---

## Skills 만들기

**Skills(스킬)** 는 **Claude가 할 수 있는 작업을 확장하는 재사용 가능한 지침·절차 패키지**입니다. `SKILL.md` 파일에 지침을 적어두면 Claude가 이를 자기 도구 모음에 추가하고, **관련 있을 때 자동으로 로드**하거나 **`/skill-name`으로 직접 호출**할 수 있습니다.

> 같은 지침·체크리스트·다단계 절차를 반복해서 채팅에 붙여넣거나, CLAUDE.md의 한 섹션이 "사실"이 아니라 "절차"로 자라났을 때 skill로 옮기세요.
> CLAUDE.md와 달리 **skill 본문은 호출될 때만 로드**되므로(점진적 공개), 긴 참조 자료도 필요할 때까지 토큰 비용이 거의 없습니다.

> 📌 참고: **사용자 정의 슬래시 명령어(`.claude/commands/`)가 Skills로 통합**되었습니다. 기존 `commands/` 파일도 계속 작동하지만, 지원 파일·frontmatter·자동 로드가 되는 Skills가 권장됩니다.

### 만드는 방법 (최소 예시)

각 skill은 `SKILL.md`를 진입점으로 하는 **디렉토리**입니다. **디렉토리 이름이 곧 명령어 이름**이 됩니다.

```bash
mkdir -p ~/.claude/skills/summarize-changes
```

`~/.claude/skills/summarize-changes/SKILL.md`:

```yaml
---
description: Summarizes uncommitted changes and flags anything risky. Use when the user asks what changed, wants a commit message, or asks to review their diff.
---

## Current changes

!`git diff HEAD`

## Instructions

위 변경을 2~3개 불릿으로 요약하고, 누락된 에러 처리·하드코딩·수정 필요한 테스트 등 위험을 나열하세요. diff가 비어 있으면 커밋되지 않은 변경이 없다고 답하세요.
```

- 자동 호출: "What did I change?"처럼 `description`과 맞는 요청 → Claude가 알아서 로드
- 직접 호출: `/summarize-changes`
- `` !`git diff HEAD` `` 는 **동적 컨텍스트 주입** — skill이 Claude에 도달하기 전에 명령을 실행해 출력으로 치환

### 저장 위치 (범위)

| 위치 | 경로 | 적용 대상 |
| --- | --- | --- |
| Enterprise | 관리 설정 | 조직 전체 |
| Personal | `~/.claude/skills/<name>/SKILL.md` | 모든 프로젝트 |
| Project | `.claude/skills/<name>/SKILL.md` | 이 프로젝트만 (git 커밋 → 팀 공유) |
| Plugin | `<plugin>/skills/<name>/SKILL.md` | 플러그인 활성 위치 |

> 같은 이름이면 Enterprise > Personal > Project 순으로 우선. skill과 command가 같은 이름이면 skill이 우선.

### 주요 frontmatter 필드

| 필드 | 설명 |
| --- | --- |
| `description` | (권장) 무엇을 하는지·언제 쓰는지. **자동 로드 판단 근거** |
| `disable-model-invocation` | `true`면 **사용자만 호출**(Claude 자동 실행 방지). `/deploy`·`/commit`처럼 부작용 있는 워크플로우용 |
| `user-invocable` | `false`면 **Claude만 호출**(`/` 메뉴에서 숨김). 배경 지식용 |
| `allowed-tools` | skill 활성 시 승인 없이 쓸 도구 (예: `Bash(git add *)`) |
| `argument-hint` / `arguments` | 인수 힌트·명명 인수 |
| `context: fork` | 격리된 서브에이전트에서 실행 (`agent`로 유형 지정) |
| `paths` | glob 패턴 일치 파일 작업 시에만 자동 로드 |

### 인수·지원 파일

- **인수**: `$ARGUMENTS`(전체), `$ARGUMENTS[N]`/`$N`(위치별). 예: `/fix-issue 123` → `Fix GitHub issue 123 ...`
- **지원 파일**: 디렉토리에 `reference.md`, `examples.md`, `scripts/helper.py` 등을 두고 `SKILL.md`에서 참조 → 필요할 때만 로드. `SKILL.md`는 500줄 이하로 유지

### 모범 사례

- **`description`에 사용자가 자연스럽게 쓸 키워드**를 넣어 자동 트리거 정확도↑
- **본문은 간결하게** — 로드되면 턴 내내 컨텍스트에 남아 토큰을 소비. "왜"보다 "무엇을 할지"
- 부작용 있는 작업(`/deploy`)은 `disable-model-invocation: true`
- 관리 명령: **`/skills`**(목록·가시성 제어), **`/reload-skills`**(변경 즉시 반영)

### 참고 링크

- [Claude를 skills로 확장하기 — 공식 문서(한국어)](https://code.claude.com/docs/ko/skills)

---

## 권한 모드 (Permission Modes)

**권한 모드**는 Claude가 파일 편집·명령 실행·네트워크 요청 전에 **얼마나 자주 승인을 요청하는지**를 제어합니다. 편의성과 감시(통제) 사이의 트레이드오프를 정합니다.

### 모드 종류

| 모드 | 요청 없이 실행되는 작업 | 최적 사용 |
| --- | --- | --- |
| `default` (Manual) | **읽기만** | 시작·민감한 작업 |
| `acceptEdits` | 읽기 + 파일 편집 + 일반 파일시스템 명령(`mkdir`,`mv`,`cp`,`rm` 등) | 검토 중인 코드 반복 작업 |
| `plan` | **읽기만** (편집 금지, 계획만 제시) | 변경 전 코드베이스 탐색 |
| `auto` | 백그라운드 안전 분류기 검사를 거친 대부분의 작업 | 장시간 작업, 프롬프트 피로 감소 |
| `dontAsk` | **사전 승인된 도구만** (나머지는 자동 거부) | 제한된 CI·스크립트 |
| `bypassPermissions` | **모든 작업** (확인 건너뜀) | 격리된 컨테이너·VM 전용 |

### 전환 방법 (CLI)

- **세션 중**: `Shift+Tab` 으로 `default` → `acceptEdits` → `plan` 순환 (상태 표시줄에 현재 모드 표시)
- **시작 시**: `claude --permission-mode plan`
- **기본값 고정**: `settings.json`의 `permissions.defaultMode`
  ```json
  { "permissions": { "defaultMode": "acceptEdits" } }
  ```
- `auto`·`dontAsk`·`bypassPermissions`는 기본 순환에 없고 플래그/설정으로 활성화

### 주의사항

- ⚠️ **`bypassPermissions`는 프롬프트 주입·오작동 보호가 전혀 없습니다.** 인터넷 없는 격리 컨테이너/VM에서만 사용. 프롬프트만 줄이려면 **`auto` 모드**(백그라운드 안전 검사 포함)를 대신 사용
- **보호된 경로**(`.git`, `.claude`, `.bashrc`, `.mcp.json` 등)는 `bypassPermissions`를 제외한 모든 모드에서 자동 승인되지 않음
- **`auto` 모드**는 별도 분류기가 위험 작업(외부 유출·프로덕션 배포·force push·대량 삭제 등)을 차단. 단, 안전을 보장하진 않으니 민감 작업 검토를 대체하지 말 것 (플랜·모델·프로바이더 요구사항 있음)
- 모드 위에 **권한 규칙(allow/ask/deny)** 을 계층화해 특정 도구를 사전 승인/차단 가능

### 참고 링크

- [권한 모드 선택 — 공식 문서(한국어)](https://code.claude.com/docs/ko/permission-modes) · [권한 규칙](https://code.claude.com/docs/ko/permissions)

---

## Plan Mode (계획 모드)

**Plan Mode(계획 모드)** 는 Claude가 **코드를 변경하지 않고, 먼저 조사·분석해 계획만 제시**하도록 하는 모드입니다. 파일을 읽고 명령으로 탐색한 뒤 계획을 세우지만, **소스 편집은 계획을 승인할 때까지 차단**됩니다.

> "바로 고치지 말고 먼저 어떻게 할지 보여줘"가 필요할 때 — 큰 변경, 리스크 높은 리팩터, 낯선 코드베이스에 이상적입니다.

### 진입·종료

- **진입**: `Shift+Tab`으로 순환하거나, 단일 프롬프트 앞에 `/plan` 붙이기, 또는 `claude --permission-mode plan`
- **종료(미승인)**: `Shift+Tab`을 다시 눌러 계획 모드 해제
- **기본값 고정**: `settings.json`의 `permissions.defaultMode: "plan"`

### 계획 검토·승인

계획이 준비되면 Claude가 제시하고 진행 방법을 묻습니다:

- **자동 모드로 승인·시작** / **편집 승인·수락** / **각 편집 수동 검토** / **피드백 주고 계획 계속** / **Ultraplan(브라우저 검토)로 개선**
- 승인하면 계획 모드가 끝나고 선택한 권한 모드로 전환되어 편집을 시작
- `Ctrl+G` — 제안된 계획을 텍스트 편집기에서 직접 수정
- 계획 수락 시 계획 내용으로 세션 이름이 자동 지정됨

### 참고 링크

- [편집 전에 계획 모드로 분석하기 — 공식 문서(한국어)](https://code.claude.com/docs/ko/permission-modes#analyze-before-you-edit-with-plan-mode)

---

## Slash Commands (슬래시 명령어)

Claude Code 실행 중 **`/`로 시작하는 명령어**로 세션 관리·설정·리뷰 등을 수행합니다. 기본 제공 명령어와 사용자 정의 명령어(= Skills)가 있습니다.

### 자주 쓰는 기본 제공 명령어

**세션·프로젝트**

| 명령어 | 기능 |
| --- | --- |
| `/help` | 도움말·명령어 목록 |
| `/clear` | 빈 컨텍스트로 새 대화 (메모리는 유지) |
| `/resume` | 이전 대화 재개 / 세션 선택 |
| `/init` | `CLAUDE.md` 생성 |
| `/memory` | CLAUDE.md·auto-memory 편집/토글 |
| `/add-dir` | 작업 디렉토리 추가 |

**모델·설정·상태**

| 명령어 | 기능 |
| --- | --- |
| `/model` | AI 모델 전환 |
| `/config` | 설정 인터페이스 |
| `/status` | 버전·모델·계정·프로바이더·연결 확인 |
| `/permissions` | 도구 권한 규칙(허용/요청/거부) 관리 |
| `/mcp` | MCP 서버 연결·OAuth 인증 관리 |
| `/hooks` | 등록된 hook 확인 |
| `/doctor` | 설치·설정 진단 및 정리 |

**리뷰·컨텍스트·워크플로우**

| 명령어 | 기능 |
| --- | --- |
| `/code-review` | diff의 정확성 버그 검토 (`--fix`로 수정 적용) |
| `/security-review` | 보안 취약점 분석 |
| `/diff` | 커밋되지 않은 변경 표시 |
| `/context` | 컨텍스트 사용량 시각화 |
| `/compact` | 대화 요약해 컨텍스트 확보 |
| `/plan` | 계획 모드 진입 |
| `/agents` | 서브에이전트 생성·관리 |
| `/skills` | skill 목록·가시성 제어 |
| `/rewind` | 대화/코드를 이전 지점으로 되감기 |

### 사용자 정의 명령어 만들기

사용자 정의 슬래시 명령어는 **[Skills](#skills-만들기)로 생성**합니다. `.claude/skills/<name>/SKILL.md`(또는 기존 `.claude/commands/<name>.md`)를 만들면 **`/<name>`** 으로 호출됩니다.

- **인수 전달**: `/fix-issue 123` → skill 내 `$ARGUMENTS`가 `123`으로 치환
- **명령어 스택**: `/code-review /fix-issue 123`처럼 여러 skill을 한 번에(최대 6개) 호출 (v2.1.199+)
- **부작용 있는 명령**(`/deploy` 등)은 `disable-model-invocation: true`로 사용자 전용 지정

> 자세한 작성법은 위 [Skills 만들기](#skills-만들기) 섹션 참조.

### 참고 링크

- [명령어 참조 — 공식 문서(한국어)](https://code.claude.com/docs/ko/commands)

---

## Slash Commands vs Skill 비교

이 둘은 **대등하게 비교하기 어렵습니다** — 최근 Claude Code에서 **사용자 정의 슬래시 명령어가 Skills로 통합**되었기 때문입니다. 층위가 다릅니다.

> - **Slash Command** = **"호출 방식(입력 표면)"** — `/이름`을 타이핑해서 실행하는 것
> - **Skill** = **"실제 기능 패키지"** — 그 `/명령`이 실행하는 지침·절차의 실체

즉 **"슬래시 명령어"는 Skill을 부르는 여러 방법 중 하나**입니다. 대체 관계가 아니라 포함 관계에 가깝습니다.

### 슬래시 명령어의 세 종류

| 종류 | 예시 | 실체 |
| --- | --- | --- |
| **기본 제공 명령어** | `/help`, `/clear`, `/compact`, `/model` | 고정 로직 (Skill 아님) |
| **번들 Skill** | `/code-review`, `/debug`, `/run`, `/loop` | **Skill** (프롬프트 기반) |
| **사용자 정의 명령어** | `/deploy`, `/fix-issue` | **Skill** (또는 레거시 `commands/` 파일) |

→ `/help` 같은 건 Skill이 아니지만, `/code-review`나 직접 만든 `/deploy`는 속을 보면 **Skill**입니다.

### Skill vs (사용자 정의) Slash Command

과거에는 `.claude/commands/deploy.md`(명령어)와 `.claude/skills/deploy/SKILL.md`(스킬)가 별개였지만, **지금은 둘 다 `/deploy`를 만들고 동일하게 작동**합니다. 기존 `commands/` 파일도 계속 지원되지만 Skill이 더 많은 기능을 제공합니다.

| 항목 | 레거시 Slash Command (`commands/`) | **Skill** (`skills/`) |
| --- | --- | --- |
| **파일 구조** | 단일 `.md` 파일 | **디렉토리** (`SKILL.md` + 지원 파일·스크립트) |
| **호출 주체** | 사용자만 (`/name`) | **사용자 + Claude 자동 호출** |
| **자동 로드** | ❌ | ✅ (`description` 기반, 관련 있을 때 자동) |
| **점진적 공개** | — | ✅ 본문은 호출될 때만 로드(토큰 절약) |
| **호출 제어 frontmatter** | 제한적 | `disable-model-invocation`, `user-invocable`, `allowed-tools`, `paths`, `context: fork` 등 |
| **지원 파일·스크립트** | ❌ | ✅ 템플릿·예제·`scripts/` 번들 |
| **인수** | 지원 | 지원 (`$ARGUMENTS`, `$N`) |

### 결정적 차이 — "자동 호출" 여부

가장 중요한 실질적 차이는 **누가 호출하는가**입니다.

- **순수 슬래시 명령어 성격** → *사용자가* `/명령`을 쳐야만 실행 (수동)
- **Skill 성격** → 사용자가 `/명령`으로 호출할 수도, **Claude가 관련 있다고 판단해 자동으로** 로드할 수도 있음

그래서 실무에서는 frontmatter로 성격을 조절합니다:

```yaml
# "슬래시 명령어처럼" — 사용자만 수동 실행 (부작용 있는 작업)
---
description: Deploy to production
disable-model-invocation: true   # Claude 자동 실행 방지
---

# "Skill답게" — Claude가 상황 봐서 자동 적용 (배경 지식·규칙)
---
description: API design patterns for this codebase
---
```

### 핵심 정리

> - **Slash Command**은 *실행하는 방법*(`/이름` 타이핑), **Skill**은 *실행되는 실체*(기능 패키지)
> - 사용자 정의 슬래시 명령어는 이제 **Skill로 통합** — `commands/*.md`도 되지만 `skills/*/SKILL.md`가 권장
> - 핵심 차이는 **자동 호출**: 슬래시 명령어는 "사용자 수동", Skill은 "사용자 + Claude 자동" (frontmatter로 조절)
> - `/help`·`/clear` 같은 **기본 제공 명령어는 Skill이 아닌 고정 로직**

### 참고 링크

- [Claude를 skills로 확장하기 — 공식 문서(한국어)](https://code.claude.com/docs/ko/skills) · [명령어 참조](https://code.claude.com/docs/ko/commands)

---

## Checkpoints & Rewind (체크포인트·되감기)

**Checkpoints(체크포인트)** 는 Claude Code가 작업 중 **Claude의 파일 편집을 자동 추적해 남기는 스냅샷**이고, **Rewind(되감기)** 는 그 스냅샷으로 **이전 상태로 되돌리는** 기능입니다. 한마디로 **"세션 단위 로컬 undo(실행 취소)"** 안전장치입니다.

> 무언가 잘못돼도 언제든 되돌릴 수 있으니, 큰 변경도 확신을 갖고 시도할 수 있습니다.

### 체크포인트 동작 방식

- **생성 시점**: **매 사용자 프롬프트 직전** 코드 상태를 자동 캡처 (프롬프트 하나 = 체크포인트 하나)
- **보관**: 세션의 **최근 100개** 스냅샷 유지, 세션과 함께 저장돼 **재개한 세션에서도 사용 가능**, **30일 후 자동 정리**(설정 가능)
- **추적 대상**: **파일 편집 도구(Edit/Write)로 한 변경만** 추적

### Rewind 사용법

- **여는 법**: `/rewind` 명령 (별칭 `/checkpoint`, `/undo`), 또는 **입력창이 빈 상태에서 `Esc` 두 번**
  - ⚠️ 입력창에 텍스트가 있으면 `Esc Esc`는 메뉴 대신 **텍스트를 지웁니다** (지운 텍스트는 `↑`로 복구)
- 메뉴에 **세션 중 보낸 각 프롬프트가 나열**됨 → 되돌릴 지점을 고르고 동작 선택:

| 동작 | 되돌리는 것 |
| --- | --- |
| **코드 및 대화 복원** | 코드 + 대화 모두 그 지점으로 |
| **대화 복원** | 대화만 (현재 코드 유지) |
| **코드 복원** | 파일 변경만 (대화 유지) |
| **여기서부터 요약** | 선택 지점 이후 대화를 요약 압축 (이전은 그대로) |
| **여기까지 요약** | 선택 지점 이전 대화를 요약 압축 (이후는 그대로) |
| **취소** | 아무것도 적용 안 함 |

- 대화 복원/여기서부터 요약을 고르면 **선택한 프롬프트가 입력창에 복원**되어 다시 보내거나 편집 가능
- `/clear` 했던 대화로도 되돌리기: 메뉴 맨 위 `/resume <id> (이전 세션)` 항목 (v2.1.191+)

### 복원(Restore) vs 요약(Summarize)

- **복원** = 상태를 **되돌림**(코드/대화/둘 다 실행 취소)
- **요약** = 디스크 파일은 **안 건드리고** 대화 일부를 AI 요약으로 압축해 **컨텍스트 공간 확보** (`/compact`의 타깃 지정 버전, 원본은 세션 기록에 보존)

> 💡 원본 세션을 **그대로 두고 다른 접근을 시도**하려면 요약 대신 **fork**: `claude --continue --fork-session`

### ⚠️ 한계 (매우 중요)

체크포인트가 **되돌리지 못하는 것**:

1. **Bash 명령으로 바뀐 파일** — `rm`·`mv`·`cp` 등으로 수정/삭제한 파일은 **rewind로 복구 불가**. 오직 편집 도구로 한 편집만 추적
2. **외부 변경** — Claude Code 밖에서 수동 편집한 파일, 다른 동시 세션의 편집은 추적 안 됨
3. **버전 관리의 대체가 아님** — 세션 단위 빠른 복구용. 영구 기록·협업·브랜치는 **Git을 계속 사용**

> **체크포인트 = "로컬 undo", Git = "영구 기록"**

### 활용 사례

- **대안 탐색** — 시작점을 잃지 않고 여러 구현 방식 시도
- **실수 복구** — 버그 넣은 변경을 빠르게 취소
- **기능 반복** — 작동하는 상태로 되돌아갈 수 있다는 확신으로 실험
- **컨텍스트 확보** — 초기 지침은 유지하고 중간부터의 긴 디버깅 대화를 요약

### 참고 링크

- [Checkpointing — 공식 문서(한국어)](https://code.claude.com/docs/ko/checkpointing)
