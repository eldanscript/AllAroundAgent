# Phase 2: hangsman-claude-code-source 리포지토리 상세 분석

> 분석 대상: `refs/hangsman-claude-code-source/`  
> 패키지: `@anthropic-ai/claude-code` v2.1.88  
> 분석 일시: 2026-03-31

---

## 1. 리포지토리 개요 및 복원 방법

### 1.1 복원 원리

이 리포지토리는 npm에 배포된 `@anthropic-ai/claude-code` 패키지의 **소스맵(source map)**에서 원본 TypeScript 소스를 복원한 것이다. 배포된 `cli.js`는 단일 minified 번들이며, 동봉된 `cli.js.map` 파일에 원본 파일 경로와 소스 코드가 인코딩되어 있다.

```
cli.js        → 단일 minified 번들 (Bun 번들러 출력)
cli.js.map    → 소스맵 (원본 TypeScript 소스 포함)
src/          → 소스맵에서 추출한 원본 소스 트리
```

### 1.2 cli.js 헤더

```javascript
#!/usr/bin/env node
// (c) Anthropic PBC. All rights reserved.
// Version: 2.1.88
// Want to see the unminified source? We're hiring!
// https://job-boards.greenhouse.io/anthropic/jobs/4816199008
```

### 1.3 패키지 메타데이터

```json
{
  "name": "@anthropic-ai/claude-code",
  "version": "2.1.88",
  "bin": { "claude": "cli.js" },
  "engines": { "node": ">=18.0.0" },
  "type": "module"
}
```

빌드 도구는 **Bun 번들러**이다. `feature()` 호출은 `bun:bundle` 모듈에서 임포트되며, 빌드 시 dead code elimination에 사용된다.

### 1.4 파일 통계

- **src/ 하위 총 파일 수**: 약 1,902개
- **총 코드 라인**: 약 512,664줄
- **최대 파일**: `src/cli/print.ts` (5,594줄)

---

## 2. 전체 디렉토리 구조

### 2.1 최상위 구조

```
refs/hangsman-claude-code-source/
├── cli.js              # minified 번들 진입점
├── cli.js.map          # 소스맵
├── package.json
├── README.md
├── LICENSE.md
├── sdk-tools.d.ts      # SDK 도구 타입 정의
├── bun.lock
├── node_modules/       # 런타임 의존성
├── vendor/             # 벤더 코드
└── src/                # 복원된 소스 트리
```

### 2.2 src/ 하위 전체 디렉토리 맵

```
src/
├── assistant/          # 어시스턴트 모드 (Kairos)
├── bootstrap/          # 부트스트랩 상태 관리 (sessionId, cwd, 비용 추적 등)
├── bridge/             # 원격 Bridge 시스템 (31개 파일)
├── buddy/              # 컴패니언 캐릭터 시스템
├── cli/                # CLI 출력/입력 처리
│   ├── handlers/
│   └── transports/
├── commands/           # 슬래시 명령어 (80+ 디렉토리)
│   ├── add-dir/        ├── agents/        ├── ant-trace/
│   ├── autofix-pr/     ├── backfill-sessions/  ├── branch/
│   ├── break-cache/    ├── bridge/        ├── btw/
│   ├── bughunter/      ├── chrome/        ├── clear/
│   ├── color/          ├── compact/       ├── config/
│   ├── context/        ├── copy/          ├── cost/
│   ├── ctx_viz/        ├── debug-tool-call/ ├── desktop/
│   ├── diff/           ├── doctor/        ├── effort/
│   ├── env/            ├── exit/          ├── export/
│   ├── extra-usage/    ├── fast/          ├── feedback/
│   ├── files/          ├── good-claude/   ├── heapdump/
│   ├── help/           ├── hooks/         ├── ide/
│   ├── install-github-app/ ├── install-slack-app/ ├── issue/
│   ├── keybindings/    ├── login/         ├── logout/
│   ├── mcp/            ├── memory/        ├── mobile/
│   ├── mock-limits/    ├── model/         ├── oauth-refresh/
│   ├── onboarding/     ├── output-style/  ├── passes/
│   ├── perf-issue/     ├── permissions/   ├── plan/
│   ├── plugin/         ├── pr_comments/   ├── privacy-settings/
│   ├── rate-limit-options/ ├── release-notes/ ├── reload-plugins/
│   ├── remote-env/     ├── remote-setup/  ├── rename/
│   ├── reset-limits/   ├── resume/        ├── review/
│   ├── rewind/         ├── sandbox-toggle/ ├── session/
│   ├── share/          ├── skills/        ├── stats/
│   ├── status/         ├── stickers/      ├── summary/
│   ├── tag/            ├── tasks/         ├── teleport/
│   ├── terminalSetup/  ├── theme/         ├── thinkback/
│   ├── thinkback-play/ ├── upgrade/       ├── usage/
│   ├── vim/            └── voice/
├── components/         # React (Ink) UI 컴포넌트
│   ├── agents/         ├── ClaudeCodeHint/ ├── CustomSelect/
│   ├── design-system/  ├── DesktopUpsell/ ├── diff/
│   ├── FeedbackSurvey/ ├── grove/         ├── HelpV2/
│   ├── HighlightedCode/ ├── hooks/        ├── LogoV2/
│   ├── LspRecommendation/ ├── ManagedSettingsSecurityDialog/
│   ├── mcp/            ├── memory/        ├── messages/
│   ├── Passes/         ├── permissions/   ├── PromptInput/
│   ├── sandbox/        ├── Settings/      ├── shell/
│   ├── skills/         ├── Spinner/       ├── StructuredDiff/
│   ├── tasks/          ├── teams/         ├── TrustDialog/
│   ├── ui/             └── wizard/
├── constants/          # 시스템 상수, 프롬프트, 도구 목록
├── context/            # React Context (알림, 보이스 등)
├── coordinator/        # 코디네이터 모드 (멀티 워커)
├── entrypoints/        # 진입점 정의
│   └── sdk/            # SDK 스키마/타입
├── hooks/              # React Hooks
│   ├── notifs/
│   └── toolPermission/
│       └── handlers/
├── ink/                # 커스텀 Ink 렌더링 엔진
│   ├── components/     ├── events/        ├── hooks/
│   ├── layout/         └── termio/
├── keybindings/        # 키 바인딩 시스템
├── memdir/             # 메모리 디렉토리 (CLAUDE.md 등)
├── migrations/         # 설정 마이그레이션
├── moreright/          # 추가 권한 시스템
├── native-ts/          # 네이티브 TypeScript 구현
│   ├── color-diff/     ├── file-index/    └── yoga-layout/
├── outputStyles/       # 출력 스타일 처리
├── plugins/            # 플러그인 시스템
│   └── bundled/
├── query/              # 쿼리 처리 (stopHooks 등)
├── remote/             # 원격 접속 관련
├── schemas/            # JSON 스키마 정의
├── screens/            # 메인 화면 (REPL.tsx, ResumeConversation.tsx)
├── server/             # MCP 서버 모드
├── services/           # 핵심 서비스 레이어
│   ├── AgentSummary/   ├── analytics/     ├── api/
│   ├── autoDream/      ├── compact/       ├── extractMemories/
│   ├── lsp/            ├── MagicDocs/     ├── mcp/
│   ├── oauth/          ├── plugins/       ├── policyLimits/
│   ├── PromptSuggestion/ ├── remoteManagedSettings/
│   ├── SessionMemory/  ├── settingsSync/  ├── teamMemorySync/
│   ├── tips/           ├── tools/         └── toolUseSummary/
├── skills/             # 스킬 시스템
│   └── bundled/
├── state/              # 앱 상태 관리 (AppState, AppStateStore)
├── tasks/              # 백그라운드 태스크 시스템
│   ├── DreamTask/      ├── InProcessTeammateTask/
│   ├── LocalAgentTask/ ├── LocalShellTask/
│   └── RemoteAgentTask/
├── tools/              # 40+ 도구 구현
├── types/              # 타입 정의
│   └── generated/      # protobuf 생성 타입
├── upstreamproxy/      # 업스트림 프록시
├── utils/              # 유틸리티 (60+ 서브디렉토리/파일)
│   ├── background/     ├── bash/          ├── claudeInChrome/
│   ├── computerUse/    ├── deepLink/      ├── dxt/
│   ├── filePersistence/ ├── git/          ├── github/
│   ├── hooks/          ├── mcp/           ├── memory/
│   ├── messages/       ├── model/         ├── nativeInstaller/
│   ├── permissions/    ├── plugins/       ├── powershell/
│   ├── processUserInput/ ├── sandbox/     ├── secureStorage/
│   ├── settings/       ├── shell/         ├── skills/
│   ├── suggestions/    ├── swarm/         ├── task/
│   ├── telemetry/      ├── teleport/      ├── todo/
│   └── ultraplan/
├── vim/                # Vim 모드 통합
└── voice/              # 보이스 모드 (voiceModeEnabled.ts)
```

