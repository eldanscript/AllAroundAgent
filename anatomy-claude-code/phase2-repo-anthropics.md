# Phase 2: Anthropic 공식 리포지토리 (anthropics/claude-code) 분석

## 1. 리포지토리 개요 및 목적

이 리포지토리는 Anthropic이 공식적으로 관리하는 **Claude Code의 공개 리포지토리**이다. 실제 소스 코드(TypeScript/Node.js)는 포함되어 있지 **않으며**, 다음 요소들만 포함한다:

- **플러그인 시스템**: 14개의 공식 플러그인
- **설정 예제**: 조직 배포용 설정 프리셋 (lax, strict, sandbox)
- **Hook 예제**: Bash 명령어 검증 훅
- **GitHub 워크플로우**: 이슈 관리 자동화 (12개 워크플로우)
- **CHANGELOG**: 버전 0.2.21부터 2.1.87까지의 전체 릴리스 기록
- **DevContainer 설정**: 샌드박스 개발 환경

README.md의 핵심 설명:

> "Claude Code is an agentic coding tool that lives in your terminal, understands your codebase, and helps you code faster by executing routine tasks, explaining complex code, and handling git workflows -- all through natural language commands."

설치 방식은 npm에서 네이티브 바이너리로 전환되었다:

```bash
# 권장 (macOS/Linux)
curl -fsSL https://claude.ai/install.sh | bash
# Homebrew
brew install --cask claude-code
# npm (deprecated)
npm install -g @anthropic-ai/claude-code
```

---

## 2. 디렉토리 구조 전체

```
anthropics-claude-code/
├── .claude/
│   └── commands/                    # 리포 자체에서 사용하는 슬래시 커맨드
│       ├── commit-push-pr.md
│       ├── dedupe.md
│       └── triage-issue.md          # 이슈 트리아지 자동화 커맨드
├── .claude-plugin/
│   └── marketplace.json             # 마켓플레이스 매니페스트 (14개 플러그인 등록)
├── .devcontainer/
│   ├── devcontainer.json            # 샌드박스 개발 컨테이너 설정
│   ├── Dockerfile
│   └── init-firewall.sh             # 방화벽 초기화 (화이트리스트 기반)
├── .github/
│   ├── ISSUE_TEMPLATE/              # 이슈 템플릿 (bug, feature, documentation, model_behavior)
│   └── workflows/                   # 12개 GitHub Actions 워크플로우
│       ├── claude.yml               # @claude 멘션 시 Claude Code Action 실행
│       ├── claude-issue-triage.yml  # 이슈 자동 트리아지 (Opus 4.6)
│       ├── claude-dedupe-issues.yml # 중복 이슈 탐지 (Sonnet 4.5)
│       ├── auto-close-duplicates.yml # 중복 이슈 자동 닫기 (매일 9시)
│       ├── sweep.yml                # 이슈 라이프사이클 타임아웃 강제 (하루 2회)
│       ├── lock-closed-issues.yml   # 7일 비활성 이슈 잠금
│       ├── issue-lifecycle-comment.yml # 라벨 변경 시 코멘트
│       ├── issue-opened-dispatch.yml   # 새 이슈 외부 디스패치
│       ├── log-issue-events.yml     # Statsig에 이슈 이벤트 로깅
│       ├── non-write-users-check.yml # PR 보안 검토
│       ├── remove-autoclose-label.yml # 활동 시 autoclose 라벨 제거
│       └── backfill-duplicate-comments.yml # 과거 이슈 중복 코멘트 백필
├── examples/
│   ├── hooks/
│   │   └── bash_command_validator_example.py  # Hook 예제
│   └── settings/
│       ├── settings-lax.json        # 최소 보안 설정
│       ├── settings-strict.json     # 최대 보안 설정
│       └── settings-bash-sandbox.json # Bash 샌드박스 설정
├── plugins/                         # 14개 공식 플러그인
│   ├── agent-sdk-dev/
│   ├── claude-opus-4-5-migration/
│   ├── code-review/
│   ├── commit-commands/
│   ├── explanatory-output-style/
│   ├── feature-dev/
│   ├── frontend-design/
│   ├── hookify/
│   ├── learning-output-style/
│   ├── plugin-dev/
│   ├── pr-review-toolkit/
│   ├── ralph-wiggum/
│   ├── security-guidance/
│   └── README.md
├── scripts/                         # 워크플로우 지원 스크립트
│   ├── auto-close-duplicates.ts
│   ├── backfill-duplicate-comments.ts
│   ├── comment-on-duplicates.sh
│   ├── edit-issue-labels.sh
│   ├── gh.sh                        # gh CLI 래퍼
│   ├── issue-lifecycle.ts
│   ├── lifecycle-comment.ts
│   └── sweep.ts
├── Script/
│   └── run_devcontainer_claude_code.ps1  # Windows DevContainer 실행
├── CHANGELOG.md                     # 전체 릴리스 기록
├── LICENSE.md
├── README.md
└── SECURITY.md
```

