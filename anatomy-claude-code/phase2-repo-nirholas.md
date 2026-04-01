# Phase 2: nirholas/claude-code 리포지토리 분석

> 분석 대상: `refs/nirholas-claude-code/`
> 원본: https://github.com/nirholas/claude-code
> 특징: 원본 소스코드 보존 + 웹 UI + MCP 서버 + 문서 + Docker/Helm/Grafana 배포 인프라를 갖춘 가장 풍부한 리포지토리

---

## 1. 리포지토리 개요

### 1.1 배경

2026-03-31 공개된 Claude Code npm 패키지의 `.map` 파일에서 추출된 전체 TypeScript 소스를 보존하는 리포지토리. 원본 소스는 `backup` 브랜치에 그대로 유지하고, 메인 브랜치에서는 웹 UI, MCP 서버, 문서, 배포 인프라를 추가로 구축했다.

README.md 서문:
```
The full source code of Anthropic's Claude Code CLI, made public on March 31, 2026
~1,900 files · 512,000+ lines of code
```

### 1.2 전체 디렉토리 구조

```
nirholas-claude-code/
├── src/                  # 원본 Claude Code 소스 (~1,900 파일, 512K+ lines)
├── web/                  # Next.js 웹 UI (추가 구현)
├── mcp-server/           # MCP 탐색 서버 (추가 구현, npm 배포)
├── docs/                 # 아키텍처 문서 5편 (추가 작성)
├── docker/               # Docker 배포 구성
├── helm/                 # Kubernetes Helm 차트
├── grafana/              # Grafana 모니터링 대시보드
├── scripts/              # 빌드/테스트/배포 스크립트
├── tests/                # 테스트
├── prompts/              # 프롬프트 파일
├── package.json          # 루트 패키지 (Bun 런타임)
├── tsconfig.json         # TypeScript 설정
├── biome.json            # Biome 린터/포매터
├── vitest.config.ts      # Vitest 테스트 설정
├── drizzle.config.ts     # Drizzle ORM (DB)
├── vercel.json           # Vercel 배포
├── Dockerfile            # 루트 Dockerfile
├── agent.md              # 에이전트 지침
├── Skill.md              # 스킬 문서
├── CONTRIBUTING.md       # 기여 가이드
└── README.md             # 상세 README (440줄)
```

### 1.3 기술 스택 요약

| 범주 | 기술 |
|------|------|
| 런타임 | Bun >= 1.1.0 |
| 언어 | TypeScript (strict) |
| 터미널 UI | React 19 + Ink |
| 웹 UI | Next.js + React + Tailwind CSS + Zustand |
| CLI 파싱 | Commander.js (`@commander-js/extra-typings`) |
| 스키마 검증 | Zod v3/v4 |
| 코드 검색 | ripgrep (GrepTool) |
| 프로토콜 | MCP SDK 1.12+ / LSP |
| API | Anthropic SDK 0.39+ |
| 텔레메트리 | OpenTelemetry + gRPC |
| 피처 플래그 | GrowthBook |
| 인증 | OAuth 2.0 / JWT / macOS Keychain |
| DB | Drizzle ORM + better-sqlite3 / PostgreSQL |
| 빌드 | esbuild (scripts/build-bundle.ts) |
| 테스트 | Vitest 4.1+ |
| 린터 | Biome |

---

## 2. src/ 핵심 코드 구조

### 2.1 핵심 파일

| 파일 | 예상 규모 | 역할 |
|------|-----------|------|
| `QueryEngine.ts` | ~46K lines | 코어 LLM API 엔진 - 스트리밍, 도구 루프, thinking 모드, 재시도, 토큰 카운팅 |
| `Tool.ts` | ~29K lines | 모든 도구의 기본 타입/인터페이스 - 입력 스키마, 권한, 진행 상태 |
| `commands.ts` | ~25K lines | 커맨드 등록 및 실행, 환경별 조건부 임포트 |
| `main.tsx` | - | CLI 파서 + React/Ink 렌더러, 병렬 프리페치로 시작 최적화 |
| `context.ts` | - | OS, 셸, git, 사용자 컨텍스트 수집 |
| `tools.ts` | - | 도구 레지스트리 |
| `cost-tracker.ts` | - | 토큰 비용 추적 |

### 2.2 핵심 파이프라인

