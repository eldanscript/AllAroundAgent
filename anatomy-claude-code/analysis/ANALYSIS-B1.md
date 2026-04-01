# Claude Code 분석 B1: 도구 시스템과 명령어 시스템

> 소스맵에서 복원된 `@anthropic-ai/claude-code` v2.1.88 (약 1,902개 파일, 512K+ 줄) 기반 분석
> Phase 1 아키텍처 문서 + Phase 2 리포지토리 분석(chatgptprojects, hangsman) 종합

---

## 6. 도구 시스템

### 6.1 Tool 인터페이스

모든 도구는 `src/Tool.ts`에 정의된 제네릭 타입 `Tool<Input, Output, Progress>`를 구현한다. 이 타입은 도구의 실행, 권한, UI 렌더링, 분류까지 포괄하는 **완전한 인터페이스**를 정의한다.

```typescript
export type Tool<
  Input extends AnyObject = AnyObject,
  Output = unknown,
  P extends ToolProgressData = ToolProgressData,
> = {
  // === 메타데이터 ===
  readonly name: string
  aliases?: string[]
  searchHint?: string
  readonly inputSchema: Input
  readonly inputJSONSchema?: ToolInputJSONSchema
  readonly shouldDefer?: boolean      // 지연 로딩 대상 여부
  readonly alwaysLoad?: boolean       // 항상 로드 여부
  readonly strict?: boolean           // 엄격 모드
  isMcp?: boolean                     // MCP 도구 여부
  isLsp?: boolean                     // LSP 도구 여부
  maxResultSizeChars: number          // 결과 최대 크기

  // === 핵심 실행 메서드 ===
  call(args, context, canUseTool, parentMessage, onProgress?): Promise<ToolResult<Output>>
  description(input, options): Promise<string>
  prompt(options): Promise<string>

  // === 권한 및 검증 ===
  checkPermissions(input, context): Promise<PermissionResult>
  validateInput?(input, context): Promise<ValidationResult>
  isEnabled(): boolean
  isReadOnly(input): boolean
  isDestructive?(input): boolean
  isConcurrencySafe(input): boolean
  interruptBehavior?(): 'cancel' | 'block'

  // === UI 렌더링 (React/Ink) ===
  renderToolUseMessage(input, options): React.ReactNode
  renderToolResultMessage?(content, progressMessages, options): React.ReactNode
  renderToolUseProgressMessage?(progressMessages, options): React.ReactNode
  userFacingName(input): string
  getToolUseSummary?(input): string | null
  getActivityDescription?(input): string | null

  // === 권한 매칭 및 분류 ===
  preparePermissionMatcher?(input): Promise<(pattern: string) => boolean>
  toAutoClassifierInput(input): unknown
  mapToolResultToToolResultBlockParam(content, toolUseID): ToolResultBlockParam
}
```

#### ToolUseContext (50+ 필드)

도구 실행 시 전달되는 컨텍스트 객체로, 파일시스템, 권한, 세션, 예산, MCP, 기능 플래그 등 **도구가 필요로 하는 모든 런타임 정보**를 포함한다.

```typescript
export type ToolUseContext = {
  // 옵션 그룹
  options: {
    commands: Command[]
    mainLoopModel: string
    tools: Tools
    thinkingConfig: ThinkingConfig
    mcpClients: MCPServerConnection[]
    // ...
  }

  // 제어
  abortController: AbortController

  // 파일시스템 상태
  readFileState: FileStateCache

  // 앱 상태 접근
  getAppState(): AppState
  setAppState(f: (prev: AppState) => AppState): void

  // 대화 이력
  messages: Message[]

  // 에이전트 식별
  agentId?: AgentId

  // ... 50+ 추가 필드
}
```

#### DeepImmutable<> 래핑

도구가 컨텍스트를 변경하는 것을 **타입 레벨에서 방지**하기 위해 `DeepImmutable` 유틸리티 타입으로 래핑한다. 이는 재귀적으로 모든 프로퍼티를 `readonly`로 변환한다.

```typescript
// DeepImmutable 유틸리티 타입 - 재귀적 불변 변환
type DeepImmutable<T> =
  T extends Map<infer K, infer V> ? ReadonlyMap<DeepImmutable<K>, DeepImmutable<V>> :
  T extends Set<infer V> ? ReadonlySet<DeepImmutable<V>> :
  T extends Array<infer V> ? ReadonlyArray<DeepImmutable<V>> :
  T extends object ? { readonly [K in keyof T]: DeepImmutable<T[K]> } :
  T;

// 사용 예: AppState는 DeepImmutable로 래핑
export type AppState = DeepImmutable<{
  settings: SettingsJson
  verbose: boolean
  mainLoopModel: ModelSetting
  // ...
}> & {
  // Mutable 필드 (의도적으로 DeepImmutable 제외)
  tasks: { [taskId: string]: TaskState }
  agentNameRegistry: Map<string, AgentId>
  mcp: { clients: MCPServerConnection[]; tools: Tool[]; /* ... */ }
  // ...
}
```