---

## 3. 플러그인 시스템 구조 (14개 플러그인 각각의 역할)

### 표준 플러그인 구조

모든 플러그인은 다음 구조를 따른다:

```
plugin-name/
├── .claude-plugin/
│   └── plugin.json          # 메타데이터 (name, version, description, author)
├── commands/                # 슬래시 커맨드 (마크다운)
├── agents/                  # 전문 에이전트 (마크다운)
├── skills/                  # 스킬 정의 (SKILL.md)
├── hooks/                   # 이벤트 핸들러 (hooks.json + 스크립트)
└── README.md
```

마켓플레이스 매니페스트(`.claude-plugin/marketplace.json`)에서 14개 플러그인이 등록된다:

```json
{
  "$schema": "https://anthropic.com/claude-code/marketplace.schema.json",
  "name": "claude-code-plugins",
  "plugins": [...]
}
```

### 14개 플러그인 상세 분석

#### 3.1 agent-sdk-dev (카테고리: development)

**역할**: Claude Agent SDK 개발 킷. Agent SDK 프로젝트 생성과 검증을 지원한다.

- **커맨드**: `/new-sdk-app` - 새 Agent SDK 프로젝트 인터랙티브 설정
- **에이전트**: `agent-sdk-verifier-py`, `agent-sdk-verifier-ts` - Python/TypeScript SDK 앱 검증

#### 3.2 claude-opus-4-5-migration (카테고리: development)

**역할**: Sonnet 4.x / Opus 4.1에서 Opus 4.5로 코드 및 프롬프트 마이그레이션을 자동화한다.

- **스킬**: `claude-opus-4-5-migration` - 모델 문자열, 베타 헤더, 프롬프트 조정을 자동 수행
- **참조 자료**: `effort.md`, `prompt-snippets.md`
- **저자**: William Hu (whu@anthropic.com)

#### 3.3 code-review (카테고리: productivity)

**역할**: 5개의 병렬 Sonnet 에이전트를 사용한 자동 PR 코드 리뷰. 신뢰도 기반 점수로 false positive를 필터링한다.

- **커맨드**: `/code-review` - 자동 PR 리뷰 워크플로우
- **저자**: Boris Cherny (boris@anthropic.com)

#### 3.4 commit-commands (카테고리: productivity)

**역할**: Git 워크플로우 자동화.

- **커맨드**: `/commit` (커밋), `/commit-push-pr` (커밋+푸시+PR), `/clean_gone` (삭제된 원격 브랜치 정리)

#### 3.5 explanatory-output-style (카테고리: learning)

**역할**: 구현 선택과 코드베이스 패턴에 대한 교육적 인사이트를 추가한다 (deprecated된 Explanatory 출력 스타일 재현).

- **훅**: `SessionStart` - 세션 시작 시 교육적 컨텍스트를 주입
- **저자**: Dickson Tsai (dickson@anthropic.com)

#### 3.6 feature-dev (카테고리: development)

**역할**: 7단계 구조화된 기능 개발 워크플로우. 탐색 -> 설계 -> 구현 -> 리뷰의 전체 개발 사이클을 지원한다.

- **커맨드**: `/feature-dev` - 가이드 기능 개발 워크플로우
- **에이전트**: `code-explorer` (코드베이스 분석), `code-architect` (아키텍처 설계), `code-reviewer` (품질 리뷰)
- **저자**: Sid Bidasaria (sbidasaria@anthropic.com)

#### 3.7 frontend-design (카테고리: development)

**역할**: 제네릭 AI 미학을 피하고, 독특하고 프로덕션 수준의 프론트엔드 인터페이스를 생성한다.