---

## 3. 핵심 파일 크기 및 복잡도 분석 (상위 20개)

| 순위 | 파일 경로 | 줄 수 | 역할 |
|------|----------|-------|------|
| 1 | `src/cli/print.ts` | 5,594 | CLI 출력 렌더링 (전체 메시지 포맷팅) |
| 2 | `src/utils/messages.ts` | 5,512 | 메시지 정규화/변환 |
| 3 | `src/utils/sessionStorage.ts` | 5,105 | 세션 저장/복원 로직 |
| 4 | `src/utils/hooks.ts` | 5,022 | Hook 시스템 (PreToolUse, PostToolUse 등) |
| 5 | `src/screens/REPL.tsx` | 5,005 | 메인 REPL 화면 (React 컴포넌트) |
| 6 | `src/main.tsx` | 4,683 | 애플리케이션 진입/초기화 로직 |
| 7 | `src/utils/bash/bashParser.ts` | 4,436 | Bash 명령어 파서 |
| 8 | `src/utils/attachments.ts` | 3,997 | 파일 첨부 처리 |
| 9 | `src/services/api/claude.ts` | 3,419 | Claude API 호출 핵심 로직 |
| 10 | `src/services/mcp/client.ts` | 3,348 | MCP 클라이언트 구현 |
| 11 | `src/utils/plugins/pluginLoader.ts` | 3,302 | 플러그인 로딩 시스템 |
| 12 | `src/commands/insights.ts` | 3,200 | insights 명령어 |
| 13 | `src/bridge/bridgeMain.ts` | 2,999 | Bridge 메인 루프 |
| 14 | `src/utils/bash/ast.ts` | 2,679 | Bash AST 파싱 |
| 15 | `src/utils/plugins/marketplaceManager.ts` | 2,643 | 플러그인 마켓플레이스 |
| 16 | `src/tools/BashTool/bashPermissions.ts` | 2,621 | Bash 권한 검사 |
| 17 | `src/tools/BashTool/bashSecurity.ts` | 2,592 | Bash 보안 검사 |
| 18 | `src/native-ts/yoga-layout/index.ts` | 2,578 | Yoga 레이아웃 엔진 포팅 |
| 19 | `src/services/mcp/auth.ts` | 2,465 | MCP OAuth 인증 |
| 20 | `src/bridge/replBridge.ts` | 2,406 | REPL Bridge 연결 |