**설계 의도**: `DeepImmutable`은 함수 타입 필드를 제외한 모든 데이터를 불변으로 만들어, 도구 실행 중 상태 오염을 컴파일 타임에 차단한다. 단, `tasks`, `mcp` 등 런타임에 빈번하게 갱신되어야 하는 필드는 의도적으로 mutable로 유지된다.

---

### 6.2 buildTool() 팩토리

도구 정의(`ToolDef`)에서 완전한 `Tool` 인스턴스를 생성하는 팩토리 함수이다. 기본값을 주입하여 모든 도구가 일관된 인터페이스를 갖추도록 보장한다.

```typescript
export function buildTool<D extends AnyToolDef>(def: D): BuiltTool<D> {
  return {
    ...TOOL_DEFAULTS,            // 기본값 주입 (isEnabled: () => true 등)
    userFacingName: () => def.name,  // 기본 사용자 표시명 = 도구 이름
    ...def,                       // 사용자 정의로 기본값 덮어쓰기
  } as BuiltTool<D>
}
```

**팩토리 패턴의 효과**:
- 도구 개발자는 `name`, `inputSchema`, `call` 등 핵심 필드만 정의하면 된다
- `TOOL_DEFAULTS`가 `isEnabled`, `isReadOnly`, `isConcurrencySafe`, `renderToolUseMessage` 등의 기본 구현을 제공한다
- 스프레드 연산자로 기본값을 덮어쓰는 패턴으로, 선택적 오버라이드가 간단하다
- 타입 추론(`BuiltTool<D>`)으로 입력 스키마와 출력 타입이 도구 정의에서 자동으로 도출된다

---

### 6.3 빌트인 도구 전체 카탈로그

`src/tools/` 디렉토리에 구현된 **40+ 도구**를 카테고리별로 분류한다.

#### 파일 조작 도구

| 도구명 | 역할 | isConcurrencySafe | 비고 |
|--------|------|:-----------------:|------|
| **FileReadTool** | 파일 읽기 (이미지, PDF, 노트북 지원) | O | `imageProcessor.ts`, `limits.ts` 포함 |
| **FileWriteTool** | 새 파일 작성 또는 전체 덮어쓰기 | X | 상태 변경 도구 |
| **FileEditTool** | 정확한 문자열 교체 기반 편집 | X | `old_string` -> `new_string` 패턴 |
| **NotebookEditTool** | Jupyter 노트북 셀 편집 | X | `.ipynb` 전용 |

#### 검색 도구

| 도구명 | 역할 | isConcurrencySafe | 비고 |
|--------|------|:-----------------:|------|
| **GlobTool** | 파일 패턴 검색 (glob 매칭) | O | 읽기 전용 |
| **GrepTool** | ripgrep 기반 정규식 콘텐츠 검색 | O | 읽기 전용 |

#### 실행 도구

| 도구명 | 역할 | isConcurrencySafe | 비고 |
|--------|------|:-----------------:|------|
| **BashTool** | 셸 명령 실행 | X | 보안 검사 18개 파일 (bashSecurity.ts 2,592줄, bashPermissions.ts 2,621줄) |
| **PowerShellTool** | PowerShell 명령 실행 | X | Windows 환경 전용 |
| **REPLTool** | VM 기반 REPL 실행 | X | ANT(Anthropic 내부) 전용 |

#### 웹 도구

| 도구명 | 역할 | isConcurrencySafe | 비고 |
|--------|------|:-----------------:|------|
| **WebFetchTool** | URL 콘텐츠 가져오기 | O | 사전 승인 URL 목록(`preapproved.ts`) 포함 |
| **WebSearchTool** | 웹 검색 | O | 읽기 전용 |
| **WebBrowserTool** | 실제 브라우저 자동화 | X | `feature('WEB_BROWSER_TOOL')` 게이트 |

#### 에이전트/협업 도구

| 도구명 | 역할 | isConcurrencySafe | 비고 |
|--------|------|:-----------------:|------|
| **AgentTool** | 서브에이전트 생성/관리 | X | `runAgent.ts`, `forkSubagent.ts`, `builtInAgents.ts`, 5개 내장 에이전트 |
| **SendMessageTool** | 피어 에이전트에게 메시지 전송 | X | 코디네이터 모드에서 사용 |
| **TeamCreateTool** | 에이전트 팀(swarm) 생성 | X | `feature('AGENT_SWARMS')` 게이트 |
| **TeamDeleteTool** | 에이전트 팀 삭제 | X | `feature('AGENT_SWARMS')` 게이트 |
| **ListPeersTool** | 피어 목록 조회 | O | `feature('UDS_INBOX')` 게이트 |

#### 태스크 관리 도구

| 도구명 | 역할 | isConcurrencySafe | 비고 |
|--------|------|:-----------------:|------|
| **TodoWriteTool** | 할 일 목록 관리 (체크리스트) | X | 기본 TODO 시스템 |
| **TaskCreateTool** | 태스크 생성 | X | TodoV2 시스템, `feature('TODO_V2')` |
| **TaskGetTool** | 태스크 조회 | O | TodoV2 시스템 |
| **TaskUpdateTool** | 태스크 갱신 | X | TodoV2 시스템 |
| **TaskListTool** | 태스크 목록 | O | TodoV2 시스템 |
| **TaskStopTool** | 작업 중단 | X | 기본 도구 |
| **TaskOutputTool** | 백그라운드 태스크 출력 | O | 기본 도구 |