- **스킬**: `frontend-design` - 프론트엔드 작업 시 자동 발동. 대담한 디자인 선택, 타이포그래피, 애니메이션 가이드 제공
- **저자**: Prithvi Rajasekaran & Alexander Bricken (prithvi@anthropic.com)

#### 3.8 hookify (카테고리: productivity)

**역할**: 대화 패턴 분석이나 명시적 지시를 통해 원치 않는 동작을 방지하는 커스텀 훅을 쉽게 생성한다. **가장 복잡한 플러그인**으로, Python 코어 엔진을 포함한다.

- **커맨드**: `/hookify`, `/hookify:list`, `/hookify:configure`, `/hookify:help`
- **에이전트**: `conversation-analyzer` - 대화에서 문제적 행동 분석
- **스킬**: `writing-rules` - hookify 규칙 문법 가이드
- **훅**: 4개 이벤트 감시 (`PreToolUse`, `PostToolUse`, `Stop`, `UserPromptSubmit`)
- **코어 Python 모듈**: `config_loader.py`, `rule_engine.py`, `matchers/`
- **저자**: Daisy Hollman (daisy@anthropic.com)

훅 설정 (`hooks.json`):

```json
{
  "hooks": {
    "PreToolUse": [{ "hooks": [{ "type": "command",
      "command": "python3 ${CLAUDE_PLUGIN_ROOT}/hooks/pretooluse.py", "timeout": 10 }] }],
    "PostToolUse": [...],
    "Stop": [...],
    "UserPromptSubmit": [...]
  }
}
```

#### 3.9 learning-output-style (카테고리: learning)

**역할**: 의사결정 포인트에서 사용자에게 의미 있는 코드 기여(5~10줄)를 요청하는 인터랙티브 학습 모드. 출시되지 않은 Learning 출력 스타일을 재현한다.

- **훅**: `SessionStart` - 학습 모드 지시사항을 `additionalContext`로 주입

세션 시작 핸들러에서 주입하는 핵심 철학:

> "Instead of implementing everything yourself, identify opportunities where the user can write 5-10 lines of meaningful code that shapes the solution."

#### 3.10 plugin-dev (카테고리: development)

**역할**: Claude Code 플러그인 개발을 위한 종합 툴킷. **가장 방대한 문서를 가진 플러그인**으로, 7개의 전문 스킬과 AI 지원 생성 기능을 포함한다.

- **커맨드**: `/plugin-dev:create-plugin` - 8단계 가이드 플러그인 빌드 워크플로우
- **에이전트**: `agent-creator`, `plugin-validator`, `skill-reviewer`
- **스킬 7개**:
  - `hook-development` - 훅 개발 (패턴, 마이그레이션, 린터 스크립트 포함)
  - `mcp-integration` - MCP 서버 통합 (stdio/http/sse 예제)
  - `command-development` - 커맨드 개발 (frontmatter, 인터랙티브, 테스팅)
  - `agent-development` - 에이전트 개발 (시스템 프롬프트 설계, 트리거링)
  - `plugin-structure` - 플러그인 구조 (매니페스트, 컴포넌트 패턴)
  - `plugin-settings` - 플러그인 설정 (파싱, frontmatter)
  - `skill-development` - 스킬 개발
- **저자**: Daisy Hollman (daisy@anthropic.com)

#### 3.11 pr-review-toolkit (카테고리: productivity)

**역할**: 6개의 전문 에이전트를 사용한 종합 PR 리뷰 도구.

- **커맨드**: `/pr-review-toolkit:review-pr` - 선택적 리뷰 관점(comments, tests, errors, types, code, simplify, all)
- **에이전트 6개**:
  - `comment-analyzer` - 코멘트 분석
  - `pr-test-analyzer` - 테스트 분석
  - `silent-failure-hunter` - 조용한 실패 탐지
  - `type-design-analyzer` - 타입 설계 분석
  - `code-reviewer` - 코드 리뷰
  - `code-simplifier` - 코드 단순화

#### 3.12 ralph-wiggum (카테고리: development)

**역할**: 자기 참조적 AI 루프를 통한 반복적 개발. Claude가 동일 작업을 완료될 때까지 반복적으로 수행한다. **Stop 훅을 사용하여 세션 종료를 가로채는** 독특한 패턴이다.