**특징**: 최대 파일들은 주로 CLI 출력(`print.ts`), 메시지 처리(`messages.ts`), REPL UI(`REPL.tsx`), API 호출(`claude.ts`)에 집중되어 있다. BashTool의 권한/보안 관련 파일만 5,000줄 이상이다.

---

## 4. 도구(tools/) 전체 목록과 스키마

### 4.1 도구 인터페이스

모든 도구는 `src/Tool.ts`에 정의된 `Tool<Input, Output, Progress>` 타입을 구현한다:

```typescript
export type Tool<
  Input extends AnyObject = AnyObject,
  Output = unknown,
  P extends ToolProgressData = ToolProgressData,
> = {
  readonly name: string
  readonly inputSchema: Input
  readonly inputJSONSchema?: ToolInputJSONSchema
  readonly shouldDefer?: boolean
  readonly alwaysLoad?: boolean
  readonly strict?: boolean
  aliases?: string[]
  searchHint?: string
  maxResultSizeChars: number
  
  call(args, context, canUseTool, parentMessage, onProgress?): Promise<ToolResult<Output>>
  description(input, options): Promise<string>
  isConcurrencySafe(input): boolean
  isEnabled(): boolean
  isReadOnly(input): boolean
  isDestructive?(input): boolean
  interruptBehavior?(): 'cancel' | 'block'
  validateInput?(input, context): Promise<ValidationResult>
  // ... 추가 메서드들
}
```

도구 생성은 `buildTool()` 팩토리를 통해 `ToolDef`에서 `Tool` 인스턴스로 변환된다.

### 4.2 기본 도구 목록 (getAllBaseTools)

`src/tools.ts`의 `getAllBaseTools()` 함수에서 등록되는 전체 도구 목록:

#### 핵심 도구 (항상 로드)

| 도구명 | 디렉토리 | 파일 구성 | inputSchema 요약 |
|--------|---------|----------|-----------------|
| **AgentTool** | `AgentTool/` | AgentTool.tsx, runAgent.ts, prompt.ts, builtInAgents.ts, loadAgentsDir.ts, agentMemory.ts, agentMemorySnapshot.ts, agentDisplay.ts, constants.ts, resumeAgent.ts, built-in/ (5개) | `{ prompt, description, subagent_type?, model?, ...}` |
| **BashTool** | `BashTool/` | BashTool.tsx, bashPermissions.ts, bashSecurity.ts, pathValidation.ts, sedValidation.ts, commandSemantics.ts, prompt.ts, UI.tsx 등 (18개) | `{ command, timeout?, run_in_background?, description? }` |
| **FileReadTool** | `FileReadTool/` | FileReadTool.ts, prompt.ts | `{ file_path, offset?, limit?, pages? }` |
| **FileEditTool** | `FileEditTool/` | FileEditTool.ts, constants.ts | `{ file_path, old_string, new_string, replace_all? }` |
| **FileWriteTool** | `FileWriteTool/` | FileWriteTool.ts, prompt.ts, UI.tsx | `{ file_path, content }` |
| **GlobTool** | `GlobTool/` | GlobTool.ts, prompt.ts | `{ pattern, path? }` |
| **GrepTool** | `GrepTool/` | GrepTool.ts, prompt.ts | `{ pattern, path?, type?, glob?, output_mode?, ...}` |
| **NotebookEditTool** | `NotebookEditTool/` | NotebookEditTool.ts, prompt.ts, constants.ts, UI.tsx | `{ notebook_path, cell_number?, ...}` |
| **WebFetchTool** | `WebFetchTool/` | WebFetchTool.ts, prompt.ts, preapproved.ts, utils.ts, UI.tsx | `{ url, prompt? }` |
| **WebSearchTool** | `WebSearchTool/` | WebSearchTool.ts, prompt.ts, UI.tsx | `{ query, ... }` |
| **TodoWriteTool** | `TodoWriteTool/` | TodoWriteTool.ts | `{ todos }` |
| **SkillTool** | `SkillTool/` | SkillTool.ts, UI.tsx | `{ skill: string, args?: string }` |
| **ToolSearchTool** | `ToolSearchTool/` | ToolSearchTool.ts | `{ query: string, max_results?: number }` |
| **TaskOutputTool** | `TaskOutputTool/` | TaskOutputTool.ts | 백그라운드 태스크 출력 |
| **TaskStopTool** | `TaskStopTool/` | TaskStopTool.ts, prompt.ts, UI.tsx | `{ task_id }` |
| **BriefTool** | `BriefTool/` | BriefTool.ts, upload.ts, attachments.ts | 사용자 메시지 전달 |
| **AskUserQuestionTool** | `AskUserQuestionTool/` | AskUserQuestionTool.tsx, prompt.ts | 사용자에게 질문 |
| **ExitPlanModeV2Tool** | `ExitPlanModeTool/` | ExitPlanModeV2Tool.ts, prompt.ts, constants.ts, UI.tsx | 플랜 모드 종료 |
| **EnterPlanModeTool** | `EnterPlanModeTool/` | EnterPlanModeTool.ts, prompt.ts, constants.ts, UI.tsx | `{ plan }` |
| **ListMcpResourcesTool** | `ListMcpResourcesTool/` | ListMcpResourcesTool.ts, prompt.ts | MCP 리소스 목록 |
| **ReadMcpResourceTool** | `ReadMcpResourceTool/` | ReadMcpResourceTool.ts, prompt.ts, UI.tsx | `{ server, uri }` |

#### 조건부 도구 (환경/피처 플래그에 따라)