#### MCP 관련 도구

| 도구명 | 역할 | isConcurrencySafe | 비고 |
|--------|------|:-----------------:|------|
| **MCPTool** | MCP 프로토콜 도구 래퍼 | 동적 | MCP 서버에서 제공하는 도구를 래핑 |
| **McpAuthTool** | MCP OAuth 인증 처리 | X | |
| **ListMcpResourcesTool** | MCP 리소스 목록 조회 | O | 읽기 전용 |
| **ReadMcpResourceTool** | MCP 리소스 읽기 | O | 읽기 전용 |

#### 플랜/모드 도구

| 도구명 | 역할 | isConcurrencySafe | 비고 |
|--------|------|:-----------------:|------|
| **EnterPlanModeTool** | 플랜 모드 진입 (읽기 전용) | X | |
| **ExitPlanModeV2Tool** | 플랜 모드 종료 및 실행 전환 | X | |
| **EnterWorktreeTool** | Git worktree 생성/진입 | X | `isWorktreeModeEnabled()` 조건 |
| **ExitWorktreeTool** | Git worktree 종료 | X | `isWorktreeModeEnabled()` 조건 |

#### Kairos/프로액티브 도구

| 도구명 | 역할 | isConcurrencySafe | 비고 |
|--------|------|:-----------------:|------|
| **SleepTool** | 대기 (백그라운드 알림 수집) | X | `feature('PROACTIVE')` \| `feature('KAIROS')` |
| **SendUserFileTool** | 사용자에게 파일 능동 전송 | X | `feature('KAIROS')` |
| **PushNotificationTool** | 모바일/데스크톱 푸시 알림 | X | `feature('KAIROS')` \| `feature('KAIROS_PUSH_NOTIFICATION')` |
| **SubscribePRTool** | PR 이벤트 웹훅 구독 | X | `feature('KAIROS_GITHUB_WEBHOOKS')` |

#### 트리거/스케줄 도구

| 도구명 | 역할 | isConcurrencySafe | 비고 |
|--------|------|:-----------------:|------|
| **CronCreateTool** | 크론 스케줄 생성 | X | `feature('AGENT_TRIGGERS')` |
| **CronDeleteTool** | 크론 스케줄 삭제 | X | `feature('AGENT_TRIGGERS')` |
| **CronListTool** | 크론 스케줄 목록 | O | `feature('AGENT_TRIGGERS')` |
| **RemoteTriggerTool** | 원격 트리거 | X | `feature('AGENT_TRIGGERS_REMOTE')` |
| **MonitorTool** | 시스템/서비스 모니터링 | O | `feature('MONITOR_TOOL')` |

#### 기타 도구

| 도구명 | 역할 | isConcurrencySafe | 비고 |
|--------|------|:-----------------:|------|
| **AskUserQuestionTool** | 사용자에게 대화형 질문 | X | 기본 도구 |
| **SkillTool** | 슬래시 명령 스킬 실행 | X | 기본 도구 |
| **ToolSearchTool** | 지연 로딩된 도구 검색 (Fuse.js) | O | `isToolSearchEnabledOptimistic()` |
| **BriefTool** | 브리핑 생성 및 파일 업로드 | X | 기본 도구 |
| **LSPTool** | Language Server Protocol 통합 | O | `ENABLE_LSP_TOOL` 환경변수 |
| **SyntheticOutputTool** | JSON 스키마 기반 구조화 출력 | X | SDK 전용 |
| **SnipTool** | 히스토리 스닙 관리 | X | `feature('HISTORY_SNIP')` |
| **CtxInspectTool** | 컨텍스트 상태 검사 | O | `feature('CONTEXT_COLLAPSE')` |
| **TerminalCaptureTool** | 터미널 캡처 | X | `feature('TERMINAL_PANEL')` |
| **WorkflowTool** | 워크플로우 스크립트 실행 | X | `feature('WORKFLOW_SCRIPTS')` |
| **OverflowTestTool** | 오버플로우 테스트 | X | `feature('OVERFLOW_TEST_TOOL')` |
| **VerifyPlanExecutionTool** | 플랜 실행 검증 | X | `feature('CLAUDE_CODE_VERIFY_PLAN')` |
| **ConfigTool** | 시스템 설정 관리 | X | ANT 전용 |
| **TungstenTool** | tmux 세션 관리 (내부 인프라) | X | ANT 전용 |
| **SuggestBackgroundPRTool** | 백그라운드 PR 제안 | X | ANT 전용 |

---

### 6.4 3계층 등록

`src/tools.ts`의 `getAllBaseTools()` 함수에서 모든 도구를 3계층으로 등록한다.

#### Always Enabled (20+ 기본 도구)