```
User Input -> CLI Parser (main.tsx / Commander.js)
  -> REPL Launcher (replLauncher.tsx)
    -> Query Engine (QueryEngine.ts) -- 스트리밍, 도구 루프
      -> Anthropic API (services/api/)
      -> Tool Execution (tools/{ToolName}/)
      -> 결과를 API에 피드백 (루프)
    -> Terminal UI (React + Ink)
```

### 2.3 주요 서브디렉토리

| 디렉토리 | 파일 수 (추정) | 설명 |
|----------|---------------|------|
| `tools/` | ~40 도구 | 에이전트 도구 구현체 (BashTool, FileEditTool 등) |
| `commands/` | ~85 커맨드 | 슬래시 커맨드 구현체 |
| `components/` | ~140 컴포넌트 | Ink 터미널 UI 컴포넌트 |
| `services/` | 대규모 | 외부 통합 (API, MCP, OAuth, LSP, 분석 등) |
| `hooks/` | ~80 훅 | React 훅 (권한, IDE, 입력, 세션 등) |
| `bridge/` | 20+ 파일 | IDE 통합 (VS Code, JetBrains) |
| `coordinator/` | - | 멀티에이전트 오케스트레이션 |
| `plugins/` | - | 플러그인 시스템 |
| `skills/` | 16+ 번들 스킬 | 스킬 시스템 |
| `tasks/` | 6가지 타입 | 태스크 관리 |
| `state/` | - | 상태 관리 (AppStateStore) |
| `schemas/` | - | Zod 기반 설정 스키마 |
| `voice/` | - | 음성 입력 (VOICE_MODE 피처 플래그) |
| `vim/` | - | Vim 모드 |
| `server/` | - | 서버/원격 모드 |
| `remote/` | - | 원격 세션 관리 |
| `memdir/` | - | 영구 메모리 (CLAUDE.md) |
| `entrypoints/` | - | 초기화 로직 (cli.tsx, init.ts, mcp.ts, sdk/) |
| `query/` | - | 쿼리 파이프라인 (transitions, tokenBudget, stopHooks) |

### 2.4 도구 시스템 상세

모든 도구는 `buildTool()` 팩토리 패턴을 따른다:

```typescript
export const MyTool = buildTool({
  name: 'MyTool',
  aliases: ['my_tool'],
  inputSchema: z.object({ param: z.string() }),
  async call(args, context, canUseTool, parentMessage, onProgress) { ... },
  async checkPermissions(input, context) { ... },
  isConcurrencySafe(input) { ... },
  isReadOnly(input) { ... },
  prompt(options) { ... },
  renderToolUseMessage(input, options) { ... },
  renderToolResultMessage(content, progressMessages, options) { ... },
})
```

주요 도구 카테고리:

| 카테고리 | 도구 |
|----------|------|
| 파일 I/O | FileReadTool, FileWriteTool, FileEditTool, NotebookEditTool, GlobTool, GrepTool |
| 셸/실행 | BashTool, PowerShellTool, REPLTool |
| 에이전트/팀 | AgentTool, SendMessageTool, TeamCreateTool, TeamDeleteTool |
| 태스크 관리 | TaskCreateTool, TaskUpdateTool, TaskGetTool, TaskListTool, TaskOutputTool, TaskStopTool |
| 웹 | WebFetchTool, WebSearchTool |
| MCP | MCPTool, ListMcpResourcesTool, ReadMcpResourceTool, McpAuthTool, ToolSearchTool |
| 통합 | LSPTool, SkillTool |
| 스케줄링 | ScheduleCronTool, RemoteTriggerTool |
| 모드/상태 | EnterPlanModeTool, ExitPlanModeTool, EnterWorktreeTool, ExitWorktreeTool, SleepTool |

### 2.5 커맨드 시스템

세 가지 커맨드 타입:

| 타입 | 설명 | 예시 |
|------|------|------|
| PromptCommand | LLM에 포맷된 프롬프트 전송 + 도구 주입 | `/review`, `/commit` |
| LocalCommand | 인프로세스 실행, 텍스트 반환 | `/cost`, `/version` |
| LocalJSXCommand | 인프로세스 실행, React JSX 반환 | `/doctor`, `/install` |

~85개의 슬래시 커맨드가 등록되어 있으며, 주요 카테고리: Git, 코드 품질, 세션/컨텍스트, 설정, 메모리, MCP/플러그인, 인증, 태스크/에이전트, 진단, IDE 통합, 원격, 디버그.