- **커맨드**: `/ralph-loop` (루프 시작), `/cancel-ralph` (루프 중단), `/help`
- **훅**: `Stop` - 종료 시도를 가로채 프롬프트를 다시 피드백
- **스크립트**: `setup-ralph-loop.sh`
- **저자**: Daisy Hollman (daisy@anthropic.com)

Stop 훅의 핵심 로직 (`stop-hook.sh`):

```bash
# 상태 파일에서 반복 횟수 추적
ITERATION=$(echo "$FRONTMATTER" | grep '^iteration:' | sed 's/iteration: *//')
MAX_ITERATIONS=$(echo "$FRONTMATTER" | grep '^max_iterations:' | sed 's/max_iterations: *//')

# 완료 약속(promise) 확인
if [[ -n "$PROMISE_TEXT" ]] && [[ "$PROMISE_TEXT" = "$COMPLETION_PROMISE" ]]; then
    echo "Ralph loop: Detected <promise>$COMPLETION_PROMISE</promise>"
    rm "$RALPH_STATE_FILE"
    exit 0
fi

# 종료를 차단하고 같은 프롬프트를 다시 피드
jq -n --arg prompt "$PROMPT_TEXT" --arg msg "$SYSTEM_MSG" \
  '{ "decision": "block", "reason": $prompt, "systemMessage": $msg }'
```

#### 3.13 security-guidance (카테고리: security)

**역할**: 파일 편집 시 보안 패턴을 검사하여 잠재적 보안 이슈를 경고한다.

- **훅**: `PreToolUse` - Edit/Write/MultiEdit 도구 사용 전 9개 보안 패턴 감시:
  1. `github_actions_workflow` - GitHub Actions 워크플로우 주입
  2. `child_process_exec` - 명령어 주입
  3. `new_function_injection` - Function 생성자 코드 주입
  4. `eval_injection` - eval() 코드 실행
  5. `react_dangerously_set_html` - XSS (React)
  6. `document_write_xss` - document.write XSS
  7. `innerHTML_xss` - innerHTML XSS
  8. `pickle_deserialization` - pickle 역직렬화
  9. `os_system_injection` - os.system 주입
- **저자**: David Dworken (dworken@anthropic.com)

세션별 상태 관리로 동일 경고를 반복하지 않으며, 30일 이상 된 상태 파일을 자동 정리한다.

#### 3.14 (리포 루트) .claude/commands/ - 리포 자체 운영 커맨드

리포 자체에서 사용하는 슬래시 커맨드 3개:

- `triage-issue.md` - 이슈 분석 및 라벨 적용 자동화
- `dedupe.md` - 중복 이슈 탐지
- `commit-push-pr.md` - 커밋/푸시/PR 생성

`triage-issue.md`의 설정 예시:

```yaml
---
allowed-tools: Bash(./scripts/gh.sh:*),Bash(./scripts/edit-issue-labels.sh:*)
description: Triage GitHub issues by analyzing and applying labels
---
```

---

## 4. 설정 예제 분석 (lax, strict, sandbox)

`examples/settings/` 디렉토리에 조직 배포용 3가지 설정 프리셋이 있다.

### 4.1 settings-lax.json (최소 보안)

```json
{
  "permissions": {
    "disableBypassPermissionsMode": "disable"
  },
  "strictKnownMarketplaces": []
}
```

- `--dangerously-skip-permissions` 비활성화
- 플러그인 마켓플레이스 차단 (빈 배열 = 모든 마켓플레이스 차단)

### 4.2 settings-strict.json (최대 보안)

```json
{
  "permissions": {
    "disableBypassPermissionsMode": "disable",
    "ask": ["Bash"],
    "deny": ["WebSearch", "WebFetch"]
  },
  "allowManagedPermissionRulesOnly": true,
  "allowManagedHooksOnly": true,
  "strictKnownMarketplaces": [],
  "sandbox": {
    "autoAllowBashIfSandboxed": false,
    "excludedCommands": [],
    "network": {
      "allowUnixSockets": [],
      "allowAllUnixSockets": false,
      "allowLocalBinding": false,
      "allowedDomains": [],
      "httpProxyPort": null,
      "socksProxyPort": null
    },
    "enableWeakerNestedSandbox": false
  }
}
```