환경과 무관하게 항상 활성화되는 핵심 도구 세트이다.

```typescript
export function getAllBaseTools(): Tools {
  return [
    AgentTool, TaskOutputTool, BashTool,
    ...(hasEmbeddedSearchTools() ? [] : [GlobTool, GrepTool]),
    ExitPlanModeV2Tool, FileReadTool, FileEditTool, FileWriteTool,
    NotebookEditTool, WebFetchTool, TodoWriteTool, WebSearchTool,
    TaskStopTool, AskUserQuestionTool, SkillTool, EnterPlanModeTool,
    BriefTool, ListMcpResourcesTool, ReadMcpResourceTool,
    // ...
  ]
}
```

포함 도구: `AgentTool`, `BashTool`, `FileReadTool`, `FileEditTool`, `FileWriteTool`, `GlobTool`, `GrepTool`, `NotebookEditTool`, `WebFetchTool`, `WebSearchTool`, `TodoWriteTool`, `TaskStopTool`, `TaskOutputTool`, `AskUserQuestionTool`, `SkillTool`, `EnterPlanModeTool`, `ExitPlanModeV2Tool`, `BriefTool`, `ListMcpResourcesTool`, `ReadMcpResourceTool`, `SendMessageTool`(lazy)

#### Conditionally Enabled (환경 의존)

런타임 환경, OS, 설정에 따라 조건부로 활성화되는 도구이다.

```typescript
// tools.ts 내 조건부 등록 패턴
...(isWorktreeModeEnabled() ? [EnterWorktreeTool, ExitWorktreeTool] : []),
...(isPowerShellToolEnabled() ? [PowerShellTool] : []),
...(isToolSearchEnabledOptimistic() ? [ToolSearchTool] : []),
...(isTodoV2Enabled() ? [TaskCreateTool, TaskGetTool, TaskUpdateTool, TaskListTool] : []),
...(isAgentSwarmsEnabled() ? [TeamCreateTool, TeamDeleteTool] : []),
```

| 도구 | 활성화 조건 |
|------|------------|
| EnterWorktreeTool / ExitWorktreeTool | `isWorktreeModeEnabled()` - Git 환경 |
| PowerShellTool | `isPowerShellToolEnabled()` - Windows 환경 |
| ToolSearchTool | `isToolSearchEnabledOptimistic()` - Beta 기능 |
| TaskCreate/Get/Update/ListTool | `isTodoV2Enabled()` - TodoV2 피처 |
| TeamCreate/DeleteTool | `isAgentSwarmsEnabled()` - 에이전트 스웜 |
| LSPTool | `ENABLE_LSP_TOOL` 환경변수 |

#### Feature-Flag Gated (15+ 미공개)

`feature()` 함수(Bun 번들러의 빌드타임 매크로)에 의해 게이트되는 미공개 도구이다. 외부 빌드에서는 dead code elimination으로 **코드 자체가 번들에서 제거**된다.

```typescript
// feature() 게이트 패턴
...(feature('KAIROS') ? [SleepTool, SendUserFileTool] : []),
...(feature('KAIROS_PUSH_NOTIFICATION') ? [PushNotificationTool] : []),
...(feature('KAIROS_GITHUB_WEBHOOKS') ? [SubscribePRTool] : []),
...(feature('AGENT_TRIGGERS') ? [...cronTools] : []),
...(feature('AGENT_TRIGGERS_REMOTE') ? [RemoteTriggerTool] : []),
...(feature('MONITOR_TOOL') ? [MonitorTool] : []),
...(feature('WEB_BROWSER_TOOL') ? [WebBrowserTool] : []),
...(feature('TERMINAL_PANEL') ? [TerminalCaptureTool] : []),
...(feature('CONTEXT_COLLAPSE') ? [CtxInspectTool] : []),
...(feature('HISTORY_SNIP') ? [SnipTool] : []),
...(feature('UDS_INBOX') ? [ListPeersTool] : []),
...(feature('WORKFLOW_SCRIPTS') ? [WorkflowTool] : []),
...(feature('OVERFLOW_TEST_TOOL') ? [OverflowTestTool] : []),
```

#### ANT 전용 (Anthropic 내부 빌드)

`USER_TYPE === 'ant'` 빌드 변형에서만 포함되는 내부 전용 도구이다.

| 도구명 | 역할 |
|--------|------|
| **REPLTool** | VM 기반 대화형 REPL 실행 |
| **ConfigTool** | 내부 시스템 설정 관리 |
| **TungstenTool** | tmux 세션 기반 내부 인프라 도구 |
| **SuggestBackgroundPRTool** | 백그라운드 PR 자동 제안 |

---

### 6.5 assembleToolPool 캐시 안정성

`assembleToolPool()`은 빌트인 도구와 MCP 도구를 병합하여 최종 도구 풀을 조립한다. **순서 안정성**이 핵심이다.

