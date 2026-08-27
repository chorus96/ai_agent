# Claude Code CLI 레퍼런스 (명령어·플래그)

> Claude Code 명령줄 인터페이스(CLI)의 명령어와 플래그(인자)를 정리한 문서입니다.
> 최종 정리일: 2026-08-27 · 출처: [CLI 참조 — 공식 문서(한국어)](https://code.claude.com/docs/ko/cli-reference)

> ⚠️ 참고: `claude --help`는 **모든 플래그를 나열하지 않습니다.** `--help`에 없다고 사용할 수 없는 것은 아닙니다.
> 일부 플래그·명령어는 특정 버전(v2.1.x) 이상을 요구합니다.

## 목차

1. [기본 사용 형태](#1-기본-사용-형태)
   - [`claude -p "query"` 상세 (인쇄/헤드리스 모드)](#claude--p-query-상세-인쇄헤드리스-모드)
2. [CLI 명령어(하위 명령)](#2-cli-명령어하위-명령)
3. [CLI 플래그](#3-cli-플래그)
   - [세션 시작·재개](#세션-시작재개)
   - [모델·노력·폴백](#모델노력폴백)
   - [권한·도구](#권한도구)
   - [시스템 프롬프트](#시스템-프롬프트)
   - [MCP·플러그인·서브에이전트](#mcp플러그인서브에이전트)
   - [인쇄(-p) / 비대화형·스크립트](#인쇄-p--비대화형스크립트)
   - [디렉토리·설정](#디렉토리설정)
   - [백그라운드·원격·워크트리](#백그라운드원격워크트리)
   - [진단·기타](#진단기타)
4. [세션 내 슬래시 명령어 (Slash Commands)](#4-세션-내-슬래시-명령어-slash-commands)
5. [자주 쓰는 예시](#5-자주-쓰는-예시)

> 📚 관련 문서: 한글 자료·도서·GitHub 모음은 [`claude_code.md`](./claude_code.md), 심화 개념·팁(Plugin·MCP·Memory·Agent SDK·Subagents·Hooks·Skills·권한 모드·Plan Mode·Slash Commands 비교)은 [`claude_code_tips.md`](./claude_code_tips.md) 참고.

---

## 1. 기본 사용 형태

| 형태 | 설명 | 예시 |
| --- | --- | --- |
| `claude` | 대화형 세션 시작 | `claude` |
| `claude "query"` | 초기 프롬프트로 대화형 세션 시작 | `claude "explain this project"` |
| `claude -p "query"` | (인쇄 모드) SDK를 통해 쿼리하고 종료 | `claude -p "explain this function"` |
| `cat file \| claude -p "query"` | 파이프된 콘텐츠 처리 | `cat logs.txt \| claude -p "explain"` |
| `claude -c` | 현재 디렉토리의 가장 최근 대화 계속 | `claude -c` |
| `claude -c -p "query"` | SDK를 통해 이어서 실행 | `claude -c -p "Check for type errors"` |
| `claude -r "<session>" "query"` | ID 또는 이름으로 세션 재개 | `claude -r "auth-refactor" "Finish this PR"` |

> 하위 명령을 잘못 입력하면 가장 가까운 일치를 제안하고 세션을 시작하지 않습니다 (예: `claude udpate` → `Did you mean claude update?`).

### `claude -p "query"` 상세 (인쇄/헤드리스 모드)

`-p`(또는 `--print`)는 **대화형 UI 없이 한 번 실행하고 결과를 출력한 뒤 종료**하는 모드입니다. 스크립트·CI·파이프라인처럼 사람이 지켜보지 않는 자동화에 씁니다. 내부적으로는 **Agent SDK와 동일한 엔진**을 CLI로 구동하는 것입니다.

```bash
claude -p "auth 모듈이 하는 일 설명해줘"
```

#### 동작 방식·특징

- **일회성 실행**: 프롬프트를 처리해 응답을 stdout으로 출력하고 종료. 세션은 기본적으로 디스크에 저장되어 이어갈 수 있음(`-c`/`--resume`)
- **컨텍스트 로드**: 기본적으로 대화형 세션과 **동일한 컨텍스트**(CLAUDE.md·skills·hooks·MCP·자동 메모리)를 로드. CI에서 머신마다 동일한 결과가 필요하면 `--bare`로 이를 모두 건너뜀
- **모든 CLI 플래그가 `-p`와 함께 작동** (`--model`, `--allowedTools`, `--output-format`, `--append-system-prompt` 등)
- 일부 플래그는 **`-p` 전용**: `--max-turns`, `--max-budget-usd`, `--json-schema`, `--output-format`, `--input-format`, `--include-partial-messages` 등

#### stdin 파이프 입력

다른 CLI 도구처럼 데이터를 파이프하고 출력을 리디렉션할 수 있습니다.

```bash
# 빌드 로그를 넣어 원인 분석 후 파일로 저장
cat build-error.txt | claude -p '이 빌드 에러의 근본 원인을 간결히 설명' > output.txt

# git diff를 넣어 오타 린트
git diff main | claude -p "너는 오타 린터야. diff의 각 오타를 filename:line 형식으로 보고해"
```

> ⚠️ 파이프된 stdin은 **10MB 제한**(v2.1.128+). 초과 시 오류로 종료됩니다. 큰 입력은 파일로 쓰고 프롬프트에서 **파일 경로를 참조**하세요.

#### 출력 형식 (`--output-format`)

| 값 | 설명 | 주요 필드 |
| --- | --- | --- |
| `text` (기본) | 일반 텍스트 응답 | — |
| `json` | 결과·세션ID·메타데이터·비용을 담은 구조화 JSON | `result`, `session_id`, `total_cost_usd` |
| `stream-json` | 줄 구분 JSON 실시간 스트리밍 | 이벤트별 객체, 마지막 줄이 `result` |

```bash
# JSON 출력 후 jq로 텍스트만 추출
claude -p "이 프로젝트 요약" --output-format json | jq -r '.result'

# 스키마에 맞는 구조화 출력 (structured_output 필드에 담김)
claude -p "auth.py의 주요 함수명 추출" --output-format json \
  --json-schema '{"type":"object","properties":{"functions":{"type":"array","items":{"type":"string"}}},"required":["functions"]}' \
  | jq '.structured_output'

# 토큰 스트리밍 (verbose + partial 필요)
claude -p "재귀 설명" --output-format stream-json --verbose --include-partial-messages
```

> `--output-format json`을 쓰면 응답에 `total_cost_usd`(호출당 비용)가 포함되어 스크립트에서 지출 추적이 쉽습니다.

#### 도구·권한 제어

`-p`는 사람이 승인할 수 없으므로 **사전 승인**이 중요합니다.

```bash
# 특정 도구만 승인
claude -p "테스트 실행하고 실패 수정" --allowedTools "Bash,Read,Edit"

# 권한 규칙 구문 (접두사 일치, 공백 주의)
claude -p "스테이징 변경 검토 후 커밋" \
  --allowedTools "Bash(git diff *),Bash(git status *),Bash(git commit *)"

# 잠긴 CI: 사전 승인 외 전부 거부
claude -p "..." --permission-mode dontAsk

# 편집만 자동 승인
claude -p "린트 수정 적용" --permission-mode acceptEdits
```

#### 세션 이어가기

```bash
claude -p "이 코드베이스 성능 이슈 검토"
claude -p "이제 DB 쿼리에 집중" --continue          # 최근 대화 이어가기

# 세션 ID를 잡아 특정 대화 재개
sid=$(claude -p "리뷰 시작" --output-format json | jq -r '.session_id')
claude -p "그 리뷰 계속" --resume "$sid"
```

#### 실행 제한·종료

- `--max-turns N` — 에이전트 턴 수 제한 (도달 시 오류 종료)
- `--max-budget-usd N` — 최대 소비 달러 (초과 시 중지)
- **종료 코드**: 성공 0, 실패 시 0이 아닌 값 (스크립트에서 `$?`로 확인)
- Claude가 백그라운드 작업(dev 서버 등)을 띄우면 stdin 종료 후 ~5초 뒤 정리, 백그라운드 서브에이전트는 최대 10분 대기

#### `--bare` (권장, 스크립트용)

`--bare`는 hooks·skills·plugins·MCP·자동 메모리·CLAUDE.md 자동 검색을 **모두 건너뛰어** 빠르게 시작하고, 머신 간 동일한 결과를 보장합니다. 필요한 것만 플래그로 명시적으로 전달합니다.

```bash
claude --bare -p "이 파일 요약" --allowedTools "Read"
```

> `--bare`는 OAuth·키체인도 건너뛰므로 인증은 `ANTHROPIC_API_KEY`(또는 `--settings`의 `apiKeyHelper`, 클라우드 프로바이더 자격증명)로 제공해야 합니다. 향후 `-p`의 기본값이 될 예정입니다.

#### 슬래시 명령어와 `-p`

- 프롬프트 문자열에 `/skill-name`을 넣으면 실행 전 확장됨 (Skills·사용자 정의 명령 사용 가능)
- `/login`처럼 대화형 창을 여는 명령은 `-p`에서 사용 불가
- `/model sonnet`, `/effort high` 등 값 인자를 받는 명령은 사용 가능 (v2.1.205+)

---

## 2. CLI 명령어(하위 명령)

세션 시작 외에 관리·인증·MCP·백그라운드 세션 등을 다루는 하위 명령입니다.

### 업데이트·설치·인증

| 명령어 | 설명 |
| --- | --- |
| `claude update` | 최신 버전으로 업데이트 |
| `claude install [version]` | 네이티브 바이너리 설치/재설치 (`2.1.118`, `stable`, `latest`) |
| `claude auth login` | 로그인 (`--email`, `--sso`, `--console` 옵션) |
| `claude auth logout` | 로그아웃 |
| `claude auth status` | 인증 상태를 JSON으로 표시 (`--text`로 사람이 읽기 쉬운 출력) |
| `claude setup-token` | CI·스크립트용 장기 OAuth 토큰 생성 (출력만, 저장 안 함) |

### MCP·플러그인

| 명령어 | 설명 |
| --- | --- |
| `claude mcp` | MCP 서버 구성 (하위 명령 다수) |
| `claude mcp login <name>` | MCP 서버 OAuth 흐름 실행 (`--no-browser`로 URL 출력) |
| `claude mcp logout <name>` | MCP 서버 저장된 OAuth 자격증명 삭제 |
| `claude plugin` (별칭 `plugins`) | 플러그인 관리 (예: `claude plugin install <name>@<marketplace>`) |

### 백그라운드 세션·에이전트 뷰

| 명령어 | 설명 |
| --- | --- |
| `claude agents` | 에이전트 뷰 열기 (`--json`, `--cwd`, `--permission-mode`, `--model`, `--effort`, `--agent`) |
| `claude attach <id>` | 백그라운드 세션에 이 터미널로 연결 |
| `claude logs <id>` | 백그라운드 세션의 최근 출력 인쇄 |
| `claude respawn <id>` | 대화 유지하며 백그라운드 세션 재시작 (`--all`) |
| `claude stop <id>` | 백그라운드 세션 중지 (별칭 `claude kill`) |
| `claude rm <id>` | 목록에서 백그라운드 세션 제거 (대화 기록은 남음) |
| `claude daemon status` | 백그라운드 감독자(supervisor) 상태 인쇄 |
| `claude daemon stop --any` | 감독자 중지 (`--keep-workers`로 워커 유지) |

### 진단·유지보수·기타

| 명령어 | 설명 |
| --- | --- |
| `claude doctor` | 읽기 전용 설치·설정 진단 (세션 미시작) |
| `claude project purge [path]` | 프로젝트 로컬 상태 삭제 (`--dry-run`, `-y`, `-i`, `--all`) |
| `claude auto-mode defaults` | 자동 모드 분류기 기본 규칙을 JSON으로 인쇄 (`--label`) |
| `claude remote-control` | Remote Control 서버 시작 (`--name`) |
| `claude gateway --config <file>` | 자체 호스팅 Claude 앱 게이트웨이 서버 시작 (관리자용) |
| `claude ultrareview [target]` | ultrareview 비대화형 실행 (`--json`, `--timeout`) |

---

## 3. CLI 플래그

### 세션 시작·재개

| 플래그 | 설명 |
| --- | --- |
| `--print`, `-p` | 대화형 모드 없이 응답 인쇄 (스크립트·SDK 용) |
| `--continue`, `-c` | 현재 디렉토리의 가장 최근 대화 로드 |
| `--resume`, `-r` | ID·이름으로 세션 재개 (없으면 대화형 선택기) |
| `--fork-session` | 재개 시 원본 재사용 대신 새 세션 ID 생성 |
| `--session-id` | 특정 세션 ID 사용 (유효한 UUID) |
| `--name`, `-n` | 세션 표시 이름 설정 |
| `--from-pr` | 특정 PR에 연결된 세션 재개 (PR 번호·URL) |
| `--cloud` (구 `--remote`) | claude.ai에 새 웹 세션 생성 |
| `--teleport` | 로컬 터미널에서 웹 세션 재개 |

### 모델·노력·폴백

| 플래그 | 설명 |
| --- | --- |
| `--model` | 모델 설정 (`sonnet`,`opus`,`haiku`,`fable` 또는 전체 ID) |
| `--fallback-model` | 기본 모델 과부하·불가 시 폴백 (쉼표 구분 순차 시도) |
| `--effort` | 노력 수준 (`low`,`medium`,`high`,`xhigh`,`max`,`ultracode`) |
| `--advisor <model>` | 서버 측 advisor 도구 활성화 |
| `--betas` | API 베타 헤더 포함 (API 키 사용자) |

### 권한·도구

| 플래그 | 설명 |
| --- | --- |
| `--permission-mode` | 시작 권한 모드 (`default`/`acceptEdits`/`plan`/`auto`/`dontAsk`/`bypassPermissions`/`manual`) |
| `--dangerously-skip-permissions` | 권한 프롬프트 건너뜀 (= `--permission-mode bypassPermissions`) |
| `--allow-dangerously-skip-permissions` | `Shift+Tab` 순환에 `bypassPermissions` 추가 (즉시 활성화 아님) |
| `--allowedTools`, `--allowed-tools` | 승인 없이 실행할 도구 (예: `"Bash(git log *)" "Read"`) |
| `--disallowedTools`, `--disallowed-tools` | 거부 규칙 (`"Edit"`, `"*"`, `"mcp__*"`, `Bash(rm *)`) |
| `--tools` | 사용할 **기본 제공 도구** 제한 (`""`/`"default"`/`"Bash,Edit,Read"`) |
| `--permission-prompt-tool` | 비대화형 권한 프롬프트 처리 MCP 도구 지정 |

> 구분: `--allowedTools`/`--disallowedTools`는 **권한 규칙**, `--tools`는 **사용 가능한 내장 도구 집합** 제한.

### 시스템 프롬프트

4가지 모두 대화형·비대화형 모두 작동. `--system-prompt`와 `--system-prompt-file`은 상호 배타적이며, 추가(append) 플래그는 바꾸기(replace) 플래그와 결합 가능.

| 플래그 | 동작 |
| --- | --- |
| `--system-prompt` | 전체 기본 프롬프트를 **교체** |
| `--system-prompt-file` | 파일 내용으로 **교체** |
| `--append-system-prompt` | 기본 프롬프트에 **추가** |
| `--append-system-prompt-file` | 파일 내용을 기본 프롬프트에 **추가** |
| `--append-subagent-system-prompt` | 모든 서브에이전트 프롬프트 끝에 추가 (`-p` 전용) |

> 정체성이 코딩 어시스턴트로 유지되면 **append**, 완전히 다른 에이전트면 **replace**. replace는 안전 지침·도구 지침까지 제거되니 주의.

### MCP·플러그인·서브에이전트

| 플래그 | 설명 |
| --- | --- |
| `--mcp-config` | JSON 파일·문자열에서 MCP 서버 로드 (공백 구분) |
| `--strict-mcp-config` | `--mcp-config`의 서버만 사용, 나머지 MCP 구성 무시 |
| `--agent` | 현재 세션의 에이전트 지정 (`agent` 설정 재정의) |
| `--agents` | JSON으로 서브에이전트 동적 정의 |
| `--plugin-dir` | 디렉토리·`.zip`에서 플러그인 로드 (반복 가능) |
| `--plugin-url` | URL에서 플러그인 `.zip` 로드 |

### 인쇄(-p) / 비대화형·스크립트

대부분 **인쇄 모드(`-p`) 전용**입니다.

| 플래그 | 설명 |
| --- | --- |
| `--output-format` | 출력 형식 (`text`/`json`/`stream-json`) |
| `--input-format` | 입력 형식 (`text`/`stream-json`) |
| `--json-schema` | JSON Schema에 맞는 검증된 구조화 출력 |
| `--max-turns` | 에이전트 턴 수 제한 (도달 시 오류 종료) |
| `--max-budget-usd` | 최대 소비 달러 (초과 시 중지) |
| `--include-partial-messages` | 부분 스트리밍 이벤트 포함 (stream-json 필요) |
| `--include-hook-events` | 모든 hook 라이프사이클 이벤트 포함 (stream-json 필요) |
| `--replay-user-messages` | stdin 사용자 메시지를 stdout으로 재방출 |
| `--prompt-suggestions` | 턴마다 예측 다음 프롬프트 방출 |
| `--no-session-persistence` | 세션을 디스크에 저장하지 않음 (재개 불가) |
| `--exclude-dynamic-system-prompt-sections` | 머신별 섹션을 첫 메시지로 이동(캐시 재사용↑) |
| `--init` / `--maintenance` | Setup hooks를 해당 매처로 실행 후 시작 |
| `--init-only` | Setup·SessionStart hooks 실행 후 대화 없이 종료 |
| `--bare` | 최소 모드(hooks·skills·plugins·MCP·CLAUDE.md 건너뜀), 빠른 시작 |

### 디렉토리·설정

| 플래그 | 설명 |
| --- | --- |
| `--add-dir` | Claude가 읽고 편집할 추가 작업 디렉토리 (파일 접근만 부여) |
| `--settings` | 설정 JSON 파일·인라인 문자열 (동일 키 재정의) |
| `--setting-sources` | 로드할 설정 소스 (`user`,`project`,`local`) |
| `--safe-mode` | 모든 사용자 정의 비활성화하고 시작 (문제 해결용) |

### 백그라운드·원격·워크트리

| 플래그 | 설명 |
| --- | --- |
| `--bg`, `--background` | 백그라운드 에이전트로 시작하고 즉시 반환 (`-p`와 결합 불가) |
| `--exec` | Claude 세션 대신 셸 명령을 백그라운드 작업으로 실행 (`--bg`와) |
| `--remote-control`, `--rc` | Remote Control 활성화된 대화형 세션 시작 |
| `--worktree`, `-w` | 격리된 git worktree에서 시작 (`#<번호>`·PR URL 가능) |
| `--tmux` | worktree용 tmux 세션 생성 (`--worktree` 필요) |
| `--teammate-mode` | 에이전트 팀 팀원 표시 (`in-process`/`auto`/`tmux`/`iterm2`) |
| `--channels` | (연구 미리보기) 채널 알림 수신 MCP 서버 |

### 진단·기타

| 플래그 | 설명 |
| --- | --- |
| `--verbose` | 자세한 로깅·전체 턴별 출력 |
| `--debug` | 디버그 모드 (카테고리 필터: `"api,mcp"`, `"!statsig"`) |
| `--debug-file <path>` | 디버그 로그를 특정 파일에 기록 |
| `--version`, `-v` | 버전 출력 |
| `--ide` | 유효한 IDE 하나면 시작 시 자동 연결 |
| `--chrome` / `--no-chrome` | Chrome 브라우저 통합 활성화/비활성화 |
| `--disable-slash-commands` | 모든 skills·명령어 비활성화 |
| `--ax-screen-reader` | 스크린 리더 친화적 평문 출력 |

---

## 4. 세션 내 슬래시 명령어 (Slash Commands)

Claude Code 실행 중(대화형 세션) **`/`로 시작해 입력**하는 명령어입니다. 위의 셸 명령/플래그(`claude ...`)와 달리 세션 안에서 작동합니다.

> - **명령어는 메시지의 시작 부분에서만 인식**됩니다.
> - **[Skill]** 표시는 고정 로직이 아니라 프롬프트 기반 **번들 Skill**(Claude가 자동 호출 가능). 최대 6개까지 연결 가능: `/skill-a /skill-b do XYZ`
> - 사용자 정의 슬래시 명령어는 Skills로 만듭니다 → [`claude_code_tips.md` Skills 만들기](./claude_code_tips.md#skills-만들기) 참고

### 기본·세션 관리

| 명령어 | 인자 | 기능 |
| --- | --- | --- |
| `/help` | — | 도움말·명령어 목록 |
| `/status` | — | 버전·모델·계정·연결성 표시 |
| `/config` (별칭 `/settings`) | `[key=value]` | 테마·모델·output style 등 설정 |
| `/login` / `/logout` | — | 로그인 / 로그아웃 |
| `/exit` (별칭 `/quit`) | — | CLI 종료 (백그라운드 세션은 유지) |
| `/vim` | — | Vim 편집 모드 토글 |
| `/terminal-setup` | — | 터미널 개행 키 바인딩 설정 |

### 프로젝트·메모리

| 명령어 | 인자 | 기능 |
| --- | --- | --- |
| `/init` | — | `CLAUDE.md` 생성 (skills·hooks 설정 포함) |
| `/memory` | — | CLAUDE.md 편집·auto-memory 토글 |
| `/cd` | `<path>` | 작업 디렉토리 이동 (프롬프트 캐시 보존) |
| `/add-dir` | `<path>` | 파일 접근용 작업 디렉토리 추가 |

### 모델·처리

| 명령어 | 인자 | 기능 |
| --- | --- | --- |
| `/model` | `[model]` | 모델 전환 (없으면 선택기) |
| `/effort` | `[level\|auto]` | 노력 수준 (`low`~`max`, `ultracode`) |
| `/fast` | `[on\|off]` | fast mode 토글 |
| `/advisor` | `[model\|off]` | advisor 도구 토글 |

### 대화·컨텍스트

| 명령어 | 인자 | 기능 |
| --- | --- | --- |
| `/clear` (별칭 `/reset`,`/new`) | `[name]` | 빈 컨텍스트로 새 대화 |
| `/compact` | `[instructions]` | 대화 요약해 컨텍스트 확보 |
| `/context` | `[all]` | 컨텍스트 사용량 시각화 |
| `/btw` | `<question>` | 대화에 미포함 side question |
| `/branch` | `[name]` | 현재 대화의 브랜치 생성 |
| `/fork` | `<directive>` | 대화 상속 forked 서브에이전트 (백그라운드) |
| `/resume` (별칭 `/continue`) | `[session]` | 대화 재개 / 선택기 |
| `/rename` | `[name]` | 세션 이름 변경 |
| `/rewind` (별칭 `/checkpoint`,`/undo`) | — | 대화/코드 이전 지점으로 되감기 |
| `/plan` | `[description]` | 계획 모드 진입 |

### 코드 검토·보안 · Diff

| 명령어 | 인자 | 기능 |
| --- | --- | --- |
| `/code-review` **[Skill]** | `[level] [--fix] [--comment] [target]` | diff 정확성 버그·정리·효율성 검토 |
| `/simplify` **[Skill]** | `[target]` | 정리만 검토 (버그 제외) |
| `/security-review` | — | 보안 취약점 분석 |
| `/review` **[Skill]** | `[PR]` | PR 빠른 단일 패스 검토 |
| `/diff` | — | 커밋되지 않은 변경·턴별 diff 뷰어 |
| `/export` | `[filename]` | 대화를 평문으로 내보내기 |
| `/copy` | `[N]` | 마지막 응답 클립보드 복사 |

### 권한·도구·설정

| 명령어 | 인자 | 기능 |
| --- | --- | --- |
| `/permissions` (별칭 `/allowed-tools`) | — | 도구 권한 규칙(허용/요청/거부) |
| `/mcp` | `[reconnect\|enable\|disable]` | MCP 서버 연결·OAuth |
| `/hooks` | — | hook 구성 보기 |
| `/keybindings` | — | 키보드 단축키 파일 열기 |
| `/statusline` | — | 상태 표시줄 구성 |
| `/color` | `[color\|default]` | 프롬프트 바 색상 |

### 세션·백그라운드

| 명령어 | 인자 | 기능 |
| --- | --- | --- |
| `/background` (별칭 `/bg`) | `[prompt]` | 세션을 백그라운드 에이전트로 분리 |
| `/tasks` (별칭 `/bashes`) | — | 백그라운드 작업 보기·관리 |
| `/stop` | — | 현재 백그라운드 세션 중지 |
| `/batch` **[Skill]** | `<instruction>` | 대규모 변경 병렬 조율 |
| `/agents` | — | 서브에이전트 생성·관리 |

### 사용량·진단

| 명령어 | 인자 | 기능 |
| --- | --- | --- |
| `/usage` (별칭 `/cost`,`/stats`) | — | 토큰 사용량·비용 |
| `/doctor` **[Skill]** (별칭 `/checkup`) | — | 설정 검사·진단·수정 |
| `/debug` **[Skill]** | `[description]` | 디버그 로깅·문제 해결 |
| `/feedback` (별칭 `/bug`,`/share`) | `[report]` | 피드백·버그 보고 |
| `/heapdump` | — | JS 힙 스냅샷 (메모리 진단) |
| `/release-notes` | — | 변경 로그 보기 |

### 번들 Skill·Workflow

| 명령어 | 인자 | 기능 |
| --- | --- | --- |
| `/run` **[Skill]** | — | 앱 실행해 변경 검증 |
| `/verify` **[Skill]** | — | 앱 빌드·실행 검증 |
| `/run-skill-generator` **[Skill]** | — | `/run`·`/verify` 레시피 학습·저장 |
| `/loop` **[Skill]** (별칭 `/proactive`) | `[interval] [prompt]` | 프롬프트 반복 실행 |
| `/claude-api` **[Skill]** | `[migrate\|...]` | Claude API 참조 로드·업그레이드 |
| `/dataviz` **[Skill]** | `[request]` | 차트·대시보드 디자인 지침 |
| `/deep-research` **[Workflow]** | `<question>` | 웹 검색·교차 검증·보고서 종합 |
| `/fewer-permission-prompts` **[Skill]** | — | 허용 목록 추가로 프롬프트 감소 |

### 스킬·플러그인

| 명령어 | 인자 | 기능 |
| --- | --- | --- |
| `/skills` | — | skill 목록·가시성 제어 |
| `/reload-skills` | — | skill 디렉토리 재스캔 |
| `/plugin` | `[subcommand]` | 플러그인 관리(list/install/enable/disable) |
| `/reload-plugins` | `[--force]` | 활성 플러그인 다시 로드 |

### 원격·통합

| 명령어 | 인자 | 기능 |
| --- | --- | --- |
| `/remote-control` (별칭 `/rc`) | — | claude.ai에서 원격 제어 가능하게 |
| `/teleport` | — | 웹 세션을 터미널로 |
| `/desktop` (별칭 `/app`) | — | 데스크톱 앱에서 계속 |
| `/mobile` (별칭 `/ios`,`/android`) | — | 모바일 앱 QR |
| `/autofix-pr` | `[prompt]` | PR 자동 수정 세션 (CI·리뷰 감시) |
| `/install-github-app` | — | GitHub App 설치 |
| `/install-slack-app` | — | Slack 앱 설치 |
| `/ide` / `/chrome` | — | IDE 통합 / Chrome 설정 |

### 기타

| 명령어 | 인자 | 기능 |
| --- | --- | --- |
| `/goal` | `[condition\|clear]` | 목표 설정 (조건 충족까지 계속) |
| `/recap` | — | 세션 한 줄 요약 |
| `/insights` | — | 세션 분석 보고서 |
| `/schedule` (별칭 `/routines`) | `[description]` | 예약 작업(routines) 생성·관리 |
| `/privacy-settings` | — | 개인정보 설정 |
| `/focus` | — | 포커스 뷰 전환 |
| `/upgrade` | — | Pro/Max 업그레이드 |

---

## 5. 자주 쓰는 예시

```bash
# 일회성 질의 후 종료
claude -p "이 함수 설명해줘"

# 파이프 입력 처리
cat error.log | claude -p "이 에러 원인 분석"

# 계획 모드로 시작
claude --permission-mode plan "결제 모듈 리팩터링 계획 세워줘"

# 모델·노력 지정
claude --model claude-sonnet-5 --effort high

# 특정 도구만 허용하고 자동 승인 (CI)
claude -p --permission-mode dontAsk \
  --allowedTools "Bash(npm test)" "Read" "Grep" \
  "테스트 실행하고 실패만 요약"

# 구조화된 JSON 출력 (스크립트)
claude -p --output-format json --json-schema '{"type":"object","properties":{"summary":{"type":"string"}}}' \
  "변경사항 요약"

# 시스템 프롬프트에 규칙 추가
claude --append-system-prompt "항상 TypeScript로 답변"

# 격리된 worktree에서 PR 작업
claude -w feature-auth

# 백그라운드 에이전트로 실행 후 즉시 반환
claude --bg "flaky 테스트 원인 조사"

# 세션 재개
claude -r "auth-refactor" "이어서 PR 마무리해줘"

# 추가 디렉토리 접근 + 인라인 MCP 설정
claude --add-dir ../shared-lib --mcp-config ./mcp.json

# 손상된 설정 문제 해결
claude --safe-mode

# 빠른 스크립트 실행 (최소 모드)
claude --bare -p "이 파일 요약: $(cat notes.md)"
```

---

> 본 문서는 공식 [CLI 참조](https://code.claude.com/docs/ko/cli-reference)를 기반으로 정리했습니다. 플래그·명령어는 버전에 따라 추가·변경될 수 있으니 최신 정보는 원문과 `claude --help`를 확인하세요.
> 관련 문서: 심화 개념·팁은 [`claude_code_tips.md`](./claude_code_tips.md), 한글 자료·GitHub 모음은 [`claude_code.md`](./claude_code.md) 참고.