| 도구명 | 조건 | 역할 |
|--------|------|------|
| **ConfigTool** | `USER_TYPE === 'ant'` | 설정 관리 |
| **TungstenTool** | `USER_TYPE === 'ant'` | 내부 도구 |
| **REPLTool** | `USER_TYPE === 'ant'` | REPL VM 래퍼 |
| **SuggestBackgroundPRTool** | `USER_TYPE === 'ant'` | PR 제안 |
| **PowerShellTool** | `isPowerShellToolEnabled()` | PowerShell 실행 |
| **LSPTool** | `ENABLE_LSP_TOOL` 환경변수 | LSP 연동 |
| **EnterWorktreeTool** | `isWorktreeModeEnabled()` | Git worktree 진입 |
| **ExitWorktreeTool** | `isWorktreeModeEnabled()` | Git worktree 종료 |
| **SendMessageTool** | 항상 (lazy) | 에이전트간 메시지 |
| **TeamCreateTool** | `isAgentSwarmsEnabled()` | 팀 생성 |
| **TeamDeleteTool** | `isAgentSwarmsEnabled()` | 팀 삭제 |
| **TaskCreateTool** | `isTodoV2Enabled()` | 태스크 생성 |
| **TaskGetTool** | `isTodoV2Enabled()` | 태스크 조회 |
| **TaskUpdateTool** | `isTodoV2Enabled()` | 태스크 갱신 |
| **TaskListTool** | `isTodoV2Enabled()` | 태스크 목록 |
| **SleepTool** | `feature('PROACTIVE')` \| `feature('KAIROS')` | 대기 |
| **CronCreateTool** | `feature('AGENT_TRIGGERS')` | 크론 생성 |
| **CronDeleteTool** | `feature('AGENT_TRIGGERS')` | 크론 삭제 |
| **CronListTool** | `feature('AGENT_TRIGGERS')` | 크론 목록 |
| **RemoteTriggerTool** | `feature('AGENT_TRIGGERS_REMOTE')` | 원격 트리거 |
| **MonitorTool** | `feature('MONITOR_TOOL')` | 모니터링 |
| **SendUserFileTool** | `feature('KAIROS')` | 파일 전송 |
| **PushNotificationTool** | `feature('KAIROS')` \| `feature('KAIROS_PUSH_NOTIFICATION')` | 푸시 알림 |
| **SubscribePRTool** | `feature('KAIROS_GITHUB_WEBHOOKS')` | PR 구독 |
| **WebBrowserTool** | `feature('WEB_BROWSER_TOOL')` | 웹 브라우저 |
| **CtxInspectTool** | `feature('CONTEXT_COLLAPSE')` | 컨텍스트 검사 |
| **TerminalCaptureTool** | `feature('TERMINAL_PANEL')` | 터미널 캡처 |
| **SnipTool** | `feature('HISTORY_SNIP')` | 히스토리 스닙 |
| **ListPeersTool** | `feature('UDS_INBOX')` | 피어 목록 |
| **WorkflowTool** | `feature('WORKFLOW_SCRIPTS')` | 워크플로우 |
| **OverflowTestTool** | `feature('OVERFLOW_TEST_TOOL')` | 오버플로우 테스트 |

### 4.3 도구별 inputSchema 상세 예시

```typescript
// ToolSearchTool
inputSchema = z.object({
  query: z.string().describe('Query to find deferred tools...'),
  max_results: z.number().optional().default(5)
    .describe('Maximum number of results to return (default: 5)'),
})

// SkillTool
inputSchema = z.object({
  skill: z.string().describe('The skill name. E.g., "commit", "review-pr", or "pdf"'),
  args: z.string().optional().describe('Optional arguments for the skill'),
})

// ReadMcpResourceTool
inputSchema = z.object({
  server: z.string().describe('The MCP server name'),
  uri: z.string().describe('The resource URI to read'),
})

// TaskCreateTool
inputSchema = z.object({
  description: z.string().describe('Task description'),
  prompt: z.string().describe('Initial prompt for the task'),
  // ... 추가 필드
})
```

---

## 5. 서비스(services/) 구조

### 5.1 services/api/claude.ts (3,419줄)

Claude API 호출의 핵심 모듈. 주요 임포트와 구조:

```typescript
import type {
  BetaContentBlock, BetaMessage, BetaMessageStreamParams,
  BetaRawMessageStreamEvent, BetaStopReason, BetaToolUnion,
  BetaUsage, BetaMessageParam as MessageParam,
} from '@anthropic-ai/sdk/resources/beta/messages/messages.mjs'

// 핵심 의존성
import { getAPIProvider, isFirstPartyAnthropicBaseUrl } from 'src/utils/model/providers.js'
import { getAttributionHeader, getCLISyspromptPrefix } from '../../constants/system.js'
import { getEmptyToolPermissionContext, type Tool, type Tools } from '../../Tool.js'
```

**주요 기능**:
- API 스트리밍 요청 구성 및 전송
- Beta 헤더 관리 (CONTEXT_1M, EFFORT, FAST_MODE, STRUCTURED_OUTPUTS 등)
- 캐시 스코프(CacheScope) 관리
- 메시지 정규화 및 API 포맷 변환
- 에러 처리 및 재시도 로직
- Connector Text 블록 처리 (`feature('CONNECTOR_TEXT')`)
- Auto Mode 상태 관리 (`feature('TRANSCRIPT_CLASSIFIER')`)
- Cached Microcompact (`feature('CACHED_MICROCOMPACT')`)
- Prompt Cache Break Detection (`feature('PROMPT_CACHE_BREAK_DETECTION')`)

### 5.2 services/mcp/client.ts (3,348줄)

MCP(Model Context Protocol) 클라이언트 구현:

```typescript
import { Client } from '@modelcontextprotocol/sdk/client/index.js'
import { SSEClientTransport } from '@modelcontextprotocol/sdk/client/sse.js'
import { StdioClientTransport } from '@modelcontextprotocol/sdk/client/stdio.js'
import { StreamableHTTPClientTransport } from '@modelcontextprotocol/sdk/client/streamableHttp.js'
```