```typescript
export function assembleToolPool(
  permissionContext: ToolPermissionContext,
  mcpTools: Tools,
): Tools {
  // 1. 빌트인 도구: 이름순 정렬 (캐시 안정성)
  const sortedBuiltin = [...builtinTools].sort((a, b) =>
    a.name.localeCompare(b.name)
  );

  // 2. MCP 도구: 서버 이름 -> 도구 이름 순 2단계 정렬
  const sortedMCP = [...mcpTools].sort((a, b) => {
    const serverCmp = a.serverName.localeCompare(b.serverName);
    return serverCmp !== 0 ? serverCmp : a.name.localeCompare(b.name);
  });

  // 3. 빌트인 먼저, MCP 뒤에 연결
  return [...sortedBuiltin, ...sortedMCP];
}
```

**캐시 안정성이 중요한 이유**:
- 도구 목록은 시스템 프롬프트에 포함되어 API로 전송된다
- 도구 순서가 변하면 시스템 프롬프트 텍스트가 달라진다
- 시스템 프롬프트가 달라지면 **prompt cache hit율이 급락**한다
- 빌트인과 MCP를 분리 정렬하는 이유: MCP 도구 추가/제거가 빌트인 도구의 정렬 순서에 영향을 주지 않도록 격리한다

또한 `assembleToolPool`은 `permissionContext`의 deny 규칙을 적용하여 금지된 도구를 필터링한다.

---

### 6.6 StreamingToolExecutor

`src/services/tools/` 디렉토리에 구현된 도구 실행 엔진으로, **병렬/순차 실행을 자동 분류**한다.

#### 병렬 실행 메커니즘

```typescript
class StreamingToolExecutor {
  // 동시성 안전 도구: 병렬 실행 가능 (읽기 전용)
  private static readonly CONCURRENCY_SAFE_TOOLS = new Set([
    'FileRead', 'Glob', 'Grep', 'WebSearch', 'WebFetch'
  ]);

  // 상태 변경 도구: 순차 실행 필수
  private static readonly STATE_MUTATING_TOOLS = new Set([
    'FileWrite', 'Bash', 'FileEdit'
  ]);

  async executeTools(toolCalls: ToolCall[]): Promise<ToolResult[]> {
    const concurrent: ToolCall[] = [];
    const sequential: ToolCall[] = [];

    for (const call of toolCalls) {
      if (this.isConcurrencySafe(call.tool)) {
        concurrent.push(call);
      } else {
        sequential.push(call);
      }
    }

    // 읽기 전용 도구: Promise.all로 동시 실행
    const concurrentResults = await Promise.all(
      concurrent.map(call => this.executeTool(call))
    );

    // 상태 변경 도구: for 루프로 순차 실행
    const sequentialResults: ToolResult[] = [];
    for (const call of sequential) {
      sequentialResults.push(await this.executeTool(call));
    }

    return [...concurrentResults, ...sequentialResults];
  }
}
```

#### AbortReason 타입

도구 실행이 중단되는 3가지 사유를 명시적으로 구분한다.

```typescript
type AbortReason =
  | 'sibling_error'        // 동시 실행 도구 중 하나가 실패 -> 나머지 중단
  | 'user_interrupted'     // 사용자가 Ctrl+C로 중단
  | 'streaming_fallback';  // 스트리밍 오류로 폴백 모드 전환
```

#### 실시간 UI 진행률

```typescript
class ToolExecutionPipeline {
  private progressAvailableResolve: (() => void) | null = null;

  async execute(toolCalls: ToolCall[]): AsyncGenerator<ToolProgress | ToolResult> {
    for (const call of toolCalls) {
      const result = this.executeTool(call);

      // progressAvailableResolve Promise로 UI에 즉시 알림
      if (this.progressAvailableResolve) {
        this.progressAvailableResolve();
        this.progressAvailableResolve = null;
      }

      yield result;
    }
  }
}
```

---

### 6.7 Tool Search (Beta)

도구 수가 40+개를 넘으면서 모든 도구 스키마를 시스템 프롬프트에 포함하는 것은 토큰 낭비가 된다. Tool Search는 **필요할 때만 도구 스키마를 로딩**하는 지연 로딩 메커니즘이다.

```typescript
// ToolSearchTool: Fuse.js 기반 퍼지 검색
class ToolSearchTool implements Tool {
  name = 'ToolSearch';

  // 모든 도구의 메타데이터(이름, 설명)만 보유
  // 실제 실행 로직과 전체 스키마는 필요 시 동적 로딩
  private toolRegistry: Map<string, ToolMetadata> = new Map();

  inputSchema = z.object({
    query: z.string().describe('Query to find deferred tools...'),
    max_results: z.number().optional().default(5)
      .describe('Maximum number of results to return (default: 5)'),
  })

  async execute(params: { query: string; max_results?: number }): Promise<ToolSearchResult> {
    // Fuse.js로 쿼리와 매칭되는 도구 검색
    const matches = this.searchTools(params.query);

    // 매칭된 도구의 전체 스키마를 반환
    // -> 모델이 다음 턴에서 해당 도구를 호출할 수 있도록
    return {
      tools: matches.map(tool => ({
        name: tool.name,
        description: tool.description,
        parameters: tool.parameterSchema,
      })),
    };
  }
}
```

