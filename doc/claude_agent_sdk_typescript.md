# Claude Agent SDK — TypeScript 프로그래밍 가이드

> Claude Agent SDK(구 Claude Code SDK)로 **TypeScript/JavaScript에서 커스텀 AI 에이전트를 만드는 법**을 정리한 문서입니다.
> Claude Code를 라이브러리로 사용해 도구·에이전트 루프·컨텍스트 관리를 그대로 프로그래밍합니다.
> 최종 정리일: 2026-08-27 · 출처: [Agent SDK TypeScript 참조](https://code.claude.com/docs/en/agent-sdk/typescript) · [GitHub](https://github.com/anthropics/claude-agent-sdk-typescript)

> 📚 관련 문서: Python 버전은 [`claude_agent_sdk_python.md`](./claude_agent_sdk_python.md), 개념·인증·오픈소스 여부는 [`claude_code_tips.md`의 Agent SDK 섹션](./claude_code_tips.md#claude-agent-sdk), CLI 사용은 [`claude_code_cli.md`](./claude_code_cli.md) 참고.

## 목차

1. [설치](#1-설치)
2. [핵심 함수: query() · startup()](#2-핵심-함수-query--startup)
3. [Query 인터페이스 (런타임 제어)](#3-query-인터페이스-런타임-제어)
4. [Options (설정)](#4-options-설정)
5. [메시지·콘텐츠 타입](#5-메시지콘텐츠-타입)
6. [커스텀 도구 (tool · createSdkMcpServer)](#6-커스텀-도구-tool--createsdkmcpserver)
7. [권한 처리 (canUseTool)](#7-권한-처리-canusetool)
8. [스트리밍 입력](#8-스트리밍-입력)
9. [세션 관리](#9-세션-관리)
10. [Hooks](#10-hooks)
11. [인증 (사내 LLM 포함)](#11-인증-사내-llm-포함)
12. [Python SDK와의 매핑](#12-python-sdk와의-매핑)

---

## 1. 설치

### npm 설치

```bash
npm install @anthropic-ai/claude-agent-sdk
```

- **네이티브 Claude Code 바이너리를 번들** — 별도 설치 불필요. SDK 버전이 번들 Claude Code 버전을 추적(예: v0.3.191 → v2.1.191 번들)
- MIT 라이선스. Node.js/Bun/Deno 지원 (`executable` 옵션)
- 인증은 API 키·사내 게이트웨이·Bedrock/Vertex/Foundry 모두 가능([11장](#11-인증-사내-llm-포함))

### 저장소 clone 후 개발 (예제·소스 참조)

```bash
git clone https://github.com/anthropics/claude-agent-sdk-typescript.git
cd claude-agent-sdk-typescript
npm install
npm run build          # 빌드
```

---

## 2. 핵심 함수: query() · startup()

### query() — 기본 진입점

프롬프트를 주면 메시지를 **비동기 제너레이터**로 스트리밍합니다.

```typescript
function query({ prompt, options }: {
  prompt: string | AsyncIterable<SDKUserMessage>;
  options?: Options;
}): Query;
```

```typescript
import { query } from "@anthropic-ai/claude-agent-sdk";

for await (const message of query({
  prompt: "Hello, what files are here?",
  options: { maxTurns: 3 }
})) {
  console.log(message);
}
```

### startup() — CLI 서브프로세스 예열

프롬프트가 준비되기 전에 CLI를 미리 띄워 첫 응답 지연을 줄입니다.

```typescript
import { startup } from "@anthropic-ai/claude-agent-sdk";

const warm = await startup({ options: { maxTurns: 3 } });
for await (const message of warm.query("What files are here?")) {
  console.log(message);
}
```

> Python SDK가 `query()`(일회성)와 `ClaudeSDKClient`(다중 턴)로 나뉜 것과 달리, TS는 **`query()`가 반환하는 `Query` 객체**로 런타임 제어를 합니다.

---

## 3. Query 인터페이스 (런타임 제어)

`query()`가 반환하는 `Query`는 `AsyncGenerator`이면서 **세션을 실시간 제어**하는 메서드를 제공합니다.

| 메서드 | 설명 |
| --- | --- |
| `interrupt()` | 스트리밍 입력 모드에서 실행 중단 |
| `setPermissionMode(mode)` | 런타임에 권한 모드 변경 |
| `setModel(model?)` | 세션 중 모델 변경 |
| `setMaxThinkingTokens(n)` | 사고 토큰 예산 변경 |
| `streamInput(stream)` | 스트리밍 입력 모드에서 새 프롬프트 제공 |
| `rewindFiles(userMessageId, {dryRun})` | 특정 지점으로 파일 복원 (`enableFileCheckpointing: true` 필요) |
| `supportedModels()` / `supportedCommands()` / `supportedAgents()` | 사용 가능 모델·명령·에이전트 조회 |
| `mcpServerStatus()` / `reconnectMcpServer()` / `toggleMcpServer()` / `setMcpServers()` | MCP 서버 상태·제어 |
| `getContextUsage()` | 컨텍스트 사용량 조회 |
| `readFile(path, opts)` | 파일 읽기 |
| `stopTask(taskId)` / `close()` | 백그라운드 작업 중지 / 종료 |

```typescript
const q = query({ prompt: userMessages, options: { maxTurns: 5 } });
for await (const message of q) {
  if (message.type === 'assistant') {
    await q.setPermissionMode('plan');     // 런타임 제어
    await q.setModel('claude-sonnet-5');
  }
}
```

---

## 4. Options (설정)

`query()`의 동작을 제어하는 핵심 옵션들입니다. (일부만 발췌)

| 옵션 | 타입 | 설명 |
| --- | --- | --- |
| `systemPrompt` | `string` / preset 객체 | 시스템 프롬프트 (`{type:'preset',preset:'claude_code',append}`) |
| `tools` | `string[]` / preset | 도구 구성 |
| `allowedTools` / `disallowedTools` | `string[]` | 자동 승인 / 거부 도구 |
| `permissionMode` | `PermissionMode` | `default`/`plan`/`auto`/`bypassPermissions` 등 |
| `canUseTool` | `CanUseTool` | 도구 권한 콜백([7장](#7-권한-처리-canusetool)) |
| `model` / `fallbackModel` | `string` | 모델 / 폴백 모델 |
| `effort` | `low`~`max` | 추론 노력 수준 |
| `thinking` | `ThinkingConfig` | 확장 사고 |
| `mcpServers` | `Record<string, McpServerConfig>` | MCP 서버 |
| `agents` | `Record<string, AgentDefinition>` | 프로그래밍 서브에이전트 |
| `hooks` | `Partial<Record<HookEvent, ...>>` | Hook 구성 |
| `skills` | `string[]` / `'all'` | 사용 가능 Skills |
| `plugins` | `SdkPluginConfig[]` | 플러그인 |
| `maxTurns` / `maxBudgetUsd` | `number` | 최대 턴 / 최대 비용($) |
| `continue` / `resume` / `sessionId` / `forkSession` | | 세션 이어가기·재개·포크 |
| `outputFormat` | `{type:'json_schema', schema}` | 구조화 출력(JSON Schema) |
| `cwd` / `env` / `additionalDirectories` | | 작업 디렉토리 / 환경변수 / 추가 디렉토리 |
| `enableFileCheckpointing` | `boolean` | rewind용 파일 체크포인트 |
| `includePartialMessages` / `includeHookEvents` | `boolean` | 스트리밍/Hook 이벤트 포함 |
| `executable` | `'node'`/`'bun'`/`'deno'` | 실행 런타임 |
| `abortController` | `AbortController` | 취소 제어 |

**환경변수로 API 타임아웃 지정** (`env`로 전달):

```typescript
const result = query({
  prompt: "Analyze this code",
  options: {
    env: { ...process.env, API_TIMEOUT_MS: "120000", CLAUDE_CODE_MAX_RETRIES: "2" },
  },
});
```

---

## 5. 메시지·콘텐츠 타입

`query()`가 방출하는 `SDKMessage` 유니온을 `message.type`으로 분기해 처리합니다.

```typescript
type SDKMessage =
  | SDKAssistantMessage    // Claude의 텍스트/사고
  | SDKResultMessage       // 최종 결과
  | SDKToolUseMessage      // 도구 호출
  | SDKToolResultMessage   // 도구 실행 결과
  | SDKSystemMessage       // 시스템 정보·capabilities
  | SDKTaskProgressMessage // 백그라운드 작업 진행
  | SDKPartialMessage      // 부분 스트리밍
  | ...;
```

| 타입 | 주요 필드 |
| --- | --- |
| `SDKAssistantMessage` | `content: (SDKTextContent \| SDKThinkingContent)[]` |
| `SDKToolUseMessage` | `id`, `name`, `input` |
| `SDKToolResultMessage` | `tool_use_id`, `content`, `is_error?` |
| `SDKResultMessage` | `content`, `stop_reason`(`end_turn`/`max_tokens`/`stop_sequence`) |
| `SDKSystemMessage` | `capabilities`, `version` |
| `SDKTaskProgressMessage` | `task_id`, `status`(`queued`/`running`/`completed`/`failed`) |
| `SDKPartialMessage` | `delta: {type:'text_delta'\|'thinking_delta', text}` |

```typescript
for await (const message of query({ prompt: "..." })) {
  if (message.type === 'assistant') {
    for (const block of message.content) {
      if (block.type === 'text') console.log(block.text);
    }
  } else if (message.type === 'result') {
    console.log('종료:', message.stop_reason);
  }
}
```

콘텐츠 블록: `{type:'text', text}` · `{type:'image', source}` · `{type:'document', source}`

---

## 6. 커스텀 도구 (tool · createSdkMcpServer)

`tool()`로 **타입 안전한 도구**(Zod 스키마)를 정의하고, `createSdkMcpServer()`로 묶습니다.

```typescript
import { createSdkMcpServer, tool, query } from "@anthropic-ai/claude-agent-sdk";
import { z } from "zod";

const greet = tool(
  "greet",
  "Greet a person",
  { name: z.string() },                          // Zod 스키마
  async ({ name }) => ({
    content: [{ type: "text", text: `Hello, ${name}!` }],
  }),
  { annotations: { readOnlyHint: true, openWorldHint: true } }
);

const myServer = createSdkMcpServer({
  name: "my-tools",
  version: "1.0.0",
  instructions: "Use these tools to help the user",
  tools: [greet],
  alwaysLoad: true,
});

for await (const message of query({
  prompt: "Greet Alice",
  options: { mcpServers: { "my-tools": myServer } },
})) {
  console.log(message);
}
```

### tool() 시그니처

```typescript
function tool<Schema>(
  name: string,
  description: string,
  inputSchema: Schema,          // Zod raw shape ({ query: z.string() })
  handler: (args, extra) => Promise<CallToolResult>,
  extras?: { annotations?: ToolAnnotations; searchHint?: string; alwaysLoad?: boolean }
): SdkMcpToolDefinition;
```

- **반환**: `{ content: [{ type: "text", text: "..." }] }`
- **ToolAnnotations**: `readOnlyHint`(환경 변경 없음)·`destructiveHint`·`idempotentHint`·`openWorldHint`(외부 상호작용)·`title`
- **외부 MCP 서버**도 `mcpServers`에 stdio/SSE/HTTP config로 연결 가능

---

## 7. 권한 처리 (canUseTool)

도구 실행 전에 **직접 승인/거부**하는 콜백을 등록합니다.

```typescript
import { query } from "@anthropic-ai/claude-agent-sdk";

for await (const message of query({
  prompt: "Delete all files",
  options: {
    permissionMode: 'default',
    allowedTools: ['Bash'],
    canUseTool: async ({ toolName, toolInput, requestId }, { signal }) => {
      if (toolName === 'Bash' && (toolInput as any).command?.includes('rm')) {
        return { approved: false };            // 거부
      }
      return { approved: true };               // 승인
    },
  }
})) {
  console.log(message);
}
```

- 반환: `{ approved: boolean, toolAlias?: string }`
- **PermissionMode**: `'default'`(프롬프트) · `'bypassPermissions'`(건너뜀) · `'plan'`(읽기 전용 계획) · `'auto'`(안전 작업 자동 승인)
- `q.setPermissionMode(...)`로 런타임 변경 가능

---

## 8. 스트리밍 입력

프롬프트를 **async 제너레이터**로 넘겨 메시지를 동적으로 흘려보냅니다. (인터럽트·런타임 제어는 스트리밍 입력 모드에서 동작)

```typescript
import { query } from "@anthropic-ai/claude-agent-sdk";

async function* userMessages() {
  yield { type: 'text' as const, text: 'Start task' };
  await new Promise(r => setTimeout(r, 1000));
  yield { type: 'text' as const, text: 'Continue with more info' };
}

const q = query({ prompt: userMessages(), options: { maxTurns: 5 } });

for await (const message of q) {
  if (message.type === 'assistant') {
    // 스트림 중 제어
    await q.setPermissionMode('plan');
    await q.setModel('claude-sonnet-5');
  }
}

// 중단
await q.interrupt();
```

---

## 9. 세션 관리

세션을 나열·조회·이름변경·태그할 수 있습니다.

```typescript
import { listSessions, getSessionMessages, renameSession, tagSession } from "@anthropic-ai/claude-agent-sdk";

// 목록
const sessions = await listSessions({ dir: "/path/to/project", limit: 10 });
for (const s of sessions) console.log(`${s.summary} (${s.sessionId})`);

// 메시지 조회
const [latest] = await listSessions({ dir: "/path/to/project", limit: 1 });
if (latest) {
  const messages = await getSessionMessages(latest.sessionId, { limit: 20 });
  for (const m of messages) console.log(`[${m.type}] ${m.uuid}`);
}

// 이름변경 / 태그
await renameSession(latest.sessionId, "Refactor auth module");
await tagSession(latest.sessionId, "needs-review");
```

`SDKSessionInfo`: `sessionId`, `summary`, `lastModified`, `customTitle`, `firstPrompt`, `gitBranch`, `cwd`, `tag`, `createdAt`.

**세션 이어가기**는 Options의 `continue: true`(최근) 또는 `resume: "<sessionId>"`(특정), `forkSession: true`(포크)로도 가능합니다.

---

## 10. Hooks

라이프사이클 이벤트에 콜백을 등록합니다.

```typescript
const q = query({
  prompt: "...",
  options: {
    hooks: {
      SessionStart: [{ hooks: [async (input) => { /* ... */ return {}; }] }],
    },
    includeHookEvents: true,   // 스트림에 hook 메시지 포함
  }
});
```

- `HookEvent`: `SessionStart` · `Setup` · `SessionEnd` · `Notification` · `PreCompact` · `PostCompact` 등
- Hook 메시지: `hook_started` / `hook_progress` / `hook_response`

> ⚠️ SDK의 Hooks는 콜백(함수)으로 등록합니다. CLI의 `settings.json` 셸 명령 Hooks와는 등록 방식이 다릅니다.

### 기타 유틸

- **`resolveSettings()`** — 세션을 띄우지 않고 유효 설정과 출처(provenance) 조회
- **AgentDefinition** — `agents` 옵션으로 프로그래밍 서브에이전트 정의(description·prompt·tools·model 등)
- **Bun 컴파일** — `bun build --compile` 시 플랫폼 패키지(`@anthropic-ai/claude-agent-sdk-<platform>/claude`)와 `extractFromBunfs`, `pathToClaudeCodeExecutable` 사용

---

## 11. 인증 (사내 LLM 포함)

TS SDK도 Claude Code 엔진을 그대로 쓰므로 **CLI와 동일한 환경변수로 인증**합니다. Console API 키만 되는 게 아니라 **사내 게이트웨이·클라우드 프로바이더**도 지원합니다.

| 인증 방식 | 환경변수 |
| --- | --- |
| Anthropic Console | `ANTHROPIC_API_KEY` |
| **사내 LLM 게이트웨이** | `ANTHROPIC_BASE_URL` (+ `ANTHROPIC_AUTH_TOKEN`) |
| Amazon Bedrock | `CLAUDE_CODE_USE_BEDROCK=1` + AWS 자격증명 |
| Google Vertex/Agent Platform | `CLAUDE_CODE_USE_VERTEX=1` + GCP 자격증명 |
| Microsoft Foundry | Foundry 설정 |

```typescript
// 예: 사내 게이트웨이로 실행 (Options.env 또는 프로세스 환경)
const q = query({
  prompt: "이 저장소의 버그를 찾아 고쳐줘",
  options: {
    env: {
      ...process.env,
      ANTHROPIC_BASE_URL: "https://llm-gateway.mycompany.com",
      ANTHROPIC_AUTH_TOKEN: "<사내 토큰>",
    },
  },
});
for await (const m of q) console.log(m);
```

> ⚠️ **금지되는 것은 claude.ai 개인 구독 로그인(Pro/Max OAuth)으로 만든 제품을 제3자에게 제공**하는 것뿐(사전 승인 필요)이며, 약관 제한이지 기술 제한이 아닙니다.

---

## 12. Python SDK와의 매핑

| 개념 | Python | TypeScript |
| --- | --- | --- |
| 일회성/기본 | `query()` | `query()` |
| 다중 턴·제어 | `ClaudeSDKClient` (별도 클래스) | `query()`가 반환하는 **`Query` 객체** 메서드 |
| 설정 | `ClaudeAgentOptions` | `Options` |
| 커스텀 도구 | `@tool` 데코레이터 | `tool()` 함수 (+ Zod 스키마) |
| MCP 서버 | `create_sdk_mcp_server` | `createSdkMcpServer` |
| 권한 콜백 | `can_use_tool` (Allow/Deny 객체) | `canUseTool` (`{approved}` 반환) |
| 인터럽트 | `client.interrupt()` | `q.interrupt()` |
| 세션 | `list_sessions` 등 | `listSessions` 등 (camelCase) |
| 비동기 | `asyncio`/`anyio` | `for await ... of` (async generator) |

> 큰 그림은 동일하고, **네이밍 컨벤션(snake_case ↔ camelCase)** 과 다중 턴 제어 방식(별도 클래스 ↔ `Query` 객체)만 다릅니다.

---

> 본 문서는 공식 [Agent SDK TypeScript 참조](https://code.claude.com/docs/en/agent-sdk/typescript)를 기반으로 정리했습니다. API는 버전에 따라 변할 수 있으니 최신 정보는 원문과 [GitHub 저장소](https://github.com/anthropics/claude-agent-sdk-typescript)를 확인하세요.
> Python SDK는 [`claude_agent_sdk_python.md`](./claude_agent_sdk_python.md), 예제는 [claude-agent-sdk-demos](https://github.com/anthropics/claude-agent-sdk-demos) 참고.