**주요 기능**:
- Stdio, SSE, StreamableHTTP 트랜스포트 지원
- MCP 도구를 Claude Code 도구로 변환
- 리소스/프롬프트 관리
- OAuth 인증 (`services/mcp/auth.ts` - 2,465줄)
- Computer Use 래퍼 (`feature('CHICAGO_MCP')`)
- MCP Skills (`feature('MCP_SKILLS')`)
- 채널 알림/권한 (`channelNotification.ts`, `channelPermissions.ts`)

### 5.3 services/ 전체 하위 모듈

| 모듈 | 파일 수 | 역할 |
|------|---------|------|
| `api/` | 16개 | Claude API 호출, 에러, 로깅, 사용량 추적 |
| `mcp/` | 20개 | MCP 클라이언트, 인증, 설정, 연결 관리 |
| `analytics/` | 8개 | GrowthBook, Datadog, 1P 이벤트 로깅 |
| `compact/` | 10개 | 대화 압축, Auto/Micro/Session Memory Compact |
| `extractMemories/` | 2개 | 메모리 추출 프롬프트 및 로직 |
| `lsp/` | 7개 | LSP 클라이언트, 서버 인스턴스, 매니저 |
| `MagicDocs/` | 2개 | 자동 문서 생성 |
| `oauth/` | 5개 | OAuth 클라이언트, 크립토, 프로필 |
| `plugins/` | 3개 | 플러그인 설치/CLI/작업 |
| `policyLimits/` | 2개 | 정책 제한 |
| `remoteManagedSettings/` | 5개 | 원격 설정 관리 |
| `SessionMemory/` | 3개 | 세션 메모리 프롬프트/유틸 |
| `settingsSync/` | 2개 | 설정 동기화 (업로드/다운로드) |
| `teamMemorySync/` | 5개 | 팀 메모리 동기화, 시크릿 스캐너 |
| `tips/` | 3개 | 팁 레지스트리/스케줄러/히스토리 |
| `tools/` | 4개 | 도구 실행, 오케스트레이션, 스트리밍 |
| `toolUseSummary/` | 1개 | 도구 사용 요약 생성 |
| `autoDream/` | 4개 | 자동 Dream 통합 |
| `PromptSuggestion/` | 2개 | 프롬프트 제안/추측 |
| 기타 | 7개 | 진단, 알림, 토큰 추정, VCR, 보이스 등 |

---

## 6. bridge/ 시스템 상세 (31개 파일)

Bridge 시스템은 Claude Code를 원격으로 제어하는 인프라이다. claude.ai 웹 UI나 GitHub 연동에서 Claude Code 세션을 원격 조작할 때 사용된다.

### 6.1 파일 목록

```
src/bridge/
├── bridgeApi.ts              # API 클라이언트 (BridgeFatalError, validateBridgeId)
├── bridgeConfig.ts           # Bridge 설정
├── bridgeDebug.ts            # 디버그 유틸리티
├── bridgeEnabled.ts          # 활성화 체크 (feature('BRIDGE_MODE') 게이트)
├── bridgeMain.ts             # 메인 루프 (2,999줄) - 폴링, 세션 관리
├── bridgeMessaging.ts        # 메시지 전송/수신
├── bridgePermissionCallbacks.ts  # 권한 콜백
├── bridgePointer.ts          # 포인터 관리
├── bridgeStatusUtil.ts       # 상태 유틸 (formatDuration)
├── bridgeUI.ts               # UI 로깅
├── capacityWake.ts           # 용량 기반 웨이크업
├── codeSessionApi.ts         # 코드 세션 API
├── createSession.ts          # 세션 생성
├── debugUtils.ts             # 디버그 (describeAxiosError)
├── envLessBridgeConfig.ts    # 환경변수 없는 설정
├── flushGate.ts              # 플러시 게이트
├── inboundAttachments.ts     # 인바운드 첨부파일
├── inboundMessages.ts        # 인바운드 메시지
├── initReplBridge.ts         # REPL Bridge 초기화
├── jwtUtils.ts               # JWT 토큰 리프레시
├── pollConfig.ts             # 폴링 설정
├── pollConfigDefaults.ts     # 폴링 기본값
├── remoteBridgeCore.ts       # 원격 Bridge 코어
├── replBridge.ts             # REPL Bridge (2,406줄)
├── replBridgeHandle.ts       # REPL Bridge 핸들
├── replBridgeTransport.ts    # REPL Bridge 트랜스포트
├── sessionIdCompat.ts        # 세션 ID 호환성
├── sessionRunner.ts          # 세션 실행기 (spawn, safeFilenameId)
├── trustedDevice.ts          # 신뢰 디바이스 토큰
├── types.ts                  # 타입 정의
└── workSecret.ts             # 작업 시크릿 (SDK URL 빌드)
```

### 6.2 bridgeMain.ts 핵심 구조

```typescript
export type BackoffConfig = {
  connInitialMs: number       // 2,000
  connCapMs: number           // 120,000 (2분)
  connGiveUpMs: number        // 600,000 (10분)
  generalInitialMs: number    // 500
  generalCapMs: number        // 30,000
  generalGiveUpMs: number     // 600,000
  shutdownGraceMs?: number    // SIGTERM→SIGKILL 유예기간 (기본 30초)
  stopWorkBaseDelayMs?: number // 1000ms
}
```

**핵심 기능**:
- `isMultiSessionSpawnEnabled()`: GrowthBook 게이트로 멀티 세션 스폰 제어
- `pollSleepDetectionThresholdMs()`: 시스템 sleep/wake 감지
- `spawnScriptArgs()`: 자식 프로세스 스폰 인자 구성
- 세션 생성, 폴링, 백오프, JWT 리프레시
- `feature('KAIROS')` 게이트: --session-id, --continue 플래그