**지연 로딩 흐름**:
1. 도구에 `shouldDefer: true` 설정 -> 시스템 프롬프트에 전체 스키마 대신 이름만 포함
2. 모델이 `ToolSearch`를 호출하여 필요한 도구를 검색
3. 검색 결과로 전체 스키마가 반환되어 `<functions>` 블록에 추가
4. 모델이 다음 턴에서 해당 도구를 정상 호출

`shouldDefer`와 `alwaysLoad` 플래그로 도구별 로딩 전략을 제어한다:
- `shouldDefer: true` + `alwaysLoad: false` = 지연 로딩 (ToolSearch로만 접근)
- `shouldDefer: false` 또는 `alwaysLoad: true` = 즉시 로딩 (시스템 프롬프트에 포함)

---

## 7. 명령어 시스템

### 7.1 명령어 레지스트리

`src/commands.ts`에서 **80개 이상의 슬래시 명령어**를 등록한다. 각 명령어는 `src/commands/` 디렉토리의 개별 폴더에 구현되어 있다.

#### 일반 사용자 명령어

| 명령어 | 카테고리 | 타입 | 역할 |
|--------|---------|------|------|
| `/help` | 정보 | local-jsx | 도움말 표시 |
| `/clear` | 세션 | local | 대화 초기화, 캐시 삭제 |
| `/compact` | 세션 | local | 컨텍스트 수동 컴팩션 |
| `/resume` | 세션 | local-jsx | 이전 세션 복원 |
| `/model` | 설정 | local-jsx | 모델 선택/변경 |
| `/config` | 설정 | local-jsx | 설정 관리 |
| `/mcp` | 통합 | local-jsx | MCP 서버 추가/관리 |
| `/permissions` | 보안 | local-jsx | 권한 규칙 관리 |
| `/hooks` | 설정 | local-jsx | 후크 설정 |
| `/memory` | 컨텍스트 | local-jsx | CLAUDE.md 메모리 파일 편집 |
| `/plugin` | 통합 | local-jsx | 플러그인 관리 (마켓플레이스 탐색, 설치, 삭제) |
| `/skills` | 정보 | local-jsx | 스킬 목록 조회 |
| `/agents` | 에이전트 | local-jsx | 에이전트 관리 |
| `/context` | 정보 | local-jsx / local | 컨텍스트 상태 표시 |
| `/cost` | 정보 | local | 세션 비용 표시 |
| `/usage` | 정보 | local-jsx | 사용량 표시 |
| `/diff` | Git | local-jsx | 변경사항 diff 표시 |
| `/rewind` | Git | local | 파일 변경 되돌리기 |
| `/branch` | Git | local | Git 브랜치 관리 |
| `/session` | 세션 | local-jsx | 세션 정보/QR 코드 |
| `/export` | 세션 | local-jsx | 대화 내보내기 |
| `/copy` | 유틸리티 | local-jsx | 마지막 메시지 복사 |
| `/theme` | UI | local-jsx | 테마 변경 |
| `/vim` | UI | local | Vim 모드 토글 |
| `/plan` | 모드 | local-jsx | 플랜 모드 토글 |
| `/fast` | 모드 | local-jsx | 빠른 모드 토글 |
| `/effort` | 모드 | local-jsx | 노력 수준 설정 |
| `/thinkback` | 디버그 | local-jsx | 사고 과정 재생 |
| `/doctor` | 진단 | local-jsx | 환경 진단 |
| `/add-dir` | 파일 | local-jsx | 추가 작업 디렉토리 |
| `/tasks` | 태스크 | local-jsx | 백그라운드 태스크 관리 |
| `/stats` | 정보 | local-jsx | 통계 표시 |
| `/files` | 파일 | local | 추적 파일 목록 |
| `/status` | 정보 | local-jsx | 상태 표시 |
| `/feedback` | 소통 | local-jsx | 피드백 전송 |
| `/ide` | 통합 | local-jsx | IDE 통합 설치 |
| `/mobile` | 통합 | local-jsx | 모바일 QR 코드 |
| `/desktop` | 통합 | local-jsx | 데스크톱 앱 설치 |
| `/upgrade` | 관리 | local-jsx | 업그레이드 |
| `/login` | 인증 | local-jsx | 로그인 |
| `/logout` | 인증 | local-jsx | 로그아웃 |
| `/install-github-app` | 통합 | local-jsx | GitHub 앱 설치 (다단계 위자드) |
| `/install-slack-app` | 통합 | local | Slack 앱 설치 |
| `/sandbox-toggle` | 보안 | local-jsx | 샌드박스 모드 토글 |
| `/keybindings` | UI | local | 키바인딩 관리 |
| `/passes` | 정보 | local-jsx | 패스 정보 |
| `/privacy-settings` | 보안 | local-jsx | 프라이버시 설정 |
| `/rate-limit-options` | 설정 | local-jsx | 속도 제한 옵션 |
| `/release-notes` | 정보 | local-jsx | 릴리스 노트 |
| `/output-style` | UI | local-jsx | 출력 스타일 변경 |
| `/rename` | 세션 | local-jsx | 세션 이름 변경 |
| `/review` | Git | local-jsx | 코드 리뷰 |
| `/tag` | 세션 | local-jsx | 세션 태그 |
| `/stickers` | UI | local-jsx | 스티커 |
| `/exit` | 세션 | local | 종료 |