핵심 정책:
- Bash 도구는 항상 사용자 승인 필요 (`ask`)
- 웹 검색/가져오기 완전 차단 (`deny`)
- 사용자/프로젝트 정의 권한 규칙 차단 (`allowManagedPermissionRulesOnly`)
- 사용자/프로젝트 정의 훅 차단 (`allowManagedHooksOnly`)
- 네트워크 완전 격리

### 4.3 settings-bash-sandbox.json (Bash 샌드박스)

```json
{
  "allowManagedPermissionRulesOnly": true,
  "sandbox": {
    "enabled": true,
    "autoAllowBashIfSandboxed": false,
    "allowUnsandboxedCommands": false,
    "excludedCommands": [],
    "network": { ... }
  }
}
```

- 샌드박스 명시적 활성화 (`enabled: true`)
- 샌드박스 외부 명령어 실행 불가 (`allowUnsandboxedCommands: false`)
- `sandbox` 속성은 Bash 도구에만 적용되며, Read/Write/WebSearch/MCP/훅에는 적용되지 않음

### 설정 계층 구조 참고

README에 명시된 중요한 사항:

> "Certain properties only take effect if specified in enterprise settings (e.g. `strictKnownMarketplaces`, `allowManagedHooksOnly`, `allowManagedPermissionRulesOnly`)."

v2.1.51부터 macOS plist 및 Windows Registry를 통한 관리 설정도 지원한다.

---

## 5. Hook 시스템 예제

### 5.1 Bash 명령어 검증 훅 예제

`examples/hooks/bash_command_validator_example.py`는 Hook 시스템의 대표적인 구현 예제이다.

**훅 이벤트**: `PreToolUse` (Bash 도구)

**설정 방법**:

```json
{
  "hooks": {
    "PreToolUse": [{
      "matcher": "Bash",
      "hooks": [{
        "type": "command",
        "command": "python3 /path/to/bash_command_validator_example.py"
      }]
    }]
  }
}
```

**검증 규칙**:

```python
_VALIDATION_RULES = [
    (r"^grep\b(?!.*\|)",
     "Use 'rg' (ripgrep) instead of 'grep' for better performance"),
    (r"^find\s+\S+\s+-name\b",
     "Use 'rg --files | rg pattern' instead of 'find -name'"),
]
```

**종료 코드 규약**:
- `exit(0)` - 도구 실행 허용
- `exit(1)` - stderr를 사용자에게 표시하되 Claude에게는 보이지 않음
- `exit(2)` - 도구 실행 차단, stderr를 Claude에게 표시

### 5.2 플러그인 내 훅 패턴들

리포에서 확인되는 훅 이벤트 종류:

| 훅 이벤트 | 사용 플러그인 | 용도 |
|-----------|-------------|------|
| `PreToolUse` | hookify, security-guidance | 도구 실행 전 검증/차단 |
| `PostToolUse` | hookify | 도구 실행 후 검증 |
| `Stop` | hookify, ralph-wiggum | 세션 종료 가로채기 |
| `UserPromptSubmit` | hookify | 사용자 입력 검증 |
| `SessionStart` | learning-output-style, explanatory-output-style | 세션 시작 시 컨텍스트 주입 |

### 5.3 CHANGELOG에서 확인되는 훅 이벤트 전체 목록

CHANGELOG 분석을 통해 파악된 전체 훅 이벤트:

- `PreToolUse` / `PostToolUse` - 도구 사용 전/후
- `Stop` / `SubagentStop` / `StopFailure` - 세션/에이전트 종료
- `SessionStart` / `SessionEnd` - 세션 시작/종료
- `UserPromptSubmit` - 사용자 입력 제출
- `PreCompact` / `PostCompact` - 컴팩션 전/후
- `Setup` - 리포 설정/유지보수
- `TaskCreated` / `TaskCompleted` / `TeammateIdle` - 태스크/팀원 이벤트
- `WorktreeCreate` / `WorktreeRemove` - 워크트리 생성/제거
- `CwdChanged` / `FileChanged` - 디렉토리/파일 변경
- `ConfigChange` - 설정 파일 변경
- `InstructionsLoaded` - CLAUDE.md/규칙 로드
- `Elicitation` / `ElicitationResult` - MCP 정보 요청