### 6.3 bridgeEnabled.ts

```typescript
export function isBridgeEnabled(): boolean {
  return feature('BRIDGE_MODE')
    ? /* GrowthBook 체크 */
    : false
}

export function isAutoConnectEnabled(): boolean {
  return feature('CCR_AUTO_CONNECT')
    ? /* GrowthBook 체크 */
    : false
}

export function isMirrorMode(): boolean {
  return feature('CCR_MIRROR')
    ? /* GrowthBook 체크 */
    : false
}
```

---

## 7. coordinator/ 시스템 상세

코디네이터 시스템은 1개의 파일 `coordinatorMode.ts`에 집중되어 있으며, 멀티 워커 오케스트레이션 패턴을 구현한다.

### 7.1 핵심 함수

```typescript
// 코디네이터 모드 활성화 체크
export function isCoordinatorMode(): boolean {
  if (feature('COORDINATOR_MODE')) {
    return isEnvTruthy(process.env.CLAUDE_CODE_COORDINATOR_MODE)
  }
  return false
}

// 세션 모드 매칭 (resume 시)
export function matchSessionMode(
  sessionMode: 'coordinator' | 'normal' | undefined
): string | undefined

// 워커 도구 컨텍스트 생성
export function getCoordinatorUserContext(
  mcpClients: ReadonlyArray<{ name: string }>,
  scratchpadDir?: string
): { [k: string]: string }

// 코디네이터 시스템 프롬프트
export function getCoordinatorSystemPrompt(): string
```

### 7.2 내부 워커 도구 집합

```typescript
const INTERNAL_WORKER_TOOLS = new Set([
  TEAM_CREATE_TOOL_NAME,
  TEAM_DELETE_TOOL_NAME,
  SEND_MESSAGE_TOOL_NAME,
  SYNTHETIC_OUTPUT_TOOL_NAME,
])
```

### 7.3 코디네이터 시스템 프롬프트 구조

코디네이터 프롬프트는 6개 섹션으로 구성:
1. **Your Role** - 코디네이터 역할 정의
2. **Your Tools** - AgentTool, SendMessageTool, TaskStopTool
3. **Workers** - 워커 도구 접근 범위
4. **Task Workflow** - Research → Synthesis → Implementation → Verification
5. **Writing Worker Prompts** - 워커 프롬프트 작성 가이드
6. **Example Session** - 전체 세션 예제

핵심 원칙: "Workers can't see your conversation. Every prompt must be self-contained."

---

## 8. voice/ 시스템 상세

### 8.1 voice/voiceModeEnabled.ts

```typescript
export function isVoiceGrowthBookEnabled(): boolean {
  return feature('VOICE_MODE')
    ? !getFeatureValue_CACHED_MAY_BE_STALE('tengu_amber_quartz_disabled', false)
    : false
}

export function hasVoiceAuth(): boolean {
  if (!isAnthropicAuthEnabled()) return false
  const tokens = getClaudeAIOAuthTokens()
  return Boolean(tokens?.accessToken)
}

export function isVoiceModeEnabled(): boolean {
  return hasVoiceAuth() && isVoiceGrowthBookEnabled()
}
```

### 8.2 services/voice.ts

네이티브 오디오 캡처 (cpal) 기반 push-to-talk 구현:

```typescript
// 오디오 캡처 모듈 (lazy load)
type AudioNapi = typeof import('audio-capture-napi')
let audioNapi: AudioNapi | null = null

// 녹음 상수
const RECORDING_SAMPLE_RATE = 16000
const RECORDING_CHANNELS = 1
const SILENCE_DURATION_SECS = '2.0'
const SILENCE_THRESHOLD = '3%'
```

- macOS, Linux, Windows에서 네이티브 오디오 캡처
- fallback: SoX `rec` 또는 `arecord` (ALSA)
- `audio-capture-napi` dlopen은 첫 voice keypress 시 실행 (~1-8초)

### 8.3 services/voiceStreamSTT.ts

STT(Speech-to-Text) 스트리밍. `feature('VOICE_MODE')` 게이트 뒤에서만 접근 가능.

### 8.4 services/voiceKeyterms.ts

보이스 키워드 인식 관련 처리.

### 8.5 관련 hooks

```typescript
// hooks/useVoiceIntegration.tsx
const { useVoice, useVoiceState, useVoiceEnabled } = feature('VOICE_MODE')
  ? require('./useVoice.js')
  : { /* 스텁 */ }
```

---

## 9. 피처 플래그 목록 전체

`feature()` 호출은 `bun:bundle` 모듈에서 임포트되며, Bun 번들러가 빌드 시 dead code elimination에 활용한다. 총 **60+개** 고유 피처 플래그가 존재한다.

### 9.1 전체 피처 플래그 목록