### 2.6 서비스 레이어 (`services/`)

| 서비스 | 경로 | 역할 |
|--------|------|------|
| API 클라이언트 | `api/` | Anthropic SDK, 파일 API, 부트스트랩, 사용량 추적 |
| MCP | `mcp/` | MCP 클라이언트 연결, 도구 발견, 인증 |
| OAuth | `oauth/` | OAuth 2.0 인증 흐름 |
| LSP | `lsp/` | Language Server Protocol 관리자 |
| 분석 | `analytics/` | GrowthBook, Datadog, 텔레메트리 |
| Compact | `compact/` | 대화 컨텍스트 압축 (auto/micro/session) |
| 메모리 추출 | `extractMemories/` | 대화에서 자동 메모리 추출 |
| 팀 메모리 | `teamMemorySync/` | 팀 지식 동기화 + 비밀 스캐닝 |
| x402 | `x402/` | x402 결제 프로토콜 |
| Auto Dream | `autoDream/` | 백그라운드 아이디어 생성 |
| Tips | `tips/` | 컨텍스트 기반 사용 팁 |

### 2.7 피처 플래그 (Dead Code Elimination)

빌드 시점에 `bun:bundle`의 `feature()` 함수로 코드 가지치기:

```typescript
import { feature } from 'bun:bundle'
if (feature('VOICE_MODE')) { /* 비활성화 시 완전 제거 */ }
```

주요 플래그: `PROACTIVE`, `KAIROS`, `BRIDGE_MODE`, `DAEMON`, `VOICE_MODE`, `AGENT_TRIGGERS`, `MONITOR_TOOL`, `COORDINATOR_MODE`, `WORKFLOW_SCRIPTS`

---

## 3. web/ 디렉토리: Next.js 웹 UI

### 3.1 개요

원본 Claude Code의 터미널 Ink UI를 브라우저에서 실행할 수 있도록 포팅한 Next.js 웹 애플리케이션. 이것은 니르홀라스 리포에서 **자체 추가 구현**한 부분이다.

### 3.2 구조

```
web/
├── app/                     # Next.js App Router
│   ├── layout.tsx           # 루트 레이아웃
│   ├── page.tsx             # 메인 페이지 -> ChatLayout
│   ├── globals.css          # 글로벌 CSS
│   ├── ink-app/page.tsx     # Ink 앱 페이지
│   ├── accessibility/page.tsx
│   ├── share/[id]/page.tsx  # 세션 공유 페이지
│   └── api/                 # API 라우트 (14+ 엔드포인트)
│       ├── chat/route.ts          # 채팅 프록시 (Rate Limit, 키 스크럽)
│       ├── health/route.ts        # 헬스 체크
│       ├── env/route.ts           # 환경 변수
│       ├── exec/route.ts          # 명령 실행
│       ├── cwd/route.ts           # 작업 디렉토리
│       ├── export/route.ts        # 내보내기
│       ├── files/                 # 파일 CRUD
│       ├── fs/                    # 파일시스템 (12개 작업: read, write, list, mkdir, copy, rename, chmod, rm, stat, symlink, append, exists, readlink, realpath)
│       ├── analytics/             # 분석 이벤트/서머리
│       └── share/                 # 세션 공유
├── WebApp.tsx              # 루트 컴포넌트 (Vite/Next.js 양용)
├── middleware.ts           # Next.js 미들웨어
├── next.config.mjs         # Next.js 설정
├── tailwind.config.ts      # Tailwind CSS
├── app.tsx                 # Vite 엔트리
├── app.css                 # 앱 CSS
├── index.html              # Vite HTML
├── components/             # 100+ React 컴포넌트 (16개 카테고리)
├── hooks/                  # 18개 커스텀 훅
├── lib/                    # 라이브러리/유틸리티 (90+ 파일)
└── styles/                 # 터미널 테마 CSS (6개 테마)
```

### 3.3 컴포넌트 카테고리 (100+ 컴포넌트)