#### ANT(내부 전용) 명령어

```typescript
export const INTERNAL_ONLY_COMMANDS = [
  backfillSessions, breakCache, bughunter, commit, commitPushPr,
  ctx_viz, goodClaude, issue, initVerifiers, mockLimits,
  bridgeKick, version, resetLimits, onboarding, share, summary,
  teleport, antTrace, perfIssue, env, oauthRefresh, debugToolCall,
  agentsPlatform, autofixPr,
]
```

| 명령어 | 역할 |
|--------|------|
| `/commit`, `/commit-push-pr` | 내부 Git 워크플로우 |
| `/ctx_viz` | 컨텍스트 시각화 |
| `/good-claude` | 모델 피드백 |
| `/issue` | 이슈 관리 |
| `/bridge-kick` | Bridge 세션 강제 재시작 |
| `/teleport` | 세션 텔레포트 |
| `/ant-trace` | 내부 추적 |
| `/perf-issue` | 성능 이슈 리포트 |
| `/debug-tool-call` | 도구 호출 디버그 |
| `/autofix-pr` | PR 자동 수정 |
| `/mock-limits` / `/reset-limits` | 제한 테스트/리셋 |
| `/backfill-sessions` | 세션 백필 |
| `/break-cache` | 캐시 무효화 |

---

### 7.2 명령어 타입

명령어는 3가지 타입으로 구분된다.

#### prompt (프롬프트 타입)

모델에게 직접 전달되는 프롬프트 기반 명령이다. 스킬(Skills) 시스템이 이 타입을 사용한다.

```typescript
// 스킬은 prompt 타입 명령어로 등록
// SkillTool이 모델 응답 중 직접 호출 가능
type PromptCommand = {
  type: 'prompt'
  name: string
  description: string
  // 실행 시 프롬프트 텍스트가 모델 입력에 주입됨
  getPrompt(args: string): Promise<string>
}
```

스킬 소스:
- **번들 스킬**: `src/skills/bundled/` 디렉토리에서 로드
- **플러그인 스킬**: 플러그인 시스템에서 제공
- **사용자 스킬**: `.claude/skills/` 디렉토리에서 로드
- **MCP 스킬**: MCP 서버에서 제공 (`feature('MCP_SKILLS')`)

#### local (로컬 타입)

로컬에서 실행되고 텍스트 결과를 반환한다. 모델을 거치지 않는 순수 클라이언트사이드 명령이다.

```typescript
type LocalCommand = {
  type: 'local'
  name: string
  description: string
  // 텍스트 문자열을 반환
  execute(args: string, context: CommandContext): Promise<string>
}
```

예: `/clear`, `/compact`, `/cost`, `/vim`, `/exit`, `/branch`, `/files`

#### local-jsx (로컬 JSX 타입)

로컬에서 실행되고 React/Ink 컴포넌트를 렌더링한다. 대화형 UI가 필요한 명령에 사용된다.

```typescript
type LocalJSXCommand = {
  type: 'local-jsx'
  name: string
  description: string
  // React 컴포넌트를 반환하여 Ink로 터미널에 렌더링
  render(args: string, context: CommandContext): React.ReactNode
}
```

예: `/help`, `/config`, `/mcp`, `/plugin`, `/model`, `/resume`, `/doctor`

---

### 7.3 핵심 명령어 상세

#### /compact - 컨텍스트 수동 컴팩션

```
타입: local
동작: 현재 대화 이력을 Claude API로 요약하여 토큰 사용량 축소
```

대화가 길어져 컨텍스트 윈도우에 근접할 때 수동으로 트리거한다. 내부적으로는 Auto-Compact와 동일한 파이프라인(이미지 스트리핑 -> API 라운드 그룹핑 -> Thinking 블록 제거 -> Claude API 요약)을 실행한다.

#### /resume - 이전 세션 복원

```
타입: local-jsx
동작: 이전 대화 세션 목록을 표시하고 선택한 세션을 복원
```

`ResumeConversation.tsx` 화면을 렌더링하여 세션 목록을 보여주고, 선택 시 해당 세션의 메시지를 로드한다. 코디네이터 모드와 일반 모드의 세션을 구분한다 (`matchSessionMode()`).

#### /mcp - MCP 서버 관리

```
타입: local-jsx
동작: MCP 서버 추가, 제거, 상태 확인
```

Stdio, SSE, StreamableHTTP 트랜스포트를 지원하며, MCP 서버 연결 후 해당 서버의 도구, 리소스, 프롬프트를 Claude Code에 통합한다. OAuth 인증이 필요한 MCP 서버도 지원한다 (`services/mcp/auth.ts` - 2,465줄).

#### /plugin - 플러그인 관리