| 플래그명 | 사용 위치 수 | 주요 용도 |
|---------|-------------|----------|
| **KAIROS** | 50+ | 어시스턴트 모드 (채팅 세션, 일일 로그, 알림) |
| **BRIDGE_MODE** | 25+ | 원격 Bridge 연결 (claude.ai 웹 UI) |
| **COORDINATOR_MODE** | 15+ | 코디네이터/워커 멀티에이전트 |
| **VOICE_MODE** | 20+ | 음성 입력 (push-to-talk, STT) |
| **TRANSCRIPT_CLASSIFIER** | 25+ | 자동 모드 분류기 (auto mode) |
| **BASH_CLASSIFIER** | 20+ | Bash 명령어 안전성 분류기 |
| **PROACTIVE** | 15+ | 프로액티브 모드 |
| **KAIROS_BRIEF** | 15+ | Brief 출력 모드 |
| **KAIROS_CHANNELS** | 8+ | 채널 시스템 |
| **KAIROS_PUSH_NOTIFICATION** | 3+ | 푸시 알림 |
| **KAIROS_GITHUB_WEBHOOKS** | 3+ | GitHub 웹훅 구독 |
| **KAIROS_DREAM** | 1+ | Dream 스킬 |
| **AGENT_TRIGGERS** | 10+ | 에이전트 트리거 (크론) |
| **AGENT_TRIGGERS_REMOTE** | 2+ | 원격 에이전트 트리거 |
| **AGENT_MEMORY_SNAPSHOT** | 1+ | 에이전트 메모리 스냅샷 |
| **BG_SESSIONS** | 5+ | 백그라운드 세션 |
| **CHICAGO_MCP** | 10+ | Computer Use MCP |
| **MCP_SKILLS** | 8+ | MCP 스킬 시스템 |
| **CONTEXT_COLLAPSE** | 10+ | 컨텍스트 축소 |
| **CACHED_MICROCOMPACT** | 10+ | 캐시 마이크로컴팩트 |
| **REACTIVE_COMPACT** | 2+ | 리액티브 컴팩트 |
| **HISTORY_SNIP** | 6+ | 히스토리 스닙 |
| **TOKEN_BUDGET** | 6+ | 토큰 예산 |
| **CONNECTOR_TEXT** | 5+ | 커넥터 텍스트 블록 |
| **EXTRACT_MEMORIES** | 6+ | 메모리 추출 |
| **MEMORY_SHAPE_TELEMETRY** | 1+ | 메모리 형태 텔레메트리 |
| **TEAMMEM** | 12+ | 팀 메모리 |
| **TEMPLATES** | 3+ | 작업 템플릿/분류기 |
| **EXPERIMENTAL_SKILL_SEARCH** | 5+ | 실험적 스킬 검색 |
| **BUILDING_CLAUDE_APPS** | 1+ | Claude 앱 빌딩 스킬 |
| **RUN_SKILL_GENERATOR** | 1+ | 스킬 생성기 |
| **REVIEW_ARTIFACT** | 1+ | 리뷰 아티팩트 |
| **ULTRAPLAN** | 6+ | 울트라플랜 (고급 계획) |
| **VERIFICATION_AGENT** | 1+ | 검증 에이전트 |
| **UDS_INBOX** | 5+ | Unix Domain Socket 인박스 |
| **FORK_SUBAGENT** | 1+ | 포크 서브에이전트 |
| **BUDDY** | 8+ | 컴패니언 캐릭터 |
| **TORCH** | 1+ | Torch 명령어 |
| **DOWNLOAD_USER_SETTINGS** | 4+ | 설정 다운로드 |
| **UPLOAD_USER_SETTINGS** | 1+ | 설정 업로드 |
| **WEB_BROWSER_TOOL** | 5+ | 웹 브라우저 도구 |
| **TERMINAL_PANEL** | 3+ | 터미널 패널 |
| **MONITOR_TOOL** | 4+ | 모니터 도구 |
| **OVERFLOW_TEST_TOOL** | 1+ | 오버플로우 테스트 |
| **MESSAGE_ACTIONS** | 3+ | 메시지 액션 |
| **COMMIT_ATTRIBUTION** | 4+ | 커밋 어트리뷰션 |
| **STREAMLINED_OUTPUT** | 1+ | 간소화 출력 |
| **FILE_PERSISTENCE** | 2+ | 파일 영속성 |
| **PROMPT_CACHE_BREAK_DETECTION** | 6+ | 프롬프트 캐시 브레이크 감지 |
| **HOOK_PROMPTS** | 1+ | 훅 프롬프트 |
| **AWAY_SUMMARY** | 2+ | 부재 요약 |
| **LODESTONE** | 2+ | Lodestone 시스템 |
| **DIRECT_CONNECT** | 4+ | 다이렉트 커넥트 |
| **SSH_REMOTE** | 3+ | SSH 원격 |
| **CCR_AUTO_CONNECT** | 1+ | 자동 연결 |
| **CCR_MIRROR** | 3+ | 미러 모드 |
| **CCR_REMOTE_SETUP** | 1+ | 원격 설정 |
| **HARD_FAIL** | 1+ | 하드 실패 |
| **DAEMON** | 1+ | 데몬 모드 |
| **QUICK_SEARCH** | 4+ | 빠른 검색 |
| **HISTORY_PICKER** | 3+ | 히스토리 피커 |
| **AUTO_THEME** | 3+ | 자동 테마 |
| **WORKFLOW_SCRIPTS** | 4+ | 워크플로우 스크립트 |
| **TREE_SITTER_BASH_SHADOW** | 3+ | Tree-sitter Bash 섀도우 |
| **ANTI_DISTILLATION_CC** | 1+ | 안티 디스틸레이션 |
| **NATIVE_CLIENT_ATTESTATION** | 1+ | 네이티브 클라이언트 증명 |
| **UNATTENDED_RETRY** | 1+ | 무인 재시도 |
| **IS_LIBC_MUSL** | 1+ | musl libc 감지 |
| **IS_LIBC_GLIBC** | 1+ | glibc 감지 |
| **COWORKER_TYPE_TELEMETRY** | 2+ | 공동작업자 유형 텔레메트리 |

### 9.2 feature() 게이트 패턴

```typescript
// 패턴 1: 조건부 require (dead code elimination)
const module = feature('FLAG')
  ? require('./module.js') as typeof import('./module.js')
  : null

// 패턴 2: 직접 체크
if (feature('FLAG')) {
  // 활성화 시 로직
}

// 패턴 3: 복합 OR 게이트
const enabled = feature('KAIROS') || feature('KAIROS_BRIEF')
```

---

## 10. 빌드 시스템

### 10.1 빌드 도구: Bun

