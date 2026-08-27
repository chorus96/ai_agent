# Claude Code CLI 레퍼런스 (명령어·플래그)

> Claude Code 명령줄 인터페이스(CLI)의 명령어와 플래그(인자)를 정리한 문서입니다.
> 최종 정리일: 2026-08-27 · 출처: [CLI 참조 — 공식 문서(한국어)](https://code.claude.com/docs/ko/cli-reference)

> ⚠️ 참고: `claude --help`는 **모든 플래그를 나열하지 않습니다.** `--help`에 없다고 사용할 수 없는 것은 아닙니다.
> 일부 플래그·명령어는 특정 버전(v2.1.x) 이상을 요구합니다.

## 목차

1. [기본 사용 형태](#1-기본-사용-형태)
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
4. [자주 쓰는 예시](#4-자주-쓰는-예시)

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

## 4. 자주 쓰는 예시

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
