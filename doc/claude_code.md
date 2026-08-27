# Claude Code 정리 (한글 자료 모음)

> Anthropic이 만든 터미널 기반 AI 코딩 에이전트인 **Claude Code**에 대한 한글 인터넷 자료를 수집·정리한 문서입니다.
> 최종 정리일: 2026-08-27

---

## 1. Claude Code란?

**Claude Code**는 Anthropic이 만든 **터미널 기반 AI 코딩 에이전트**입니다. 자연어로 지시를 내리면 코드 작성·수정·테스트·리팩토링·커밋까지 프로젝트 전체를 직접 다뤄 줍니다. 단순한 코드 자동완성 도구가 아니라, 코드베이스를 스스로 탐색하고 파일을 읽어 컨텍스트를 이해한 뒤 작업을 수행하는 **에이전트(agent)** 라는 점이 핵심입니다.

- 터미널(CLI)뿐 아니라 **웹**([claude.ai/code](https://claude.ai/code)), **데스크톱 앱**, **VS Code**, **JetBrains IDE**, **Slack**, **GitHub Actions**, **GitLab CI/CD** 등 여러 인터페이스에서 사용할 수 있습니다.
- 사용하려면 Claude 구독(Pro, Max, Team, Enterprise) 또는 Claude Console(API 크레딧) 계정이 필요합니다.

---

## 2. 설치 및 시작하기

### 설치 (Native Install, 권장)

**macOS / Linux / WSL:**
```bash
curl -fsSL https://claude.ai/install.sh | bash
```

**Windows PowerShell:**
```powershell
irm https://claude.ai/install.ps1 | iex
```

이 외에도 Homebrew(`brew install --cask claude-code`), WinGet(`winget install Anthropic.ClaudeCode`), apt/dnf/apk 등 리눅스 패키지 매니저로도 설치할 수 있습니다.

> 💡 설치는 정상인데 `claude: command not found`가 나온다면, 설치 경로가 PATH에 등록되지 않은 경우가 많습니다.

### 로그인 및 첫 실행

```bash
# 프로젝트 디렉토리로 이동
cd /path/to/your/project

# Claude Code 실행 (첫 실행 시 브라우저 인증 진행)
claude
```

세션 안에서 `/login`으로 계정을 전환하거나 재인증할 수 있습니다.

### 기본 워크플로우

1. **코드베이스 이해** — "이 프로젝트는 무엇을 하나요?", "폴더 구조를 설명해주세요"
2. **코드 변경** — "주 파일에 hello world 함수 추가" (변경 전 항상 승인 요청)
3. **Git 작업** — "설명적인 메시지로 변경 사항 커밋", "새 브랜치 생성"
4. **버그 수정 / 기능 추가** — 자연어로 원하는 것을 설명
5. **리팩토링 · 테스트 작성 · 문서화 · 코드 리뷰**

---

## 3. 필수 명령어

### 셸 명령 (터미널에서 실행)

| 명령 | 기능 |
| --- | --- |
| `claude` | 대화형 모드 시작 |
| `claude "task"` | 일회성 작업 실행 |
| `claude -p "query"` | 일회성 쿼리 실행 후 종료 |
| `claude -c` | 현재 디렉토리에서 가장 최근 대화 계속 |
| `claude -r` | 이전 대화 재개 |

### 세션 명령 (Claude Code 실행 중 `/`로 시작)

| 명령 | 기능 |
| --- | --- |
| `/init` | 프로젝트 루트에 `CLAUDE.md` 자동 생성 |
| `/clear` | 대화 기록 지우기 |
| `/help` | 사용 가능한 명령 표시 |
| `/login` | 계정 로그인 / 전환 |
| `/resume` | 이전 대화 이어가기 |
| `/exit` (또는 Ctrl+D) | 종료 |

> 💡 단축키: `/` 입력으로 모든 명령·skills 보기, `Tab`으로 명령 완성, `↑`로 명령 기록, `Shift+Tab`으로 권한 모드 순환.

---

## 4. 핵심 개념

Claude Code의 확장 기능은 흔히 다음과 같이 비유됩니다.

| 구성 요소 | 한 줄 설명 | 비유 |
| --- | --- | --- |
| **CLAUDE.md** | 프로젝트 규칙·컨벤션·컨텍스트를 저장하는 메모리 파일 (Claude가 자동으로 읽음) | 신입에게 주는 업무 지침서 |
| **Commands (슬래시 커맨드)** | `/`로 시작하는 명령 | 자주 쓰는 단축 명령 |
| **Skills** | 필요할 때 참고하는 기술·업무 매뉴얼 | 업무 매뉴얼 |
| **Agents (서브에이전트)** | 특정 업무를 위임하는 별도의 에이전트 | 업무를 맡기는 팀원 |
| **Hooks** | 이벤트(코드 수정·명령 실행·작업 완료 등)에 반응하는 자동화 | 자동화 시스템 |
| **MCP** | 외부 도구·DB·API 연결 표준 프로토콜 | 외부 협력사 연결 |

### CLAUDE.md

`/init` 명령으로 프로젝트 루트에 생성되는 파일로, 프로젝트 구조·빌드 명령어·코딩 컨벤션 등을 담습니다. Claude가 세션마다 자동으로 읽어 프로젝트 규칙을 기억하게 만듭니다.

### Hooks (자동화)

Claude Code는 코드 수정·명령어 실행·작업 완료 같은 이벤트를 발생시킵니다. Hooks를 설정하면 이 이벤트에 반응해 사람 개입 없이 원하는 동작(예: 자동 포맷팅, 테스트 실행, 린트)을 수행합니다.

### MCP (Model Context Protocol)

AI 모델이 외부 도구·데이터베이스·API와 상호작용하기 위한 **오픈 소스 표준 프로토콜**입니다. Claude Code는 MCP 클라이언트로서 다양한 MCP 서버에 연결해 기능을 확장할 수 있습니다.

### 추천 학습 경로

① 구독으로 설치·첫 실행 → ② `CLAUDE.md`로 프로젝트 규칙 기억시키기 → ③ 슬래시 커맨드·Hooks로 반복 작업 자동화 → ④ MCP·서브에이전트로 외부 도구 연결과 분업까지 확장.

---

## 5. 한글 자료 링크 모음

### 공식 문서 (한국어)

- [빠른 시작 — Claude Code Docs](https://code.claude.com/docs/ko/quickstart) — 설치부터 첫 세션, 기본 워크플로우까지 공식 입문 가이드
- [MCP를 통해 Claude Code를 도구에 연결하기 — Claude Code Docs](https://code.claude.com/docs/ko/mcp) — MCP 서버 연결 공식 문서

### 입문 · 종합 가이드

- [Claude Code 완벽 가이드: 설치부터 실전 활용까지 한 번에 (2026) — 오픈위키](https://wikidocs.net/blog/@openwiki/21860/) — 설치·요금제·핵심 기능·실전 워크플로우 총정리
- [CC101 — Claude Code 한국어 입문 가이드](https://cc101.axwith.com/) — 한국어 입문자용 정리 사이트
- [Claude Code 완벽 가이드 (한글 요약본) — Velog(skysoo)](https://velog.io/@skysoo/Claude-Code-%EC%99%84%EB%B2%BD-%EA%B0%80%EC%9D%B4%EB%93%9C-%ED%95%9C%EA%B8%80-%EC%9A%94%EC%95%BD%EB%B3%B8) — 공식 완벽 가이드의 한글 요약
- [Claude Code 완벽 가이드 — Chaos and Order (youngju.dev)](https://www.youngju.dev/blog/llm/claude_code_complete_guide) — 개발 생산성 관점의 종합 정리

### 설치 · 환경 구축

- [Claude Code 설치 및 환경 구축하기 — 브런치](https://brunch.co.kr/@publichr/179) — VS Code 연동 등 환경 구축 중심
- [Claude Code 설치부터 사용까지: 바이브코딩 시작 가이드 — wookingwoo](https://blog.wookingwoo.com/83)
- [Claude Code 설치/사용법: Cursor·코파일럿 대신 써야 할 이유 — R.View](https://content.rview.com/ko/blog/claudecode/)

### 비개발자 · 실무 활용

- [2026 클로드 코드(Claude Code) 사용법: 설치부터 실전 활용까지 — High Output Club](https://blog.highoutputclub.com/2026-claude-code-for-non-developers/) — 비개발자용 실전 플레이북
- [Claude Code 사용법: 설치부터 Agent 활용 노하우까지 — 이랜서 블로그](https://www.elancer.co.kr/blog/detail/1012)
- [Claude Code 사용 가이드 — 하이퍼리즘 기술 블로그](https://tech.hyperithm.com/claude_code_guides)

### 핵심 기능 · 심화

- [Claude Code 실전 가이드 — 데이터웨이브 (Obsidian Publish)](https://publish.obsidian.md/datawave/%ED%81%B4%EB%A1%9C%EB%93%9C+%ED%81%B4%EB%9E%98%EC%8A%A4/Claude+Code+%EC%8B%A4%EC%A0%84+%EA%B0%80%EC%9D%B4%EB%93%9C)
- [Claude code — plugin, commands, skills, agents, hooks 에 대하여 — Velog(ghdtjrrl94)](https://velog.io/@ghdtjrrl94/Claude-code-plugin-commands-skills-agents-hooks-%EC%97%90-%EB%8C%80%ED%95%98%EC%97%AC) — 확장 기능 개념 정리
- [Claude Code Cheat Sheet 정리 — 단축키·Slash Command·MCP — Eottabom's Lab](https://eottabom.github.io/post/claude-code-cheat-sheet-guide/) — 자주 쓰는 항목 치트시트
- [claude-code-guide (GitHub) — tomtomjskim](https://github.com/tomtomjskim/claude-code-guide) — MCP 설정·개발 파이프라인·에이전트 페르소나·문서화 규칙 템플릿

### 관련 오픈소스 · 도구

- [Claude Code Router — 요청을 다양한 모델로 라우팅하는 오픈소스 — GeekNews](https://news.hada.io/topic?id=22288)

---

## 6. 초보자를 위한 팁

- **요청을 구체적으로** — "버그 수정" ❌ → "잘못된 자격 증명 입력 후 빈 화면이 나오는 로그인 버그 수정" ✅
- **복잡한 작업은 단계로 나누기** — 1) DB 테이블 생성 → 2) API 엔드포인트 → 3) 웹페이지 구축
- **먼저 탐색하게 하기** — 변경 전 "데이터베이스 스키마 분석"처럼 코드를 이해시키기
- **동료처럼 대화하기** — 달성하고 싶은 목표를 설명하면 Claude가 방법을 찾아 실행합니다.

---

> 본 문서의 링크·요약은 웹 검색 및 공식 문서를 기반으로 정리되었으며, 각 블로그의 상세 내용과 최신 정보는 원문 링크에서 확인하는 것을 권장합니다.
