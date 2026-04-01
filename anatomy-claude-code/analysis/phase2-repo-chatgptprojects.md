# Phase 2: chatgptprojects/claude-code 리포지토리 분석

## 1. 리포지토리 개요

### 원본과의 관계

이 리포지토리는 Anthropic의 공식 npm 패키지 `@anthropic-ai/claude-code` v2.1.88에서 **소스맵 파일(`cli.js.map`)을 추출**하여 원본 TypeScript 소스코드를 복원한 비공식 리포지토리이다. 2026년 3월 31일 [@Fried_rice](https://x.com/Fried_rice)가 발견하고 공개했으며, chatgptprojects 사용자가 GitHub에 정리해 올렸다.

- **npm 패키지:** `@anthropic-ai/claude-code@2.1.88`
- **추출 방법:** `cli.js.map`의 `sourcesContent` 필드에서 원본 `.ts`/`.tsx` 파일 복원
- **빌드 시스템:** esbuild 기반 (`build.mjs`), Bun 런타임 대상
- **패키지 타입:** ESM (`"type": "module"`)

### 리포지토리 루트 구성

```
build.mjs                  # esbuild 빌드 스크립트
cli.js                     # 번들된 CLI 진입점
package.json               # 의존성 및 메타데이터
tsconfig.json              # TypeScript 설정
vendor/                    # 벤더 의존성
src/                       # 전체 소스코드
```

---

## 2. src/ 디렉토리 구조 전체

```
src/
├── main.tsx               # CLI 메인 진입점 (Commander 기반 CLI 파싱, REPL 실행)
├── QueryEngine.ts         # SDK/headless 쿼리 엔진 (대화 라이프사이클 관리)
├── query.ts               # 핵심 쿼리 루프 (API 호출, 도구 실행, 자동 컴팩션)
├── Tool.ts                # Tool 타입 정의 및 buildTool 팩토리
├── tools.ts               # 전체 도구 풀 조립 (getAllBaseTools, getTools, assembleToolPool)
├── commands.ts            # 슬래시 명령어 레지스트리 (getCommands, skills 로딩)
├── context.ts             # 시스템/사용자 컨텍스트 구성
├── cost-tracker.ts        # API 비용 추적
├── history.ts             # 대화 히스토리 관리
├── setup.ts               # 초기 설정
├── Task.ts                # 태스크 관리
├── tasks.ts               # 태스크 유틸리티
├── ink.ts                 # Ink 터미널 UI 진입점
├── replLauncher.tsx        # REPL 실행기
├── dialogLaunchers.tsx     # 다이얼로그 실행기
├── interactiveHelpers.tsx  # 대화형 헬퍼
├── projectOnboardingState.ts # 프로젝트 온보딩 상태
│
├── assistant/             # 어시스턴트 모드 (Kairos)
├── bootstrap/             # 부트스트랩 상태 관리
├── bridge/                # 원격 제어 브리지
├── buddy/                 # 컴패니언(버디) 기능
├── cli/                   # CLI 핸들러 및 트랜스포트
│   ├── handlers/
│   └── transports/
├── commands/              # 60+ 슬래시 명령어 구현
├── components/            # React/Ink UI 컴포넌트
├── constants/             # 상수 및 설정값
├── context/               # React 컨텍스트 (알림, 메일박스 등)
├── coordinator/           # 코디네이터 모드 (다중 에이전트 조율)
├── entrypoints/           # 진입점 (init, SDK 타입)
├── hooks/                 # React 커스텀 훅
├── ink/                   # Ink 터미널 렌더링 엔진 (커스텀 포크)
│   ├── components/        # Box, Text, ScrollBox 등
│   ├── events/            # 이벤트 시스템
│   ├── hooks/             # Ink 전용 훅
│   ├── layout/            # 레이아웃 엔진 (Yoga)
│   └── termio/            # 터미널 I/O 파서
├── keybindings/           # 키바인딩 시스템
├── memdir/                # 메모리 디렉토리 (CLAUDE.md)
├── migrations/            # 데이터 마이그레이션
├── moreright/             # 추가 권한 관리
├── native-ts/             # 네이티브 TypeScript 유틸리티
├── outputStyles/          # 출력 스타일
├── plugins/               # 플러그인 시스템
├── query/                 # 쿼리 관련 모듈 (config, deps, transitions, stopHooks)
├── remote/                # 원격 세션
├── schemas/               # JSON 스키마
├── screens/               # UI 화면
├── server/                # 서버 모듈
├── services/              # 핵심 서비스 (API, MCP, OAuth, analytics 등)
├── skills/                # 스킬 시스템
├── state/                 # AppState 상태 관리
├── tasks/                 # 태스크 타입 및 관리
├── tools/                 # 40+ 도구 구현
├── types/                 # 공유 타입 정의
├── typings/               # 타입 선언
├── upstreamproxy/         # 업스트림 프록시
├── utils/                 # 유틸리티 (250+ 파일)
├── vim/                   # Vim 모드
└── voice/                 # 음성 모드
```

---

## 3. 핵심 엔진 파일 분석

### 3.1 main.tsx - CLI 메인 진입점

main.tsx는 약 785KB에 달하는 거대 파일로, Commander.js 기반 CLI 파싱과 REPL 실행을 담당한다. 모듈 로딩 최적화를 위해 시작 시 프로파일링 체크포인트와 선행 프리페치를 실행한다.

```typescript
// 시작 시 MDM 설정, 키체인 프리페치를 병렬로 실행
profileCheckpoint('main_tsx_entry');
startMdmRawRead();
startKeychainPrefetch();
```

주요 역할:
- **Commander.js 기반 CLI 파싱**: `--model`, `--agent`, `--tools`, `--max-turns` 등 옵션 처리
- **인증 및 초기화**: GrowthBook, OAuth, 정책 제한, 원격 설정 로딩
- **REPL 실행**: `launchRepl()` 호출로 인터랙티브 모드 시작
- **헤드리스 모드**: `-p` (print) 플래그로 비대화형 실행
- **feature 플래그**: `bun:bundle`의 `feature()` 함수로 Dead Code Elimination 수행

### 3.2 QueryEngine.ts - SDK/headless 쿼리 엔진

`QueryEngine` 클래스는 대화 라이프사이클을 관리하는 핵심 클래스이다. 하나의 대화에 하나의 QueryEngine 인스턴스가 대응한다.

```typescript
export type QueryEngineConfig = {
  cwd: string
  tools: Tools
  commands: Command[]
  mcpClients: MCPServerConnection[]
  agents: AgentDefinition[]
  canUseTool: CanUseToolFn
  getAppState: () => AppState
  setAppState: (f: (prev: AppState) => AppState) => void
  initialMessages?: Message[]
  readFileCache: FileStateCache
  customSystemPrompt?: string
  appendSystemPrompt?: string
  userSpecifiedModel?: string
  fallbackModel?: string
  thinkingConfig?: ThinkingConfig
  maxTurns?: number
  maxBudgetUsd?: number
  taskBudget?: { total: number }
  jsonSchema?: Record<string, unknown>
  // ...기타 옵션
}

export class QueryEngine {
  private config: QueryEngineConfig
  private mutableMessages: Message[]
  private abortController: AbortController
  private permissionDenials: SDKPermissionDenial[]
  private totalUsage: NonNullableUsage

  async *submitMessage(
    prompt: string | ContentBlockParam[],
    options?: { uuid?: string; isMeta?: boolean },
  ): AsyncGenerator<SDKMessage, void, unknown>

  interrupt(): void
  getMessages(): readonly Message[]
  getReadFileState(): FileStateCache
  getSessionId(): string
  setModel(model: string): void
}
```

핵심 동작:
1. **시스템 프롬프트 구성**: `fetchSystemPromptParts()`로 기본 프롬프트, 사용자 컨텍스트, 시스템 컨텍스트 조합
2. **사용자 입력 처리**: `processUserInput()`으로 슬래시 명령 파싱, 첨부파일 처리
3. **쿼리 루프 실행**: `query()`를 호출하여 API 응답과 도구 실행 반복
4. **비용/예산 관리**: `maxBudgetUsd`, `taskBudget` 등으로 비용 한도 체크
5. **결과 스트리밍**: `AsyncGenerator<SDKMessage>`로 메시지를 yield하여 SDK 소비자에게 전달

편의 래퍼 함수 `ask()`는 단일 프롬프트를 위한 one-shot 사용을 지원한다.

### 3.3 query.ts - 핵심 쿼리 루프

`query()` 함수는 **API 호출 -> 스트리밍 -> 도구 실행 -> 첨부 -> 재귀** 루프의 핵심이다. `while(true)` 루프 안에서 각 턴을 관리한다.

```typescript
export type QueryParams = {
  messages: Message[]
  systemPrompt: SystemPrompt
  userContext: { [k: string]: string }
  systemContext: { [k: string]: string }
  canUseTool: CanUseToolFn
  toolUseContext: ToolUseContext
  fallbackModel?: string
  querySource: QuerySource
  maxTurns?: number
  taskBudget?: { total: number }
  deps?: QueryDeps
}

export async function* query(
  params: QueryParams,
): AsyncGenerator<
  StreamEvent | RequestStartEvent | Message | TombstoneMessage | ToolUseSummaryMessage,
  Terminal
>
```

쿼리 루프의 핵심 단계:
1. **Snip 컴팩션** (HISTORY_SNIP): 오래된 히스토리 제거
2. **마이크로 컴팩션**: 도구 결과 압축
3. **Context Collapse** (CONTEXT_COLLAPSE): 컨텍스트 축소
4. **자동 컴팩션** (autocompact): 토큰 한도 초과 시 전체 히스토리 요약
5. **API 호출** (`deps.callModel`): 스트리밍 응답
6. **스트리밍 도구 실행** (`StreamingToolExecutor`): API 응답 중 도구를 병렬 실행
7. **첨부 메시지** (`getAttachmentMessages`): 메모리, 스킬 디스커버리 주입
8. **대기 명령 처리**: 큐에 있는 명령 드레인
9. **Stop Hooks**: 응답 완료 후 후크 실행

특수 복구 경로:
- **Prompt-too-long 복구**: Context Collapse 드레인 -> Reactive Compact -> 에러 표출
- **Max-output-tokens 복구**: OTK 에스컬레이션(8k->64k) -> 다중 턴 복구 (최대 3회)
- **모델 폴백**: `FallbackTriggeredError` 발생 시 `fallbackModel`로 전환

### 3.4 Tool.ts - 도구 타입 시스템

모든 도구의 인터페이스를 정의하는 핵심 타입 파일이다.

```typescript
export type Tool<
  Input extends AnyObject = AnyObject,
  Output = unknown,
  P extends ToolProgressData = ToolProgressData,
> = {
  readonly name: string
  aliases?: string[]
  searchHint?: string
  readonly inputSchema: Input
  readonly inputJSONSchema?: ToolInputJSONSchema
  readonly shouldDefer?: boolean
  readonly alwaysLoad?: boolean
  readonly strict?: boolean
  isMcp?: boolean
  isLsp?: boolean
  maxResultSizeChars: number

  // 핵심 메서드
  call(args, context, canUseTool, parentMessage, onProgress?): Promise<ToolResult<Output>>
  description(input, options): Promise<string>
  prompt(options): Promise<string>
  checkPermissions(input, context): Promise<PermissionResult>
  validateInput?(input, context): Promise<ValidationResult>
  isEnabled(): boolean
  isReadOnly(input): boolean
  isDestructive?(input): boolean
  isConcurrencySafe(input): boolean

  // UI 렌더링
  renderToolUseMessage(input, options): React.ReactNode
  renderToolResultMessage?(content, progressMessages, options): React.ReactNode
  renderToolUseProgressMessage?(progressMessages, options): React.ReactNode
  userFacingName(input): string
  getToolUseSummary?(input): string | null
  getActivityDescription?(input): string | null

  // 권한 및 분류
  preparePermissionMatcher?(input): Promise<(pattern: string) => boolean>
  toAutoClassifierInput(input): unknown
  mapToolResultToToolResultBlockParam(content, toolUseID): ToolResultBlockParam
}

export type ToolUseContext = {
  options: {
    commands: Command[]
    mainLoopModel: string
    tools: Tools
    thinkingConfig: ThinkingConfig
    mcpClients: MCPServerConnection[]
    // ...
  }
  abortController: AbortController
  readFileState: FileStateCache
  getAppState(): AppState
  setAppState(f: (prev: AppState) => AppState): void
  messages: Message[]
  agentId?: AgentId
  // ...50+ 필드
}
```

`buildTool()` 팩토리 함수는 기본값을 주입하여 완전한 Tool 객체를 생성한다:

```typescript
export function buildTool<D extends AnyToolDef>(def: D): BuiltTool<D> {
  return {
    ...TOOL_DEFAULTS,
    userFacingName: () => def.name,
    ...def,
  } as BuiltTool<D>
}
```

### 3.5 tools.ts - 도구 풀 조립

모든 내장 도구를 등록하고 환경에 따라 필터링하는 모듈이다.

```typescript
export function getAllBaseTools(): Tools {
  return [
    AgentTool, TaskOutputTool, BashTool,
    ...(hasEmbeddedSearchTools() ? [] : [GlobTool, GrepTool]),
    ExitPlanModeV2Tool, FileReadTool, FileEditTool, FileWriteTool,
    NotebookEditTool, WebFetchTool, TodoWriteTool, WebSearchTool,
    TaskStopTool, AskUserQuestionTool, SkillTool, EnterPlanModeTool,
    // 조건부 도구 (feature flag, 환경 변수)
    ...(isWorktreeModeEnabled() ? [EnterWorktreeTool, ExitWorktreeTool] : []),
    ...(SleepTool ? [SleepTool] : []),
    ...cronTools,
    BriefTool, ListMcpResourcesTool, ReadMcpResourceTool,
    ...(isToolSearchEnabledOptimistic() ? [ToolSearchTool] : []),
    // ...기타 조건부 도구
  ]
}

export function assembleToolPool(
  permissionContext: ToolPermissionContext,
  mcpTools: Tools,
): Tools   // 내장 도구 + MCP 도구를 병합하고 deny 규칙으로 필터링
```

---

## 4. 도구(tools/) 디렉토리: 전체 목록과 역할

### 핵심 도구 (항상 활성화)

| 도구 | 역할 |
|------|------|
| **AgentTool** | 서브에이전트 생성 및 관리. 탐색, 계획, 검증, 범용 에이전트 등 내장 에이전트 포함. `runAgent.ts`, `forkSubagent.ts`, `resumeAgent.ts` 등 지원 파일 |
| **BashTool** | 셸 명령 실행. 보안 검사(`bashSecurity.ts`), 권한 관리(`bashPermissions.ts`), 명령 시맨틱 분류(`commandSemantics.ts`), 파괴적 명령 경고, sed 검증, 샌드박스 판단 등 |
| **FileReadTool** | 파일 읽기. 이미지 처리(`imageProcessor.ts`), 읽기 제한(`limits.ts`) 포함 |
| **FileEditTool** | 파일 편집 (정확한 문자열 교체). 유틸리티와 타입 지원 |
| **FileWriteTool** | 새 파일 작성 또는 전체 덮어쓰기 |
| **GlobTool** | 파일 패턴 검색 (glob 매칭) |
| **GrepTool** | ripgrep 기반 정규식 콘텐츠 검색 |
| **WebFetchTool** | URL 가져오기. 사전 승인 URL 목록(`preapproved.ts`) 포함 |
| **WebSearchTool** | 웹 검색 기능 |
| **TodoWriteTool** | 할 일 목록 관리 (체크리스트) |
| **TaskStopTool** | 작업 중단 |
| **TaskOutputTool** | 작업 출력 표시 |
| **AskUserQuestionTool** | 사용자에게 질문 (대화형 프롬프트) |
| **SkillTool** | 슬래시 명령 스킬 실행 |
| **NotebookEditTool** | Jupyter 노트북 셀 편집 |
| **BriefTool** | 브리핑 생성 및 파일 업로드 |
| **ToolSearchTool** | 지연 로딩된 도구 검색 (Fuse.js 키워드 매칭) |

### 플랜 모드 도구

| 도구 | 역할 |
|------|------|
| **EnterPlanModeTool** | 플랜 모드 진입 (읽기 전용) |
| **ExitPlanModeV2Tool** | 플랜 모드 종료 및 실행 전환 |

### Worktree 도구

| 도구 | 역할 |
|------|------|
| **EnterWorktreeTool** | Git worktree 생성 및 진입 |
| **ExitWorktreeTool** | Git worktree 종료 |

### MCP 관련 도구

| 도구 | 역할 |
|------|------|
| **MCPTool** | MCP 프로토콜 도구 래퍼 |
| **McpAuthTool** | MCP 인증 처리 |
| **ListMcpResourcesTool** | MCP 리소스 목록 조회 |
| **ReadMcpResourceTool** | MCP 리소스 읽기 |

### LSP 도구

| 도구 | 역할 |
|------|------|
| **LSPTool** | Language Server Protocol 통합 (심볼 컨텍스트, 포매터) |

### 태스크 관리 도구 (TodoV2)

| 도구 | 역할 |
|------|------|
| **TaskCreateTool** | 태스크 생성 |
| **TaskGetTool** | 태스크 조회 |
| **TaskUpdateTool** | 태스크 업데이트 |
| **TaskListTool** | 태스크 목록 |

### 팀/협업 도구

| 도구 | 역할 |
|------|------|
| **TeamCreateTool** | 에이전트 팀(swarm) 생성 |
| **TeamDeleteTool** | 에이전트 팀 삭제 |
| **SendMessageTool** | 피어에게 메시지 전송 |

### 조건부 도구 (feature flag)

| 도구 | 조건 | 역할 |
|------|------|------|
| **SleepTool** | PROACTIVE / KAIROS | 대기 (백그라운드 알림 수집) |
| **CronCreateTool/CronDeleteTool/CronListTool** | AGENT_TRIGGERS | 크론 스케줄 관리 |
| **RemoteTriggerTool** | AGENT_TRIGGERS_REMOTE | 원격 트리거 |
| **MonitorTool** | MONITOR_TOOL | 모니터링 |
| **SendUserFileTool** | KAIROS | 사용자에게 파일 전송 |
| **PushNotificationTool** | KAIROS | 푸시 알림 |
| **SubscribePRTool** | KAIROS_GITHUB_WEBHOOKS | PR 웹훅 구독 |
| **PowerShellTool** | Windows 환경 | PowerShell 명령 실행 |
| **ConfigTool** | ant 사용자 전용 | 설정 관리 |
| **TungstenTool** | ant 사용자 전용 | tmux 세션 관리 |
| **REPLTool** | ant 사용자 전용 | VM 기반 REPL 실행 |
| **SuggestBackgroundPRTool** | ant 사용자 전용 | 백그라운드 PR 제안 |
| **WorkflowTool** | WORKFLOW_SCRIPTS | 워크플로우 스크립트 실행 |
| **SnipTool** | HISTORY_SNIP | 히스토리 스닙 관리 |
| **ListPeersTool** | UDS_INBOX | 피어 목록 조회 |
| **WebBrowserTool** | WEB_BROWSER_TOOL | 웹 브라우저 도구 |
| **TerminalCaptureTool** | TERMINAL_PANEL | 터미널 캡처 |
| **CtxInspectTool** | CONTEXT_COLLAPSE | 컨텍스트 검사 |
| **OverflowTestTool** | OVERFLOW_TEST_TOOL | 오버플로우 테스트 |
| **VerifyPlanExecutionTool** | CLAUDE_CODE_VERIFY_PLAN | 플랜 실행 검증 |

### 특수 도구

| 도구 | 역할 |
|------|------|
| **SyntheticOutputTool** | JSON 스키마 기반 구조화된 출력 (SDK 전용) |
| **TestingPermissionTool** | 테스트 전용 권한 도구 |

---

## 5. 명령어(commands/) 디렉토리: 주요 명령어 분석

commands.ts에서 약 **80개 이상의 슬래시 명령어**를 등록한다. 명령어 타입은 3가지:
- **`prompt`**: 모델에게 전달되는 프롬프트 기반 명령 (스킬 포함)
- **`local`**: 로컬에서 실행되고 텍스트 결과를 반환
- **`local-jsx`**: 로컬에서 실행되고 Ink UI를 렌더링

### 핵심 명령어

| 명령어 | 타입 | 역할 |
|--------|------|------|
| `/help` | local-jsx | 도움말 표시 |
| `/clear` | local | 대화 초기화, 캐시 삭제 |
| `/compact` | local | 컨텍스트 수동 컴팩션 |
| `/resume` | local-jsx | 이전 세션 복원 |
| `/model` | local-jsx | 모델 선택/변경 |
| `/config` | local-jsx | 설정 관리 |
| `/mcp` | local-jsx | MCP 서버 추가/관리 |
| `/permissions` | local-jsx | 권한 규칙 관리 |
| `/hooks` | local-jsx | 후크 설정 |
| `/memory` | local-jsx | CLAUDE.md 메모리 파일 편집 |
| `/plugin` | local-jsx | 플러그인 관리 (마켓플레이스 탐색, 설치, 삭제) |
| `/skills` | local-jsx | 스킬 목록 조회 |
| `/agents` | local-jsx | 에이전트 관리 |
| `/context` | local-jsx / local | 컨텍스트 상태 표시 |
| `/cost` | local | 세션 비용 표시 |
| `/usage` | local-jsx | 사용량 표시 |
| `/diff` | local-jsx | 변경사항 diff 표시 |
| `/rewind` | local | 파일 변경 되돌리기 |
| `/session` | local-jsx | 세션 정보/QR 코드 |
| `/export` | local-jsx | 대화 내보내기 |
| `/copy` | local-jsx | 마지막 메시지 복사 |
| `/theme` | local-jsx | 테마 변경 |
| `/vim` | local | Vim 모드 토글 |
| `/plan` | local-jsx | 플랜 모드 토글 |
| `/fast` | local-jsx | 빠른 모드 토글 |
| `/effort` | local-jsx | 노력 수준 설정 |
| `/thinkback` | local-jsx | 사고 과정 재생 |
| `/doctor` | local-jsx | 환경 진단 |
| `/add-dir` | local-jsx | 추가 작업 디렉토리 |
| `/branch` | local | Git 브랜치 관리 |
| `/tasks` | local-jsx | 백그라운드 태스크 관리 |
| `/stats` | local-jsx | 통계 표시 |
| `/files` | local | 추적 파일 목록 |
| `/status` | local-jsx | 상태 표시 |
| `/feedback` | local-jsx | 피드백 전송 |
| `/ide` | local-jsx | IDE 통합 설치 |
| `/mobile` | local-jsx | 모바일 QR 코드 |
| `/desktop` | local-jsx | 데스크톱 앱 설치 |
| `/upgrade` | local-jsx | 업그레이드 |
| `/login` / `/logout` | local-jsx | 인증 관리 |
| `/install-github-app` | local-jsx | GitHub 앱 설치 (다단계 위자드) |
| `/install-slack-app` | local | Slack 앱 설치 |
| `/sandbox-toggle` | local-jsx | 샌드박스 모드 토글 |
| `/keybindings` | local | 키바인딩 관리 |

### Ant(내부 전용) 명령어

```typescript
export const INTERNAL_ONLY_COMMANDS = [
  backfillSessions, breakCache, bughunter, commit, commitPushPr,
  ctx_viz, goodClaude, issue, initVerifiers, mockLimits,
  bridgeKick, version, resetLimits, onboarding, share, summary,
  teleport, antTrace, perfIssue, env, oauthRefresh, debugToolCall,
  agentsPlatform, autofixPr,
]
```

### 스킬/프롬프트 명령어

스킬은 `prompt` 타입 명령어로, 모델이 SkillTool을 통해 직접 호출할 수 있다:
- **번들 스킬**: `skills/bundled/` 디렉토리에서 로드
- **플러그인 스킬**: 플러그인 시스템에서 제공
- **사용자 스킬**: `.claude/skills/` 디렉토리에서 로드
- **MCP 스킬**: MCP 서버에서 제공

---

## 6. 서비스(services/) 디렉토리

### 6.1 services/api/ - API 서비스

Claude API와의 통신을 담당한다.

- **claude.ts**: 핵심 API 호출 (`callModel`), 스트리밍 처리, 모델 폴백
- **bootstrap.ts**: 부트스트랩 데이터 프리페치
- **errors.ts**: API 에러 분류 (`categorizeRetryableAPIError`, `isPromptTooLongMessage`)
- **logging.ts**: API 사용량 로깅 (`NonNullableUsage`, `EMPTY_USAGE`)
- **withRetry.ts**: 재시도 로직 (`FallbackTriggeredError`)
- **dumpPrompts.ts**: 프롬프트 덤프 (디버깅용)
- **filesApi.ts**: 파일 다운로드 API
- **referral.ts**: 패스 적격성 프리페치

### 6.2 services/mcp/ - Model Context Protocol

MCP 서버 연결 및 도구/리소스 관리:

- **client.ts**: MCP 클라이언트 (`getMcpToolsCommandsAndResources`)
- **types.ts**: `MCPServerConnection`, `McpServerConfig`, `ServerResource` 등 타입
- **channelPermissions.ts**: 채널 권한 콜백
- **elicitationHandler.ts**: MCP elicitation 이벤트
- **officialRegistry.ts**: 공식 MCP URL 프리페치

### 6.3 services/oauth/ - OAuth 인증

OAuth 2.0 인증 플로우 관리

### 6.4 services/analytics/ - 분석

- **index.ts**: `logEvent()` 함수 - 텔레메트리 이벤트 전송
- **growthbook.ts**: GrowthBook feature flag 시스템
- **sink.ts**: 분석 게이트 초기화
- **config.ts**: 분석 비활성화 설정

### 6.5 services/compact/ - 컴팩션 서비스

대화 히스토리 압축:

- **autoCompact.ts**: 자동 컴팩션 (`isAutoCompactEnabled`, `calculateTokenWarningState`, `AutoCompactTrackingState`)
- **compact.ts**: 컴팩션 실행 (`buildPostCompactMessages`)
- **reactiveCompact.ts**: 반응형 컴팩션 (413 에러 복구)
- **snipCompact.ts**: 스닙 컴팩션 (오래된 메시지 제거)
- **snipProjection.ts**: 스닙 프로젝션

### 6.6 기타 서비스

| 서비스 | 역할 |
|--------|------|
| **services/tools/** | `StreamingToolExecutor`, `toolOrchestration` - 도구 병렬 실행 엔진 |
| **services/toolUseSummary/** | Haiku 모델로 도구 사용 요약 생성 |
| **services/lsp/** | Language Server Protocol 클라이언트 |
| **services/plugins/** | 플러그인 CLI 명령 (`VALID_INSTALLABLE_SCOPES`) |
| **services/policyLimits/** | 정책 제한 로드/갱신 |
| **services/remoteManagedSettings/** | 원격 관리 설정 |
| **services/settingsSync/** | 설정 동기화 |
| **services/teamMemorySync/** | 팀 메모리 동기화 |
| **services/tips/** | 팁 표시 |
| **services/PromptSuggestion/** | 프롬프트 제안 |
| **services/SessionMemory/** | 세션 메모리 |
| **services/extractMemories/** | 메모리 추출 |
| **services/MagicDocs/** | 문서 자동 생성 |
| **services/AgentSummary/** | 에이전트 요약 |
| **services/autoDream/** | 자동 드림 기능 |
| **services/contextCollapse/** | 컨텍스트 축소 |
| **services/skillSearch/** | 스킬 검색 (로컬 인덱스, 프리페치) |

---

## 7. 상태 관리(state/) 구조

### 7.1 AppState 타입

`AppStateStore.ts`에서 정의되는 `AppState`는 **전역 애플리케이션 상태**를 포함한다:

```typescript
export type AppState = DeepImmutable<{
  settings: SettingsJson
  verbose: boolean
  mainLoopModel: ModelSetting
  mainLoopModelForSession: ModelSetting
  toolPermissionContext: ToolPermissionContext
  agent: string | undefined
  kairosEnabled: boolean

  // 원격/브릿지 상태
  remoteSessionUrl: string | undefined
  remoteConnectionStatus: 'connecting' | 'connected' | 'reconnecting' | 'disconnected'
  replBridgeEnabled: boolean
  replBridgeConnected: boolean
  replBridgeSessionActive: boolean
  // ...15+ 브릿지 관련 필드

  // UI 상태
  expandedView: 'none' | 'tasks' | 'teammates'
  isBriefOnly: boolean
  footerSelection: FooterItem | null
  statusLineText: string | undefined
  // ...
}> & {
  // Mutable 필드 (DeepImmutable 제외)
  tasks: { [taskId: string]: TaskState }
  agentNameRegistry: Map<string, AgentId>
  mcp: {
    clients: MCPServerConnection[]
    tools: Tool[]
    commands: Command[]
    resources: Record<string, ServerResource[]>
    pluginReconnectKey: number
  }
  plugins: {
    enabled: LoadedPlugin[]
    disabled: LoadedPlugin[]
    commands: Command[]
    errors: PluginError[]
    installationStatus: { marketplaces: [...]; plugins: [...] }
    needsRefresh: boolean
  }
  agentDefinitions: AgentDefinitionsResult
  fileHistory: FileHistoryState
  attribution: AttributionState
  todos: { [agentId: string]: TodoList }
  notifications: { current: Notification | null; queue: Notification[] }
  elicitation: { queue: ElicitationRequestEvent[] }
  thinkingEnabled: boolean | undefined
  promptSuggestionEnabled: boolean
  sessionHooks: SessionHooksState
  speculation: SpeculationState
  // ...컴퓨터 사용, REPL, 팀, 인박스, 등
}
```

### 7.2 상태 관리 패턴

상태 관리는 **React 외부 스토어** 패턴을 사용한다:

```typescript
// store.ts - 외부 스토어 생성
export type Store<T> = {
  getState: () => T
  setState: (updater: (prev: T) => T) => void
  subscribe: (listener: () => void) => () => void
}

// AppState.tsx - React 바인딩
export function useAppState<T>(selector: (state: AppState) => T): T {
  const store = useAppStore()
  const get = () => selector(store.getState())
  return useSyncExternalStore(store.subscribe, get, get)
}

export function useSetAppState() {
  return useAppStore().setState
}
```

핵심 특성:
- `useSyncExternalStore`로 React 18+ 동시 모드 호환
- 셀렉터 기반 구독으로 불필요한 리렌더링 방지
- `DeepImmutable` 래퍼로 상태 불변성 강제 (함수 타입 필드는 제외)
- `onChangeAppState` 콜백으로 상태 변경 사이드이펙트 처리

### 7.3 state/ 디렉토리 파일

```
state/
├── AppState.tsx              # React Provider (AppStateProvider, useAppState, useSetAppState)
├── AppStateStore.ts          # AppState 타입 정의 및 getDefaultAppState()
├── onChangeAppState.ts       # 상태 변경 시 사이드이펙트 핸들러
├── selectors.ts              # 상태 셀렉터
├── store.ts                  # 외부 스토어 구현 (createStore)
└── teammateViewHelpers.ts    # 팀메이트 뷰 헬퍼
```

---

## 8. package.json 의존성 전체 분석

### 8.1 핵심 AI/API 의존성

| 패키지 | 버전 | 역할 |
|--------|------|------|
| `@anthropic-ai/sdk` | ^0.74.0 | Anthropic Claude API SDK |
| `@anthropic-ai/bedrock-sdk` | ^0.26.4 | AWS Bedrock 연동 |
| `@anthropic-ai/vertex-sdk` | ^0.14.4 | Google Vertex AI 연동 |
| `@anthropic-ai/foundry-sdk` | ^0.2.3 | Foundry 연동 |
| `@anthropic-ai/mcpb` | ^2.1.2 | MCP 빌더 |
| `@modelcontextprotocol/sdk` | ^1.29.0 | MCP 프로토콜 SDK |

### 8.2 클라우드 프로바이더

| 패키지 | 버전 | 역할 |
|--------|------|------|
| `@aws-sdk/client-bedrock` | ^3.1020.0 | AWS Bedrock 클라이언트 |
| `@aws-sdk/client-bedrock-runtime` | ^3.1020.0 | Bedrock 런타임 |
| `@aws-sdk/credential-provider-node` | ^3.972.28 | AWS 인증 |
| `@aws-sdk/credential-providers` | ^3.1020.0 | AWS 자격 증명 |
| `@azure/identity` | ^4.13.1 | Azure 인증 |
| `google-auth-library` | ^10.6.2 | Google 인증 |
| `@smithy/core` | ^3.23.13 | AWS SDK 핵심 |
| `@smithy/node-http-handler` | ^4.5.1 | AWS HTTP 핸들러 |

### 8.3 텔레메트리/분석

| 패키지 | 버전 | 역할 |
|--------|------|------|
| `@opentelemetry/api` | ^1.9.0 | OTel API |
| `@opentelemetry/api-logs` | ^0.214.0 | OTel 로그 |
| `@opentelemetry/core` | ^2.2.0 | OTel 핵심 |
| `@opentelemetry/exporter-logs-otlp-http` | ^0.214.0 | OTLP 로그 내보내기 |
| `@opentelemetry/exporter-trace-otlp-http` | ^0.214.0 | OTLP 트레이스 내보내기 |
| `@opentelemetry/resources` | ^2.2.0 | OTel 리소스 |
| `@opentelemetry/sdk-logs` | ^0.214.0 | OTel 로그 SDK |
| `@opentelemetry/sdk-metrics` | ^2.2.0 | OTel 메트릭 SDK |
| `@opentelemetry/sdk-trace-base` | ^2.2.0 | OTel 트레이스 SDK |
| `@growthbook/growthbook` | ^1.6.5 | Feature flag / A/B 테스트 |

### 8.4 UI/터미널 렌더링

| 패키지 | 버전 | 역할 |
|--------|------|------|
| `react` | ^19.2.0 | React (Ink용) |
| `react-reconciler` | ^0.33.0 | React 커스텀 렌더러 |
| `chalk` | ^5.6.2 | ANSI 색상 |
| `cli-boxes` | ^4.0.1 | 박스 문자 |
| `cli-highlight` | ^2.1.11 | 코드 하이라이팅 |
| `highlight.js` | ^11.11.1 | 구문 강조 |
| `figures` | ^6.1.0 | 유니코드 기호 |
| `wrap-ansi` | ^10.0.0 | ANSI 문자열 래핑 |
| `strip-ansi` | ^7.2.0 | ANSI 이스케이프 제거 |
| `indent-string` | ^5.0.0 | 들여쓰기 |
| `@alcalzone/ansi-tokenize` | ^0.3.0 | ANSI 토큰화 |
| `bidi-js` | ^1.0.3 | 양방향 텍스트 |
| `emoji-regex` | ^10.6.0 | 이모지 정규식 |
| `get-east-asian-width` | ^1.5.0 | 동아시아 문자 너비 |
| `supports-hyperlinks` | ^4.4.0 | 터미널 하이퍼링크 지원 |
| `qrcode` | ^1.5.4 | QR 코드 생성 |
| `asciichart` | ^1.5.25 | ASCII 차트 |

### 8.5 파일/프로세스/네트워크

| 패키지 | 버전 | 역할 |
|--------|------|------|
| `chokidar` | ^4.0.0 | 파일 시스템 감시 |
| `execa` | ^7.2.0 | 자식 프로세스 실행 |
| `tree-kill` | ^1.2.2 | 프로세스 트리 종료 |
| `signal-exit` | ^4.1.0 | 종료 시그널 처리 |
| `axios` | 1.14.0 | HTTP 클라이언트 |
| `undici` | ^7.24.6 | HTTP/2 클라이언트 |
| `https-proxy-agent` | ^8.0.0 | HTTPS 프록시 |
| `ws` | ^8.20.0 | WebSocket |
| `cacache` | ^20.0.4 | 컨텐츠 기반 캐시 |
| `proper-lockfile` | ^4.1.2 | 파일 잠금 |
| `fflate` | ^0.8.2 | 압축/해제 |

### 8.6 파싱/변환/유틸리티

| 패키지 | 버전 | 역할 |
|--------|------|------|
| `commander` | ^14.0.3 | CLI 인수 파싱 |
| `@commander-js/extra-typings` | ^14.0.0 | Commander 타입 |
| `zod` | ^4.0.0 | 스키마 검증 |
| `ajv` | ^8.18.0 | JSON 스키마 검증 |
| `yaml` | ^2.8.3 | YAML 파싱 |
| `jsonc-parser` | ^3.3.1 | JSONC 파싱 |
| `marked` | ^17.0.5 | Markdown 파싱 |
| `turndown` | ^7.2.2 | HTML -> Markdown 변환 |
| `diff` | ^8.0.4 | 텍스트 diff |
| `fuse.js` | ^7.1.0 | 퍼지 검색 |
| `picomatch` | ^4.0.4 | glob 매칭 |
| `ignore` | ^7.0.5 | .gitignore 스타일 필터 |
| `shell-quote` | ^1.8.3 | 셸 명령 파싱 |
| `plist` | ^3.1.0 | plist 파싱 (macOS) |
| `semver` | ^7.7.4 | 시맨틱 버전 비교 |
| `xss` | ^1.0.15 | XSS 방지 |
| `code-excerpt` | ^4.0.0 | 코드 발췌 |
| `lodash-es` | ^4.17.21 | 유틸리티 함수 |
| `lru-cache` | ^11.2.7 | LRU 캐시 |
| `p-map` | ^7.0.4 | 병렬 매핑 |
| `auto-bind` | ^5.0.1 | 자동 바인딩 |
| `env-paths` | ^4.0.0 | OS별 경로 |
| `stack-utils` | ^2.0.6 | 스택 트레이스 |
| `type-fest` | ^5.5.0 | 타입 유틸리티 |
| `usehooks-ts` | ^3.1.1 | React 훅 유틸리티 |
| `vscode-languageserver-protocol` | ^3.17.5 | LSP 프로토콜 |

### 8.7 선택적 의존성 (이미지 처리)

```json
"optionalDependencies": {
  "@img/sharp-darwin-arm64": "^0.34.2",
  "@img/sharp-darwin-x64": "^0.34.2",
  "@img/sharp-linux-arm64": "^0.34.2",
  "@img/sharp-linux-x64": "^0.34.2",
  "@img/sharp-win32-x64": "^0.34.2",
  "sharp": "^0.34.2"
}
```

모든 주요 플랫폼(macOS ARM/x64, Linux ARM/x64, Windows x64)에 대한 Sharp 네이티브 바인딩을 포함하여 이미지 리사이즈/처리를 지원한다.

### 8.8 개발 의존성

| 패키지 | 역할 |
|--------|------|
| `typescript` ^5.8.0 | TypeScript 컴파일러 |
| `esbuild` ^0.27.4 | 번들러 |
| `bun-types` ^1.2.0 | Bun 런타임 타입 |
| `@types/react` ^19.0.0 | React 타입 |
| `@types/node` ^22.0.0 | Node.js 타입 |
| 기타 `@types/*` | 타입 선언 |

### 8.9 의존성 분석 요약

- **총 dependencies**: 69개
- **총 optionalDependencies**: 10개 (sharp 플랫폼별)
- **총 devDependencies**: 8개
- **AI/API**: 6개 (Anthropic SDK, Bedrock, Vertex, Foundry, MCP)
- **클라우드**: 7개 (AWS, Azure, Google)
- **텔레메트리**: 10개 (OpenTelemetry + GrowthBook)
- **UI**: 16개 (React, chalk, highlight 등)
- **네트워크/파일**: 10개 (axios, undici, ws, chokidar 등)
- **유틸리티/파싱**: 20+개

이 의존성 구조는 Claude Code가 단순한 CLI 도구가 아닌, 다중 클라우드 프로바이더를 지원하고, 풍부한 터미널 UI를 제공하며, 실시간 텔레메트리와 feature flag를 갖춘 **엔터프라이즈급 에이전트 프레임워크**임을 보여준다.