```
타입: local-jsx
동작: 마켓플레이스 탐색, 플러그인 설치/삭제, 활성화/비활성화
```

`pluginLoader.ts` (3,302줄)과 `marketplaceManager.ts` (2,643줄)이 핵심 로직을 담당한다. 플러그인은 도구, 명령어, 스킬을 제공할 수 있다.

#### /config - 설정 관리

```
타입: local-jsx
동작: 사용자 및 프로젝트 설정 편집
```

8개 소스 우선순위(userSettings > projectSettings > localSettings > flagSettings > policySettings > cliArg > command > session)에 따라 설정이 해석된다.

#### /help - 도움말

```
타입: local-jsx
동작: 사용 가능한 명령어, 도구, 키바인딩 목록 표시
```

#### /commit - 커밋 (스킬)

```
타입: prompt (스킬)
동작: Git 변경사항을 분석하여 커밋 메시지를 생성하고 커밋
```

`src/skills/bundled/` 디렉토리에서 로드되는 번들 스킬이다. SkillTool을 통해 모델이 직접 호출할 수도 있다.

---

### 7.4 미공개 명령어

피처 플래그 뒤에 숨겨진 미공개 명령어들이다. 60+개의 피처 플래그 중 명령어와 관련된 주요 항목:

| 명령어 | 피처 플래그 | 역할 |
|--------|-----------|------|
| `/voice` | `VOICE_MODE` | 음성 모드 활성화. OAuth 전용, push-to-talk STT. 킬스위치: `tengu_amber_quartz_disabled` |
| `/proactive` | `PROACTIVE` | 프로액티브 모드. 사용자 요청 없이 자발적 작업 수행 |
| `/buddy` | `BUDDY` | 컴패니언 캐릭터 시스템. `src/buddy/` 디렉토리에 구현 |
| `/ultraplan` | `ULTRAPLAN` | 울트라플랜. 고급 계획 수립 기능. `src/utils/ultraplan/` |
| `/torch` | `TORCH` | Torch 명령어 (상세 미공개) |
| `/peers` | `UDS_INBOX` | Unix Domain Socket 기반 피어 목록/메시징. 멀티디바이스 통신 |
| `/fork` | `FORK_SUBAGENT` | 서브에이전트 포크. 현재 대화를 분기하여 독립 에이전트로 실행 |
| `/chrome` | - | Chrome 관련 기능 (`src/commands/chrome/`) |
| `/bridge` | `BRIDGE_MODE` | Bridge 원격 제어 설정 |
| `/remote-setup` | `CCR_REMOTE_SETUP` | 원격 환경 설정 |
| `/remote-env` | - | 원격 환경 변수 |
| `/stickers` | - | 스티커 시스템 |
| `/thinkback-play` | - | 사고 과정 재생 (애니메이션) |
| `/insights` | - | 인사이트 분석 (`src/commands/insights.ts` - 3,200줄) |

#### Voice Mode 상세

```typescript
// voice/voiceModeEnabled.ts
export function isVoiceModeEnabled(): boolean {
  return hasVoiceAuth() && isVoiceGrowthBookEnabled()
}

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
```

네이티브 오디오 캡처 기반:
- **macOS/Linux/Windows**: `audio-capture-napi` (cpal 기반)
- **Fallback**: SoX `rec` 또는 `arecord` (ALSA)
- **설정**: 16kHz, mono, 2초 무음 감지, 3% 무음 임계값

#### Coordinator Mode 관련 명령어

코디네이터 모드가 활성화되면 워커 에이전트 관리 명령어가 사용 가능해진다:

```typescript
export function isCoordinatorMode(): boolean {
  if (feature('COORDINATOR_MODE')) {
    return isEnvTruthy(process.env.CLAUDE_CODE_COORDINATOR_MODE)
  }
  return false
}

// 코디네이터 전용 도구 집합
const INTERNAL_WORKER_TOOLS = new Set([
  TEAM_CREATE_TOOL_NAME,
  TEAM_DELETE_TOOL_NAME,
  SEND_MESSAGE_TOOL_NAME,
  SYNTHETIC_OUTPUT_TOOL_NAME,
])
```

---

## 부록: 도구/명령어 수치 요약

| 항목 | 수치 |
|------|------|
| 빌트인 도구 총수 | 40+ |
| Always Enabled 도구 | 20+ |
| Feature-Flag Gated 도구 | 15+ |
| ANT 전용 도구 | 4 (REPLTool, ConfigTool, TungstenTool, SuggestBackgroundPRTool) |
| 슬래시 명령어 총수 | 80+ |
| ANT 전용 명령어 | 24 |
| 명령어 타입 | 3 (prompt, local, local-jsx) |
| ToolUseContext 필드 | 50+ |
| 피처 플래그 총수 | 60+ |
| BashTool 보안 관련 파일 | 18개 (5,213줄+) |
| MCP 클라이언트 코드 | 3,348줄 (client.ts) + 2,465줄 (auth.ts) |
| 플러그인 로더 코드 | 3,302줄 (pluginLoader.ts) + 2,643줄 (marketplaceManager.ts) |