이 프로젝트는 **Bun 번들러**를 사용하여 빌드된다. `feature()` 함수는 `bun:bundle` 모듈에서 임포트되며, Bun의 빌드 시 dead code elimination(DCE) 매크로로 동작한다.

```typescript
import { feature } from 'bun:bundle'
```

빌드 시 `feature('FLAG_NAME')`이 `true` 또는 `false`로 치환되어, false일 경우 해당 코드 블록 전체가 번들에서 제거된다.

### 10.2 번들 출력

- **단일 파일 번들**: `cli.js` (minified)
- **소스맵**: `cli.js.map` (동봉)
- **진입점**: `src/entrypoints/cli.tsx`

### 10.3 진입점 파일

```
src/entrypoints/
├── cli.tsx              # CLI 메인 진입점
├── init.ts              # 초기화
├── mcp.ts               # MCP 서버 모드 진입점
├── agentSdkTypes.ts     # Agent SDK 타입
├── sandboxTypes.ts      # 샌드박스 타입
└── sdk/
    ├── controlSchemas.ts  # SDK 제어 스키마
    ├── coreSchemas.ts     # SDK 코어 스키마
    └── coreTypes.ts       # SDK 코어 타입
```

### 10.4 빌드 변형 (Build Variants)

`USER_TYPE` 환경변수로 빌드 변형 결정:
- `'ant'`: Anthropic 내부 빌드 (ConfigTool, TungstenTool, REPLTool 포함)
- 기타: 외부(공개) 빌드

`feature()` 플래그는 빌드 시 설정되어 외부 빌드에서는 내부 전용 기능이 완전히 제거됨.

### 10.5 런타임 의존성

```json
{
  "optionalDependencies": {
    "@img/sharp-darwin-arm64": "^0.34.2",
    "@img/sharp-darwin-x64": "^0.34.2",
    "@img/sharp-linux-arm64": "^0.34.2",
    "@img/sharp-linux-x64": "^0.34.2",
    "@img/sharp-win32-x64": "^0.34.2"
    // ... 기타 sharp 네이티브 바인딩
  }
}
```

node_modules에는 다음 주요 패키지가 포함:
- `@anthropic-ai/sdk` - Anthropic API SDK
- `@anthropic-ai/bedrock-sdk` - AWS Bedrock 지원
- `@anthropic-ai/vertex-sdk` - Google Vertex AI 지원
- `@anthropic-ai/foundry-sdk` - Foundry 지원
- `@modelcontextprotocol/sdk` - MCP SDK
- `@azure/msal-common` - Azure 인증
- `lodash-es` - 유틸리티
- `zod/v4` - 스키마 검증

### 10.6 bootstrap/state.ts 구조

글로벌 상태 관리 모듈 (`src/bootstrap/state.ts`):

```typescript
// 세션 ID 관리
function getSessionId(): string
function regenerateSessionId(opts?): string
function switchSession(id, projectDir?)

// 비용/사용량 추적
function getTotalCostUSD(): number
function addToTotalCostState(cost, usage, model)

// 모델 관리
function getMainLoopModelOverride(): string | undefined
function setMainLoopModelOverride(model)

// 피처 플래그 상태
function getKairosActive(): boolean
function setKairosActive(v: boolean)

// 350+ getter/setter 함수
```

---

## 부록: 아키텍처 다이어그램 (텍스트)

```
                    ┌─────────────┐
                    │  cli.tsx     │ ← 진입점
                    └──────┬──────┘
                           │
                    ┌──────▼──────┐
                    │   main.tsx   │ ← 초기화, 옵션 파싱
                    └──────┬──────┘
                           │
              ┌────────────┼────────────┐
              │            │            │
       ┌──────▼──────┐ ┌──▼───┐ ┌─────▼─────┐
       │  REPL.tsx    │ │ CLI  │ │ SDK/MCP   │
       │  (화면)      │ │ print│ │ 서버 모드  │
       └──────┬──────┘ └──────┘ └───────────┘
              │
       ┌──────▼──────┐
       │  query.ts    │ ← 쿼리 엔진 (메인 루프)
       │ QueryEngine  │
       └──────┬──────┘
              │
    ┌─────────┼─────────┐
    │         │         │
┌───▼───┐ ┌──▼──┐ ┌───▼────┐
│ tools/ │ │api/ │ │services│
│ 40+   │ │claude│ │compact │
│ 도구   │ │.ts   │ │mcp/   │
└───────┘ └─────┘ │hooks  │
                   └───────┘
```

---

## 참고: 핵심 파일 경로 요약

| 카테고리 | 파일 경로 |
|---------|----------|
| 진입점 | `src/entrypoints/cli.tsx` |
| 메인 초기화 | `src/main.tsx` (4,683줄) |
| 쿼리 엔진 | `src/query.ts`, `src/QueryEngine.ts` |
| REPL 화면 | `src/screens/REPL.tsx` (5,005줄) |
| 도구 인터페이스 | `src/Tool.ts` |
| 도구 레지스트리 | `src/tools.ts` |
| API 클라이언트 | `src/services/api/claude.ts` (3,419줄) |
| MCP 클라이언트 | `src/services/mcp/client.ts` (3,348줄) |
| Bridge 메인 | `src/bridge/bridgeMain.ts` (2,999줄) |
| 코디네이터 | `src/coordinator/coordinatorMode.ts` |
| 보이스 | `src/voice/voiceModeEnabled.ts`, `src/services/voice.ts` |
| 부트스트랩 상태 | `src/bootstrap/state.ts` |
| CLI 출력 | `src/cli/print.ts` (5,594줄) |
| 메시지 처리 | `src/utils/messages.ts` (5,512줄) |
| Bash 보안 | `src/tools/BashTool/bashSecurity.ts` (2,592줄) |
| Bash 권한 | `src/tools/BashTool/bashPermissions.ts` (2,621줄) |