| 카테고리 | 파일 수 | 주요 컴포넌트 |
|----------|--------|--------------|
| `ui/` | 20+ | Button, Dialog, Tabs, ScrollArea, CommandPalette, ThemeToggle |
| `chat/` | 12 | ChatLayout, ChatWindow, ChatInput, MessageBubble, CodeBlock, ThinkingBlock, VirtualMessageList |
| `layout/` | 11 | Sidebar, Header, FileExplorer, FileTreeNode, ChatHistory, PerformanceObserver |
| `tools/` | 15 | ToolRouter, ToolBash, ToolFileEdit, ToolFileRead, ToolFileWrite, ToolGrep, ToolGlob, DiffView, AnsiRenderer |
| `settings/` | 9 | SettingsView, GeneralSettings, ModelSettings, McpSettings, PermissionSettings, ApiSettings |
| `collaboration/` | 8 | CollaborationProvider, PresenceAvatars, CursorGhost, AnnotationThread, ShareDialog, TypingIndicator |
| `auth/` | 6 | AuthProvider, LoginPage, ProtectedRoute, ApiKeyInput, OAuthButtons, UserMenu |
| `mobile/` | 6 | MobileSidebar, MobileHeader, MobileInput, BottomSheet, SwipeableRow, MobileFileViewer |
| `search/` | 6 | GlobalSearch, SearchInput, SearchFilters, SearchResults, DateRangePicker |
| `file-viewer/` | 8+ | FileViewer, FileEditor, DiffViewer, ImageViewer, FileBreadcrumb, CodeDisplay |
| `history/` | 6 | HistoryView, Timeline, CalendarHeatmap, ConversationTags, BulkActions, HistoryStats |
| `a11y/` | 5 | SkipToContent, FocusTrap, VisuallyHidden, Announcer, LiveRegion |
| `terminal/` | 4 | TerminalWindow, TerminalTitleBar, TerminalSpinner, CursorBlink |
| `input/` | 4 | WebTextInput, VimModeIndicator, InputHistory, TabCompletion |
| `command-palette/` | 2 | CommandPalette, CommandPaletteItem |
| `shortcuts/` | 2 | ShortcutsHelp, ShortcutBadge |
| `adapted/` | 9 | Ink 컴포넌트 웹 어댑터 (Message, Markdown, HighlightedCode, StatusLine 등) |
| `export/` | 4 | ExportDialog, ExportOptions, ExportPreview, FormatSelector |
| `notifications/` | 6 | NotificationCenter, ToastProvider, ToastStack 등 |
| `analytics/` | 3 | AdminDashboard, AnalyticsProvider, ConsentBanner |

### 3.4 lib/ 라이브러리 (90+ 파일)

| 모듈 | 파일 | 역할 |
|------|------|------|
| `store.ts` | 1 | Zustand 기반 글로벌 상태 관리 (대화, 설정, UI 상태) |
| `ink-compat/` | 12 | Ink -> 웹 호환 레이어 (Box, Text, useApp, useInput, useFocus 등) |
| `api/` | 6 | 백엔드 API 클라이언트 (client, stream, conversations, files, messages) |
| `platform/` | 7 | 플랫폼 추상화 (웹용 fs, exec, os, path, process shim) |
| `shims/` | 9 | Node.js API 웹 shim (fs, crypto, net, tls, readline 등) |
| `collaboration/` | 4 | 실시간 협업 (presence, socket, permissions) |
| `search/` | 3 | 클라이언트 검색 (client-search, highlighter, search-api) |
| `analytics/` | 4 | 분석 (batch, client, events, privacy) |
| `export/` | 4 | 내보내기 (html, json, markdown, plaintext) |
| `input/` | 6 | 입력 처리 (focus, history, ime, keyboard, paste, vim) |
| `performance/` | 3 | 성능 (metrics, streaming-optimizer, worker-pool) |
| `monitoring/` | 3 | 모니터링 (error-boundary, performance, web-vitals) |
| `workers/` | 3 | Web Worker (highlight, markdown, search) |

### 3.5 커스텀 훅 (18개)

```
useChat, useConversation, useConversations, useFiles, useStream,
useCollaboration, usePresence, useCommandRegistry, useKeyboardShortcuts,
useMediaQuery, useViewportHeight, useTouchGesture, useFocusReturn,
useReducedMotion, useAriaLive, useNotifications, useToast, useTheme
```

### 3.6 터미널 테마 CSS

6가지 테마: `amber.css`, `monokai.css`, `dracula.css`, `solarized-dark.css`, `green-screen.css`, `tokyo-night.css`

### 3.7 WebApp 루트 컴포넌트

`web/WebApp.tsx`의 프로바이더 계층:
```
ThemeProvider -> InkCompatProvider -> BackendProvider -> ConnectionStatusBanner -> ChatLayout
```