---

## 6. GitHub 워크플로우/자동화

이 리포는 **12개의 GitHub Actions 워크플로우**를 통해 대규모 오픈소스 이슈 관리를 자동화한다. 특히 **Claude Code 자체를 이슈 관리에 활용**하는 "dogfooding" 전략이 눈에 띈다.

### 6.1 Claude Code Action 기반 자동화

#### claude.yml - @claude 멘션 대응

```yaml
if: |
  (github.event_name == 'issue_comment' && contains(github.event.comment.body, '@claude')) ||
  (github.event_name == 'pull_request_review_comment' && contains(github.event.comment.body, '@claude'))
steps:
  - uses: anthropics/claude-code-action@v1
    with:
      anthropic_api_key: ${{ secrets.ANTHROPIC_API_KEY }}
      claude_args: "--model claude-sonnet-4-5-20250929"
```

이슈/PR에서 `@claude`를 멘션하면 Claude Code Action이 자동 실행된다.

#### claude-issue-triage.yml - 이슈 자동 트리아지

```yaml
- uses: anthropics/claude-code-action@v1
  with:
    prompt: "/triage-issue REPO: ${{ github.repository }} ISSUE_NUMBER: ..."
    claude_args: "--model claude-opus-4-6"
```

**Opus 4.6 모델**을 사용하여 `.claude/commands/triage-issue.md` 슬래시 커맨드를 실행한다. 이슈를 분석하고 라벨을 자동 적용하며, 댓글은 남기지 않는다.

#### claude-dedupe-issues.yml - 중복 이슈 탐지

```yaml
prompt: "/dedupe ${{ github.repository }}/issues/${{ github.event.issue.number }}"
claude_args: "--model claude-sonnet-4-5-20250929"
```

**Sonnet 4.5 모델**로 새 이슈의 중복 여부를 탐지하고, Statsig에 이벤트를 로깅한다.

### 6.2 스케줄 기반 자동화

| 워크플로우 | 스케줄 | 기능 |
|-----------|--------|------|
| `auto-close-duplicates.yml` | 매일 09:00 UTC | 중복 이슈 자동 닫기 (Bun + TypeScript) |
| `sweep.yml` | 매일 10:00, 22:00 UTC | 이슈 라이프사이클 타임아웃 강제 |
| `lock-closed-issues.yml` | 매일 14:00 UTC | 7일 비활성 닫힌 이슈 잠금 |

### 6.3 이벤트 기반 자동화

