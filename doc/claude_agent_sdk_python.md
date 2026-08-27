# Claude Agent SDK — Python 프로그래밍 가이드

> Claude Agent SDK(구 Claude Code SDK)로 **Python에서 커스텀 AI 에이전트를 만드는 법**을 정리한 문서입니다.
> Claude Code를 라이브러리로 사용해 도구·에이전트 루프·컨텍스트 관리를 그대로 프로그래밍합니다.
> 최종 정리일: 2026-08-27 · 출처: [Agent SDK Python 참조](https://code.claude.com/docs/en/agent-sdk/python) · [GitHub](https://github.com/anthropics/claude-agent-sdk-python)

> 📚 관련 문서: 개념·인증·오픈소스 여부는 [`claude_code_tips.md`의 Claude Agent SDK 섹션](./claude_code_tips.md#claude-agent-sdk), CLI 사용은 [`claude_code_cli.md`](./claude_code_cli.md) 참고.

## 목차

1. [설치](#1-설치)
2. [두 가지 진입점: query() vs ClaudeSDKClient](#2-두-가지-진입점-query-vs-claudesdkclient)
3. [ClaudeAgentOptions (설정)](#3-claudeagentoptions-설정)
4. [메시지·콘텐츠 타입](#4-메시지콘텐츠-타입)
5. [커스텀 도구 (@tool · MCP 서버)](#5-커스텀-도구-tool--mcp-서버)
6. [권한 처리 (can_use_tool)](#6-권한-처리-can_use_tool)
7. [스트리밍 입력](#7-스트리밍-입력)
8. [인터럽트](#8-인터럽트)
9. [서브에이전트 (AgentDefinition)](#9-서브에이전트-agentdefinition)
10. [세션 관리](#10-세션-관리)
11. [인증 (사내 LLM 포함)](#11-인증-사내-llm-포함)

---

## 1. 설치

### 방법 A — pip 설치 (일반 사용)

```bash
python3 -m venv .venv
source .venv/bin/activate
pip install claude-agent-sdk
```

- **요구사항**: Python **3.10+**
- **Claude Code CLI**: 패키지가 **자동으로 번들**하므로 별도 설치 불필요
- Python 라이브러리 (MIT 라이선스). 비동기(`asyncio`) 기반
- 인증은 API 키·사내 게이트웨이·Bedrock/Vertex/Foundry 모두 가능([11장](#11-인증-사내-llm-포함))

### 방법 B — 저장소 clone 후 개발 (예제 실행·소스 참조·기여)

공식 저장소를 직접 clone하면 **예제 코드를 바로 실행**하고 소스를 참조하며 개발할 수 있습니다.

```bash
# 1) 저장소 clone
git clone https://github.com/anthropics/claude-agent-sdk-python.git
cd claude-agent-sdk-python

# 2) 가상환경 생성·활성화
python3 -m venv .venv
source .venv/bin/activate          # Windows PowerShell: .venv\Scripts\Activate.ps1

# 3) 편집 가능(editable) 설치 — 소스 수정이 바로 반영됨
pip install -e .
```

**포함된 예제 실행** (`examples/` 디렉토리):

```bash
python examples/quick_start.py          # 기본 query() 예제
python examples/streaming_mode.py       # 스트리밍/다중 턴
python examples/mcp_calculator.py       # 커스텀 도구(@tool) + MCP 서버

# IPython 대화형 탐색
ipython -i examples/streaming_mode_ipython.py
```

**내 스크립트로 시작하기** — clone 없이도 되지만, 저장소 안에서 바로 짜볼 수 있습니다:

```python
# my_agent.py
import anyio
from claude_agent_sdk import query

async def main():
    async for message in query(prompt="What is 2 + 2?"):
        print(message)

anyio.run(main)
```

```bash
python my_agent.py
```

> 💡 예제는 `anyio.run(main)`을, 이 문서 다른 예제는 `asyncio.run(main())`을 씁니다 — 둘 다 async 진입점을 실행하는 방법이며 동작은 같습니다.

**기여자용 개발 설정** (선택): 저장소에는 CI와 동일한 lint를 걸어주는 git 훅 설정 스크립트가 있습니다.

```bash
./scripts/initial-setup.sh    # pre-push lint 훅 설치 (임시 우회: git push --no-verify)
```

---

## 2. 두 가지 진입점: query() vs ClaudeSDKClient

| 기능 | `query()` | `ClaudeSDKClient` |
| --- | --- | --- |
| **세션** | 매번 새로 | 같은 세션 재사용 |
| **대화** | 단일 교환 | 다중 턴 |
| **연결** | 자동 관리 | 수동 제어 |
| **스트리밍 입력** | ✅ | ✅ |
| **인터럽트** | ❌ | ✅ |
| **Hooks / 커스텀 도구** | ✅ / ✅ | ✅ / ✅ |
| **대화 이어가기** | 수동(`continue_conversation`/`resume`) | ✅ 자동 |
| **용도** | 일회성 작업 | 연속 대화·인터랙티브 |

### query() — 일회성 작업

프롬프트를 주면 메시지를 **비동기 이터레이터**로 스트리밍하고 종료합니다.

```python
import asyncio
from claude_agent_sdk import query, ClaudeAgentOptions

async def main():
    options = ClaudeAgentOptions(
        system_prompt="You are an expert Python developer",
        permission_mode="acceptEdits",
    )
    async for message in query(prompt="Create a Python web server", options=options):
        print(message)

asyncio.run(main())
```

### ClaudeSDKClient — 다중 턴 대화

**세션을 유지**하며 여러 번 주고받고, 인터럽트·모델 변경 등 고급 제어가 가능합니다.

```python
import asyncio
from claude_agent_sdk import ClaudeSDKClient, AssistantMessage, TextBlock

async def main():
    async with ClaudeSDKClient() as client:
        await client.query("What's the capital of France?")
        async for message in client.receive_response():
            if isinstance(message, AssistantMessage):
                for block in message.content:
                    if isinstance(block, TextBlock):
                        print(f"Claude: {block.text}")

        # 후속 질문 — 이전 컨텍스트 유지
        await client.query("What's the population of that city?")
        async for message in client.receive_response():
            if isinstance(message, AssistantMessage):
                for block in message.content:
                    if isinstance(block, TextBlock):
                        print(f"Claude: {block.text}")

asyncio.run(main())
```

**주요 메서드**: `connect()`, `query(prompt, session_id)`, `receive_messages()`, `receive_response()`, `interrupt()`, `set_permission_mode()`, `set_model()`, `rewind_files()`, `get_mcp_status()`, `disconnect()` 등. `async with`로 자동 연결/해제.

> `receive_messages()`는 모든 메시지를 계속 받고, `receive_response()`는 **한 응답 턴이 끝날 때까지**만 받습니다.

---

## 3. ClaudeAgentOptions (설정)

에이전트 동작을 제어하는 핵심 설정 객체입니다.

| 속성 | 타입 | 설명 |
| --- | --- | --- |
| `system_prompt` | `str` / preset / file | 시스템 프롬프트 (커스텀·프리셋) |
| `tools` | `list[str]` / preset | 도구 구성 (`{"type":"preset","preset":"claude_code"}`로 기본값) |
| `allowed_tools` | `list[str]` | 프롬프트 없이 자동 승인할 도구 |
| `disallowed_tools` | `list[str]` | 명시적으로 거부할 도구 |
| `permission_mode` | `PermissionMode` | `default`/`acceptEdits`/`plan`/`dontAsk`/`bypassPermissions`/`auto` |
| `model` | `str` | 모델 별칭·전체 ID |
| `thinking` | `ThinkingConfig` | 확장 사고 설정 |
| `effort` | `low`~`max` | 추론 노력 수준 |
| `mcp_servers` | `dict` | MCP 서버 구성 |
| `continue_conversation` / `resume` / `session_id` | | 세션 이어가기·재개·지정 |
| `max_turns` / `max_budget_usd` | | 최대 턴 수 / 최대 비용($) |
| `can_use_tool` | 콜백 | 도구 권한 콜백([6장](#6-권한-처리-can_use_tool)) |
| `hooks` | `dict` | Hook 구성 |
| `agents` | `dict[str, AgentDefinition]` | 서브에이전트 정의([9장](#9-서브에이전트-agentdefinition)) |
| `skills` | `list[str]` / `"all"` | 사용 가능 Skills |
| `output_format` | `dict` | JSON Schema 검증(구조화 출력) |
| `cwd` / `env` | | 작업 디렉토리 / 환경변수 |
| `enable_file_checkpointing` | `bool` | rewind용 파일 변경 추적 |
| `include_partial_messages` | `bool` | 스트리밍 `StreamEvent` 포함 |

### ThinkingConfig

```python
{"type": "adaptive", "display": "summarized"}              # Claude가 알아서 사고
{"type": "enabled", "budget_tokens": 20000, "display": "summarized"}  # 토큰 예산
{"type": "disabled"}                                        # 비활성화
```

---

## 4. 메시지·콘텐츠 타입

`query()`·`receive_messages()`가 yield하는 메시지들입니다. `isinstance()`로 분기해 처리합니다.

### 메시지 타입

| 타입 | 의미 | 주요 필드 |
| --- | --- | --- |
| `AssistantMessage` | Claude의 응답 | `content: list[ContentBlock]`, `session_id` |
| `UserMessage` | 사용자 입력 | `content`, `session_id` |
| `ResultMessage` | 턴 종료(사용량·비용) | `result`, `cost_data`, `terminal_reason` |
| `StreamEvent` | 부분 스트리밍 이벤트 | `event` (partial 활성 시) |
| `TaskNotificationMessage` | 백그라운드 작업 상태 | `status`, `task_id` |

`ResultMessage.terminal_reason`: `stop_sequence`/`max_tokens`/`tool_use`/`end_turn`/`aborted_streaming`/`aborted_tools`

### 콘텐츠 블록 (`AssistantMessage.content` 안)

| 블록 | 의미 | 주요 필드 |
| --- | --- | --- |
| `TextBlock` | 일반 텍스트 | `text` |
| `ToolUseBlock` | 도구 호출 | `id`, `name`, `input` |
| `ToolResultBlock` | 도구 실행 결과 | `tool_use_id`, `content`, `is_error` |
| `ThinkingBlock` | 확장 사고 | `thinking` |

```python
from claude_agent_sdk import AssistantMessage, TextBlock, ToolUseBlock, ResultMessage

async for message in query(prompt="..."):
    if isinstance(message, AssistantMessage):
        for block in message.content:
            if isinstance(block, TextBlock):
                print(block.text)
            elif isinstance(block, ToolUseBlock):
                print(f"도구 호출: {block.name}({block.input})")
    elif isinstance(message, ResultMessage):
        print(f"종료: {message.terminal_reason}, 비용: {message.cost_data}")
```

---

## 5. 커스텀 도구 (@tool · MCP 서버)

`@tool` 데코레이터로 **직접 도구를 정의**하고, `create_sdk_mcp_server`로 묶어 에이전트에 연결합니다. (프로세스 내 인메모리 MCP 서버)

```python
from claude_agent_sdk import tool, create_sdk_mcp_server, query, ClaudeAgentOptions

@tool("add", "Add two numbers", {"a": float, "b": float})
async def add(args):
    return {"content": [{"type": "text", "text": f"Sum: {args['a'] + args['b']}"}]}

@tool("multiply", "Multiply two numbers", {"a": float, "b": float})
async def multiply(args):
    return {"content": [{"type": "text", "text": f"Product: {args['a'] * args['b']}"}]}

calculator = create_sdk_mcp_server(name="calculator", version="2.0.0", tools=[add, multiply])

options = ClaudeAgentOptions(
    mcp_servers={"calc": calculator},
    allowed_tools=["mcp__calc__add", "mcp__calc__multiply"],  # mcp__<서버>__<도구>
)

async def main():
    async for msg in query("What is 21 doubled?", options=options):
        print(msg)
```

### 입력 스키마

```python
# 간단한 타입 매핑 (권장)
{"text": str, "count": int, "enabled": bool}

# 또는 JSON Schema
{"type": "object", "properties": {"text": {"type": "string"}}, "required": ["text"]}
```

### 도구 반환 형식

```python
return {"content": [{"type": "text", "text": "결과 문자열"}]}
```

### ToolAnnotations (선택)

`readOnlyHint`(환경 변경 없음), `destructiveHint`, `idempotentHint`, `openWorldHint`(외부 상호작용), `title`로 도구 성격을 표시합니다.

```python
from claude_agent_sdk import tool, ToolAnnotations

@tool("search", "Search the web", {"query": str},
      annotations=ToolAnnotations(readOnlyHint=True, openWorldHint=True, title="Web Search"))
async def search(args):
    return {"content": [{"type": "text", "text": f"Results for: {args['query']}"}]}
```

### 외부 MCP 서버 연결

인메모리 도구 대신 외부 서버도 연결 가능: **stdio**(로컬 명령), **SSE**, **HTTP** 타입 지원.

```python
options = ClaudeAgentOptions(mcp_servers={
    "fs": {"type": "stdio", "command": "npx", "args": ["-y", "@modelcontextprotocol/server-filesystem", "/path"]},
    "remote": {"type": "http", "url": "https://mcp.example.com", "headers": {"Authorization": "Bearer ..."}},
})
```

---

## 6. 권한 처리 (can_use_tool)

도구 실행 전에 **직접 허용/거부/입력 수정**을 결정하는 콜백을 등록합니다.

```python
from claude_agent_sdk import ClaudeSDKClient, ClaudeAgentOptions
from claude_agent_sdk.types import PermissionResultAllow, PermissionResultDeny, ToolPermissionContext

async def custom_permission_handler(tool_name, input_data, context: ToolPermissionContext):
    # 시스템 디렉토리 쓰기 차단
    if tool_name == "Write" and input_data.get("file_path", "").startswith("/system/"):
        return PermissionResultDeny(message="System write not allowed", interrupt=True)

    # 민감 파일은 sandbox로 경로 변경(입력 수정)
    if tool_name in ["Write", "Edit"] and "config" in input_data.get("file_path", ""):
        safe_path = f"./sandbox/{input_data['file_path']}"
        return PermissionResultAllow(updated_input={**input_data, "file_path": safe_path})

    return PermissionResultAllow(updated_input=input_data)

options = ClaudeAgentOptions(can_use_tool=custom_permission_handler)
```

- **`PermissionResultAllow`** — `updated_input`으로 도구 입력을 수정해 승인 가능
- **`PermissionResultDeny`** — `message`(이유), `interrupt=True`로 즉시 중단
- **`ToolPermissionContext`** — `tool_use_id`, `blocked_path`, `decision_reason`, `title`(전체 프롬프트 문장), `display_name` 등 제공

> 권한 모드(`permission_mode`)로 세션 기본값을 정하고, `can_use_tool`로 세밀하게 재정의하는 조합이 일반적입니다.

---

## 7. 스트리밍 입력

프롬프트를 **비동기 제너레이터**로 넘겨, 메시지를 동적으로 흘려보낼 수 있습니다.

```python
import asyncio
from claude_agent_sdk import query

async def message_stream():
    yield {"type": "user", "message": {"role": "user", "content": "Analyze this data:"}}
    await asyncio.sleep(0.5)
    yield {"type": "user", "message": {"role": "user", "content": "Value: 42"}}

async def main():
    async for message in query(prompt=message_stream()):
        print(message)
```

`ClaudeSDKClient`에서도 `await client.query(message_stream())`로 동일하게 사용 가능합니다.

---

## 8. 인터럽트

**`ClaudeSDKClient`에서만** 실행 중인 작업을 중단할 수 있습니다.

```python
import asyncio
from claude_agent_sdk import ClaudeSDKClient, ClaudeAgentOptions, ResultMessage

async def interruptible_task():
    options = ClaudeAgentOptions(allowed_tools=["Bash"], permission_mode="acceptEdits")
    async with ClaudeSDKClient(options=options) as client:
        await client.query("Count from 1 to 100 slowly, using sleep")
        await asyncio.sleep(2)
        await client.interrupt()          # 중단

        # ⚠️ 새 쿼리 전에 반드시 버퍼를 비워야 함
        async for message in client.receive_response():
            if isinstance(message, ResultMessage):
                print(f"Interrupted: {message.terminal_reason}")  # aborted_streaming/aborted_tools

        await client.query("Just say hello instead")
        async for message in client.receive_response():
            print(message)
```

> **주의**: `interrupt()` 후 `receive_response()`로 **중단된 작업의 메시지를 먼저 소진**하지 않으면, 다음 쿼리에서 엉뚱한(중단된) 응답을 받게 됩니다.

---

## 9. 서브에이전트 (AgentDefinition)

`agents` 옵션으로 **프로그래밍 방식 서브에이전트**를 정의합니다. 파일(`.claude/agents/`) 없이 코드로 역할을 부여합니다.

```python
from claude_agent_sdk import AgentDefinition, ClaudeAgentOptions, query

async def main():
    async for message in query(
        prompt="Review this PR",
        options=ClaudeAgentOptions(
            agents={
                "code-reviewer": AgentDefinition(
                    description="Reviews code changes",
                    prompt="You are a code reviewer. Report issues in the diff.",
                ),
            },
            allowed_tools=["Read", "Grep", "Glob"],
        ),
    ):
        print(message)
```

**`AgentDefinition` 필드**: `description`(필수)·`prompt`(필수)·`tools`·`disallowedTools`·`model`·`skills`·`memory`·`mcpServers`·`maxTurns`·`background`·`effort`·`permissionMode`·`initialPrompt`.

---

## 10. 세션 관리

세션을 나열·조회·이름변경·태그할 수 있습니다.

```python
from claude_agent_sdk import (
    list_sessions, get_session_messages, get_session_info, rename_session, tag_session,
)

# 목록
for s in list_sessions(directory="/path/to/project", limit=10):
    print(f"{s.summary} ({s.session_id})")

# 메시지 조회
msgs = get_session_messages(s.session_id)

# 정보/이름변경/태그
info = get_session_info("550e8400-...")
rename_session(s.session_id, "Refactor auth module")
tag_session(s.session_id, "needs-review")
```

**세션 이어가기**는 `ClaudeAgentOptions`의 `continue_conversation=True`(최근 세션) 또는 `resume="<session_id>"`(특정 세션)로도 가능합니다.

---

## 11. 인증 (사내 LLM 포함)

Agent SDK는 Claude Code 엔진을 그대로 쓰므로 **CLI와 동일한 환경변수로 인증**합니다. Console API 키만 되는 게 아니라 **사내 게이트웨이·클라우드 프로바이더**도 지원합니다.

| 인증 방식 | 환경변수 |
| --- | --- |
| Anthropic Console | `ANTHROPIC_API_KEY` |
| **사내 LLM 게이트웨이** | `ANTHROPIC_BASE_URL` (+ `ANTHROPIC_AUTH_TOKEN`) |
| Amazon Bedrock | `CLAUDE_CODE_USE_BEDROCK=1` + AWS 자격증명 |
| Google Vertex/Agent Platform | `CLAUDE_CODE_USE_VERTEX=1` + GCP 자격증명 |
| Microsoft Foundry | Foundry 설정 |

```python
# 예: 사내 게이트웨이로 실행 (프로세스 환경에 설정)
#   export ANTHROPIC_BASE_URL="https://llm-gateway.mycompany.com"
#   export ANTHROPIC_AUTH_TOKEN="<사내 토큰>"
import asyncio
from claude_agent_sdk import query

async def main():
    async for msg in query(prompt="이 저장소의 버그를 찾아 고쳐줘"):
        print(msg)

asyncio.run(main())
```

> ⚠️ **금지되는 것은 claude.ai 개인 구독 로그인(Pro/Max OAuth)으로 만든 제품을 제3자에게 제공**하는 것뿐(사전 승인 필요)이며, 이는 약관 제한이지 기술 제한이 아닙니다.

---

> 본 문서는 공식 [Agent SDK Python 참조](https://code.claude.com/docs/en/agent-sdk/python)를 기반으로 정리했습니다. API는 버전에 따라 변할 수 있으니 최신 정보는 원문과 [GitHub 저장소](https://github.com/anthropics/claude-agent-sdk-python)를 확인하세요.
> TypeScript SDK는 [claude-agent-sdk-typescript](https://github.com/anthropics/claude-agent-sdk-typescript), 예제는 [claude-agent-sdk-demos](https://github.com/anthropics/claude-agent-sdk-demos) 참고.