Vite와 Next.js 모두에서 동작하며, `VITE_API_URL` 또는 `NEXT_PUBLIC_API_URL` 환경변수로 백엔드 URL 설정.

### 3.8 상태 관리 (`web/lib/store.ts`)

Zustand + persist 미들웨어 사용. 기본 설정:
```typescript
{
  theme: "dark",
  terminalTheme: "tokyo-night",
  terminalEffects: { scanlines, glow, curvature, flicker },
  model: DEFAULT_MODEL,
  maxTokens: 8096,
  temperature: 1.0,
  permissions: { autoApprove: { file_read, file_write, bash, web_search }, restrictedDirs },
  mcpServers: [],
  keybindings: { ... }
}
```

---

## 4. mcp-server/: MCP 서버 구현

### 4.1 개요

Claude Code 소스를 탐색하기 위한 독립 MCP 서버. npm 패키지 `claude-code-explorer-mcp`로 배포됨.

```
mcp-server/
├── src/
│   ├── server.ts     # 핵심: createServer() 팩토리 (트랜스포트 비의존)
│   ├── index.ts      # STDIO 엔트리포인트
│   └── http.ts       # HTTP + SSE 엔트리포인트 (Express)
├── api/
│   ├── index.ts
│   └── vercelApp.ts  # Vercel 서버리스 배포
├── package.json      # npm: claude-code-explorer-mcp v1.1.0
├── Dockerfile        # 컨테이너 배포
├── railway.json      # Railway 배포
├── server.json       # MCP 서버 메타데이터
└── README.md
```

### 4.2 도구 8개

| 도구 | 설명 |
|------|------|
| `list_tools` | ~40개 에이전트 도구 목록 (소스 파일 포함) |
| `list_commands` | ~50개 슬래시 커맨드 목록 |
| `get_tool_source` | 특정 도구의 전체 소스 코드 읽기 |
| `get_command_source` | 특정 커맨드의 소스 코드 읽기 |
| `read_source_file` | src/ 내 임의 파일 읽기 (줄 범위 지원) |
| `search_source` | 정규식으로 소스 트리 전체 검색 |
| `list_directory` | src/ 하위 디렉토리 탐색 |
| `get_architecture` | 하이레벨 아키텍처 개요 생성 |

### 4.3 리소스 3개

| URI | 설명 |
|-----|------|
| `claude-code://architecture` | README.md 기반 아키텍처 개요 |
| `claude-code://tools` | 전체 도구 레지스트리 (JSON) |
| `claude-code://commands` | 전체 커맨드 레지스트리 (JSON) |

추가로 `claude-code://source/{path}` 리소스 템플릿으로 임의 소스 파일 접근 가능.

### 4.4 프롬프트 5개

| 프롬프트 | 인자 | 설명 |
|----------|------|------|
| `explain_tool` | toolName | 도구의 목적, 스키마, 권한, 실행 흐름 설명 |
| `explain_command` | commandName | 슬래시 커맨드 구현 설명 |
| `architecture_overview` | - | 전체 아키텍처 가이드 투어 |
| `how_does_it_work` | feature | 특정 서브시스템 설명 (18개 기능 매핑) |
| `compare_tools` | tool1, tool2 | 두 도구 비교 |

`how_does_it_work` 프롬프트는 내부적으로 기능-파일 매핑을 가지고 있다:
```typescript
const featureMap: Record<string, string[]> = {
  "permission system": ["utils/permissions/", "hooks/toolPermission/", "Tool.ts"],
  "mcp client": ["services/mcp/", "tools/MCPTool/", "tools/ListMcpResourcesTool/"],
  "query engine": ["QueryEngine.ts", "query/"],
  bridge: ["bridge/"],
  plugins: ["plugins/"],
  skills: ["skills/"],
  tasks: ["tasks.ts", "tasks/", "tools/TaskCreateTool/"],
  memory: ["memdir/", "commands/memory/"],
  voice: ["voice/"],
  // ... 총 18개 기능 매핑
}
```

### 4.5 트랜스포트

- **STDIO** (`index.ts`): 로컬 사용 (Claude Desktop, Claude Code)
- **Streamable HTTP** (`http.ts`): 원격 호스팅 (POST/GET `/mcp`)
- **Legacy SSE** (`http.ts`): 구형 클라이언트 지원 (GET `/sse`, POST `/messages`)
- **Vercel 서버리스** (`api/vercelApp.ts`)