| 워크플로우 | 트리거 | 기능 |
|-----------|--------|------|
| `issue-lifecycle-comment.yml` | 라벨 변경 | 라이프사이클 코멘트 게시 |
| `issue-opened-dispatch.yml` | 이슈 생성 | 외부 리포에 디스패치 이벤트 전송 |
| `log-issue-events.yml` | 이슈 생성/닫기 | Statsig에 이벤트 로깅 |
| `non-write-users-check.yml` | PR (.github/** 변경) | `allowed_non_write_users` 보안 경고 |
| `remove-autoclose-label.yml` | 이슈 코멘트 | 새 코멘트 시 autoclose 라벨 제거 |
| `backfill-duplicate-comments.yml` | 수동 실행 | 과거 이슈 중복 코멘트 백필 |

### 6.4 DevContainer 샌드박스

`.devcontainer/` 설정으로 방화벽 격리된 개발 환경을 제공한다:

```bash
# init-firewall.sh - 화이트리스트 기반 방화벽
# 허용 도메인: GitHub, npmjs, api.anthropic.com, Sentry, Statsig, VS Code Marketplace
iptables -P OUTPUT DROP  # 기본: 모든 아웃바운드 차단
iptables -A OUTPUT -m set --match-set allowed-domains dst -j ACCEPT  # 화이트리스트만 허용
```

---

## 7. CHANGELOG에서 파악되는 버전 히스토리 주요 마일스톤

CHANGELOG는 **v0.2.21부터 v2.1.87까지** 약 200개 이상의 릴리스를 기록하고 있다.

### 초기 (v0.2.x) - 기반 기능 구축

| 버전 | 주요 기능 |
|------|---------|
| 0.2.21 | CHANGELOG 시작 (가장 오래된 기록) |
| 0.2.26 | `/approved-tools` 커맨드, Word-level diff |
| 0.2.31 | **커스텀 슬래시 커맨드** (`.claude/commands/`), MCP 디버그 모드 |
| 0.2.34 | Vim 바인딩 |
| 0.2.36 | Claude Desktop에서 MCP 서버 임포트 |
| 0.2.44 | **thinking 모드** ("think harder", "ultrathink") |
| 0.2.47 | Tab 자동완성, **자동 컴팩션** (무한 대화 길이) |
| 0.2.50 | MCP "project" 스코프 (`.mcp.json` 커밋 가능) |
| 0.2.53 | **WebFetch 도구** |
| 0.2.54 | MCP SSE 트랜스포트, 메모리 ('#' 단축키) |
| 0.2.59 | 이미지 복사/붙여넣기 |
| 0.2.75 | 메시지 큐잉, 이미지 드래그, @-멘션, MCP `--mcp-config` |
| 0.2.82 | 도구 이름 변경 (LSTool -> LS, View -> Read 등) |

### 1.0 시리즈 - 본격 프로덕션

| 버전 | 주요 기능 |
|------|---------|
| 1.0.33 | v1.0 시리즈 시작 (CHANGELOG 기준) |
| 1.0.38 | **Hooks 시스템 출시** |
| 1.0.44 | `/export` 커맨드, MCP resource_link |
| 1.0.48 | PreCompact 훅, Vim 모션 (c, f/F, t/T) |
| 1.0.51 | **네이티브 Windows 지원** |
| 1.0.54 | **UserPromptSubmit 훅** |
| 1.0.58 | **PDF 읽기 지원** |
| 1.0.60 | **커스텀 서브에이전트** (`/agents`) |
| 1.0.62 | SessionStart 훅, @에이전트 멘션 |
| 1.0.69 | **Opus 4.1로 업그레이드** |
| 1.0.72 | `/permissions` 도입 |
| 1.0.81 | **출력 스타일** 출시 (Explanatory, Learning) |

### 2.0 시리즈 - 메이저 업그레이드

| 버전 | 주요 기능 |
|------|---------|
| 2.0.60 | **백그라운드 에이전트** |
| 2.0.64 | 비동기 에이전트/bash, `/stats`, 이름 지정 세션, `.claude/rules/` |
| 2.0.67 | thinking 모드 기본 활성화 (Opus 4.5), **엔터프라이즈 관리 설정** |
| 2.0.72 | **Chrome 확장 연동** (Claude in Chrome Beta) |
| 2.0.74 | **LSP (Language Server Protocol) 도구** |

### 2.1 시리즈 - 플러그인/에이전트 팀/성능

| 버전 | 주요 기능 |
|------|---------|
| 2.1.0 | 스킬 핫 리로드, `context: fork`, 커스텀 키바인딩 |
| 2.1.3 | 슬래시 커맨드와 스킬 통합 |
| 2.1.15 | npm 설치 deprecated 알림 |
| 2.1.32 | **Claude Opus 4.6 출시**, **에이전트 팀** (멀티에이전트 협업), 자동 메모리 |
| 2.1.36 | **Fast 모드** (Opus 4.6) |
| 2.1.45 | **Claude Sonnet 4.6 지원** |
| 2.1.47 | ConfigChange 훅, 관리 설정 계층 강화 |
| 2.1.49 | **`--worktree` 플래그** (격리된 git worktree) |
| 2.1.59 | **자동 메모리** 저장/회상 |
| 2.1.63 | `/simplify`, `/batch`, HTTP 훅 |
| 2.1.68 | Opus 4.6 기본 medium effort, "ultrathink" 키워드 부활 |
| 2.1.69 | `/claude-api` 스킬, `/reload-plugins`, `InstructionsLoaded` 훅 |
| 2.1.71 | `/loop` 커맨드 (반복 실행) |
| 2.1.75 | **Opus 4.6 1M 컨텍스트 윈도우** (Max/Team/Enterprise) |
| 2.1.76 | **MCP Elicitation** 지원 |
| 2.1.77 | Opus 4.6 기본 최대 출력 64k 토큰 |
| 2.1.78 | `StopFailure` 훅, `${CLAUDE_PLUGIN_DATA}` 영속 데이터 |
| 2.1.80 | **채널** (연구 프리뷰), `source: 'settings'` 플러그인 |
| 2.1.81 | `--bare` 플래그 (최소 모드), `--channels` 권한 릴레이 |
| 2.1.83 | `managed-settings.d/` 드롭인 디렉토리, `CwdChanged`/`FileChanged` 훅 |
| 2.1.84 | **PowerShell 도구** (Windows opt-in), `TaskCreated` 훅 |
| 2.1.85 | 훅 조건부 `if` 필드, 플러그인 조직 정책 차단 |
| 2.1.87 | 최신 버전 (현재 CHANGELOG 기준) |

---

## 8. 공식 리포와 실제 코드베이스의 차이점

### 8.1 포함되지 않는 것

| 항목 | 공식 리포 | 실제 코드베이스 |
|------|---------|--------------|
| TypeScript/Node.js 소스 코드 | X | O (비공개) |
| 빌드 시스템 (esbuild/webpack) | X | O |
| 패키지 매니페스트 (package.json) | X | O |
| 테스트 코드 | X | O |
| CLI 엔트리포인트 | X | O |
| 도구 구현 (Bash, Read, Write, Edit 등) | X | O |
| API 클라이언트 / 스트리밍 로직 | X | O |
| 권한 시스템 구현 | X | O |
| TUI (Terminal UI) 렌더링 엔진 | X | O |
| VSCode 확장 소스 | X | O |

### 8.2 공식 리포의 실질적 역할

1. **공개 이슈 트래커**: 사용자 버그 리포트와 기능 요청을 수집하는 공식 채널
2. **릴리스 노트 저장소**: CHANGELOG.md에 모든 버전의 변경사항을 상세히 기록
3. **플러그인 마켓플레이스**: 14개 공식 플러그인을 마켓플레이스 소스로 제공
4. **설정 예제 배포**: 조직 배포를 위한 보안 설정 프리셋 제공
5. **자동화 데모**: Claude Code Action을 실제로 운용하는 dogfooding 사례

### 8.3 코드 분석으로 추론 가능한 내부 구조

CHANGELOG와 플러그인 문서에서 추론할 수 있는 실제 코드베이스의 구조:

- **도구 시스템**: Bash, Read, Write, Edit, MultiEdit, Glob, Grep, WebFetch, WebSearch, ToolSearch, Agent, Task, LSP 등
- **권한 시스템**: `allow`/`ask`/`deny` 3단계, 정규식 매칭 (`Bash(git *)`, `mcp__server__*`)
- **샌드박스**: Linux seatbelt + macOS App Sandbox, 파일시스템/네트워크 격리
- **MCP 통합**: stdio/HTTP/SSE 트랜스포트, OAuth 2.0, CIMD, Elicitation
- **메모리 시스템**: CLAUDE.md, `.claude/rules/`, 자동 메모리 (MEMORY.md)
- **에이전트 시스템**: 서브에이전트, 백그라운드 에이전트, 에이전트 팀(tmux 기반)
- **빌드 타겟**: npm 패키지, 네이티브 바이너리(macOS/Linux/Windows/ARM64), VSCode 확장

### 8.4 리포 운영에 사용되는 도구 스택

- **런타임**: Bun (스크립트 실행), Node.js
- **CI/CD**: GitHub Actions + `anthropics/claude-code-action@v1`
- **분석**: Statsig (이벤트 로깅, 기능 플래그)
- **모델**: Claude Opus 4.6 (트리아지), Sonnet 4.5 (중복 탐지, @claude 응답)

---

## 부록: 플러그인 저자 목록

| 저자 | 이메일 | 플러그인 |
|------|--------|---------|
| Daisy Hollman | daisy@anthropic.com | hookify, plugin-dev, ralph-wiggum |
| Boris Cherny | boris@anthropic.com | code-review, learning-output-style |
| William Hu | whu@anthropic.com | claude-opus-4-5-migration |
| Sid Bidasaria | sbidasaria@anthropic.com | feature-dev |
| Dickson Tsai | dickson@anthropic.com | explanatory-output-style |
| David Dworken | dworken@anthropic.com | security-guidance |
| Prithvi Rajasekaran & Alexander Bricken | prithvi@anthropic.com | frontend-design |
| Anthropic (팀) | support@anthropic.com | agent-sdk-dev, commit-commands, pr-review-toolkit |