HTTP 모드는 선택적 Bearer 토큰 인증 지원 (`MCP_API_KEY`).

### 4.6 안전장치

경로 탐색 방지:
```typescript
function safePath(relPath: string): string | null {
  const resolved = path.resolve(SRC_ROOT, relPath);
  if (!resolved.startsWith(SRC_ROOT)) return null;
  return resolved;
}
```

---

## 5. docs/ 문서 구조와 핵심 내용

5편의 상세 문서:

### 5.1 architecture.md - 아키텍처

핵심 파이프라인(Entrypoint -> Initialization -> QueryEngine -> Tool System -> Command System), 상태 관리(AppState + React Context), UI 레이어(~140 컴포넌트, 3개 스크린, ~80 훅), 빌드 시스템(Bun, 피처 플래그, 레이지 로딩), 동시성 모델(싱글 스레드 + 비동기 + React 동시 렌더링)을 다룬다.

### 5.2 tools.md - 도구 레퍼런스

~40개 에이전트 도구의 완전한 카탈로그. 카테고리: File System(7), Shell(3), Agent/Orchestration(10), Task Management(6), Web(2), MCP(5), Integration(2), Scheduling(2), Utility(3). 각 도구의 Read-Only 여부, 권한 모델, 도구 프리셋 설명.

### 5.3 commands.md - 커맨드 레퍼런스

~85개 슬래시 커맨드의 완전한 카탈로그. 14개 카테고리: Git(6), Code Quality(4), Session/Context(8), Config/Settings(13), Memory(3), MCP/Plugins(4), Auth(3), Tasks/Agents(4), Diagnostics(8), Install/Setup(6), IDE/Desktop(6), Remote(4), Misc(18), Internal/Debug(12).

### 5.4 subsystems.md - 서브시스템 가이드

10개 서브시스템 심층 분석: Bridge(IDE 통합, 15개 핵심 파일), MCP(클라이언트/서버 양면), Permission(4가지 모드), Plugin(라이프사이클 5단계), Skill(16개 번들 스킬), Task(6가지 태스크 타입), Memory(프로젝트/유저/추출/팀 4단계), Coordinator(멀티에이전트), Voice(5개 컴포넌트), Service Layer(18개 서비스).

### 5.5 exploration-guide.md - 탐색 가이드

코드베이스 탐색 실전 안내. 5가지 학습 경로(도구 E2E, UI, IDE 통합, 플러그인, MCP), 코드 패턴 인식(`buildTool()`, 피처 플래그, Anthropic 내부 게이트, 레이지 임포트, ESM .js 확장자), grep 패턴 8개 제공.

---

## 6. docker/, helm/, grafana/ 배포 인프라

### 6.1 Docker (`docker/`)

| 파일 | 용도 |
|------|------|
| `Dockerfile.all-in-one` | 올인원 컨테이너 (2-stage: Bun builder + runtime) |
| `Dockerfile` | 기본 Dockerfile |
| `docker-compose.yml` | 단일 서비스 개발/셀프호스팅용 |
| `docker-compose.prod.yml` | 프로덕션 멀티서비스 (claude-web + Caddy 리버스 프록시) |
| `Caddyfile` | Caddy 설정 (자동 HTTPS/Let's Encrypt) |
| `nginx.conf` | Nginx 설정 |
| `entrypoint.sh` | 컨테이너 엔트리포인트 |
| `.dockerignore` | 빌드 제외 파일 |

올인원 Dockerfile 특징:
- `oven/bun:1` 기반
- `node-pty` 네이티브 C++ 애드온 컴파일 (python3, make, g++)
- non-root 사용자 `claude` 생성
- `/data` 단일 볼륨으로 모든 영구 상태 관리
- 헬스 체크: `curl -f http://localhost:3000/health/live`

프로덕션 설정:
- Caddy 자동 TLS (HTTP/3 포함)
- 리소스 제한 (2 CPU, 1G 메모리)
- 용량 관리: `MAX_SESSIONS`, `MAX_SESSIONS_PER_USER`, `MAX_SESSIONS_PER_HOUR`
- 인증: token / oauth / apikey 3가지 모드

### 6.2 Helm (`helm/claude-code/`)

Kubernetes 배포용 Helm 차트:

```
helm/claude-code/
├── Chart.yaml              # appVersion: "latest", type: application
├── values.yaml             # 기본 설정값
└── templates/
    ├── _helpers.tpl        # 헬퍼 템플릿
    ├── deployment.yaml     # 디플로이먼트
    ├── service.yaml        # 서비스 (ClusterIP:80 -> 3000)
    ├── ingress.yaml        # 인그레스
    ├── hpa.yaml            # 오토스케일링 (2-10 replicas)
    ├── pdb.yaml            # Pod Disruption Budget
    ├── pvc.yaml            # Persistent Volume Claim (10Gi)
    ├── secret.yaml         # 시크릿 (API 키, 세션 시크릿)
    ├── configmap.yaml      # 설정
    └── serviceaccount.yaml # 서비스 어카운트
```

기본 설정값 (`values.yaml`):
- 오토스케일링: min 2 ~ max 10 replicas, CPU 70% / Memory 80% 기준
- PDB: minAvailable 1
- 리소스: requests 256Mi/250m, limits 512Mi/500m
- 보안: runAsNonRoot, allowPrivilegeEscalation=false, capabilities drop ALL
- 인그레스: WebSocket 지원 설정 (주석으로 안내)

### 6.3 Grafana (`grafana/`)

```
grafana/
├── dashboards/
│   ├── overview.json         # 전체 현황 대시보드
│   ├── conversations.json    # 대화 메트릭 대시보드
│   ├── costs.json            # 비용 추적 대시보드
│   └── infrastructure.json   # 인프라 모니터링 대시보드
└── provisioning/
    ├── dashboards.yaml       # 대시보드 프로비저닝
    └── datasources.yaml      # 데이터소스 (Prometheus + Loki)
```

데이터소스:
- **Prometheus**: 메트릭 수집 (15s 간격)
- **Loki**: 로그 수집 (maxLines: 1000)

---

## 7. 빌드 시스템과 스크립트

### 7.1 빌드 스크립트 (`scripts/build-bundle.ts`)

esbuild 기반 번들러. 주요 특징:

1. **단일 파일 출력**: `dist/cli.mjs` (코드 스플리팅 없음)
2. **플랫폼**: node20, es2022, ESM
3. **두 가지 커스텀 플러그인**:
   - `src-resolver`: `src/` 기반 bare import 해결 (tsconfig baseUrl)
   - `stub-missing`: 존재하지 않는 Anthropic 내부 모듈 스텁 처리
4. **빌드 시점 상수 주입**:
   ```typescript
   define: {
     'MACRO.VERSION': JSON.stringify(version),
     'process.env.USER_TYPE': '"external"',  // Anthropic 내부 코드 브랜치 제거
   }
   ```
5. **외부 패키지**: Node 빌트인, 네이티브 애드온 (node-pty, fsevents), AWS/Azure/GCP SDK, OpenTelemetry 익스포터 등 50+ 패키지
6. **후처리**: lodash-es UMD 감지 코드 패치 (Bun ESM 호환)

### 7.2 package.json 스크립트

| 스크립트 | 명령 | 용도 |
|----------|------|------|
| `build` | `bun scripts/build-bundle.ts` | 개발 빌드 |
| `build:prod` | `bun scripts/build-bundle.ts --minify` | 프로덕션 빌드 |
| `build:watch` | `bun scripts/build-bundle.ts --watch` | 감시 모드 빌드 |
| `build:web` | `bun scripts/build-web.ts` | 웹 UI 빌드 |
| `typecheck` | `tsc --noEmit` | 타입 검사 |
| `lint` | `biome check src/` | 린트 |
| `format` | `biome format --write src/` | 포맷 |
| `test` | `vitest run` | 테스트 |
| `db:generate` | `drizzle-kit generate` | DB 마이그레이션 생성 |
| `db:migrate` | `drizzle-kit migrate` | DB 마이그레이션 실행 |
| `db:seed` | 시드 | DB 시드 |
| `db:studio` | `drizzle-kit studio` | DB GUI |
| `package:npm` | `bun scripts/package-npm.ts` | npm 패키징 |

### 7.3 기타 스크립트

| 스크립트 | 용도 |
|----------|------|
| `scripts/build.sh` | 쉘 빌드 스크립트 |
| `scripts/ci-build.sh` | CI 빌드 |
| `scripts/dev.ts` | 개발 서버 |
| `scripts/install.sh` | 설치 |
| `scripts/backup.sh` | 백업 |
| `scripts/restore.sh` | 복원 |
| `scripts/post-deploy.sh` | 배포 후 처리 |
| `scripts/test-*.ts` | 개별 테스트 (auth, commands, ink, mcp, query, services, tools) |
| `scripts/bun-plugin-shims.ts` | Bun 플러그인 shim |
| `scripts/node-loader-hook.mjs` | Node.js 로더 훅 |

---

## 8. 다른 리포와 차별화되는 고유 콘텐츠

### 8.1 유일한 완전한 웹 UI 구현

이 리포만이 100+ 컴포넌트의 Next.js 웹 UI를 갖추고 있다. Ink 호환 레이어(`lib/ink-compat/`)를 통해 원본 터미널 컴포넌트를 브라우저에서 재사용할 수 있게 설계했다.

고유 기능:
- **실시간 협업**: `components/collaboration/` (PresenceAvatars, CursorGhost, AnnotationThread)
- **모바일 지원**: `components/mobile/` (BottomSheet, SwipeableRow, 전용 Header/Sidebar/Input)
- **접근성**: `components/a11y/` (SkipToContent, FocusTrap, LiveRegion, Announcer)
- **커맨드 팔레트**: VS Code 스타일 커맨드 검색/실행
- **파일 뷰어/에디터**: 내장 코드 에디터, Diff 뷰어, 이미지 뷰어
- **히스토리 시각화**: CalendarHeatmap, Timeline, ConversationTags
- **6가지 터미널 테마** + 스캔라인/글로우/곡률/플리커 효과
- **Web Workers**: highlight, markdown, search 병렬 처리

### 8.2 유일한 MCP 탐색 서버 (npm 배포)

`claude-code-explorer-mcp`로 npm에 배포된 유일한 MCP 서버. 도구 8개 + 리소스 3개 + 프롬프트 5개의 완전한 MCP 구현. STDIO/HTTP/SSE/Vercel 4가지 트랜스포트 지원.

### 8.3 유일한 프로덕션급 배포 인프라

- **Docker**: 개발/프로덕션 2가지 Docker Compose + Caddy 자동 TLS
- **Helm**: 오토스케일링, PDB, PVC, Ingress가 포함된 Kubernetes 차트
- **Grafana**: 4개 대시보드 (overview, conversations, costs, infrastructure) + Prometheus/Loki

### 8.4 가장 상세한 문서

5편의 체계적 문서 (architecture, tools, commands, subsystems, exploration-guide)는 다른 리포에 없는 수준의 깊이를 제공한다. 특히 exploration-guide.md의 5가지 학습 경로와 grep 패턴은 코드베이스 탐색에 실질적으로 유용하다.

### 8.5 실제 빌드 가능한 빌드 시스템

`scripts/build-bundle.ts`는 esbuild 기반으로 실제 빌드가 가능하도록 설계되었다. Anthropic 내부 모듈의 스텁 처리, `bun:bundle` shim, `MACRO.*` 상수 주입, 후처리(lodash-es ESM 패치)까지 포함하여, 원본 소스를 외부 환경에서 빌드할 수 있게 했다.

### 8.6 DB 통합

Drizzle ORM + better-sqlite3/PostgreSQL 지원으로, 서버 모드에서의 세션/사용자 데이터 영구 저장을 위한 DB 레이어를 갖추고 있다.

### 8.7 고유 서비스 구현

원본 `src/` 코드 외에도 `src/server/` 디렉토리에 PTY 서버, 보안(rate-limiter, key-encryption), 헬스 체크 등 웹 서비스 인프라가 구현되어 있다.

---

## 요약

nirholas/claude-code 리포지토리는 다음 계층으로 구성된 가장 포괄적인 프로젝트:

1. **원본 소스 보존** (`src/`): 1,900 파일, 512K+ lines의 Claude Code 소스
2. **웹 UI** (`web/`): 100+ 컴포넌트의 Next.js 앱으로 브라우저에서 Claude Code 경험 제공
3. **MCP 서버** (`mcp-server/`): npm 배포 가능한 소스 탐색 MCP 서버
4. **문서** (`docs/`): 5편의 체계적 아키텍처/레퍼런스/가이드 문서
5. **배포 인프라** (`docker/`, `helm/`, `grafana/`): Docker부터 Kubernetes까지 프로덕션급 배포
6. **빌드 시스템** (`scripts/`): 외부 환경에서 빌드 가능한 esbuild 기반 빌드 파이프라인
