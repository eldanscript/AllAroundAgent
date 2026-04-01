# Phase 2: leaf-kit/claude-analysis 리포지토리 분석

## 1. 리포지토리 개요

**leaf-kit/claude-analysis**는 Claude Code CLI 소스코드(1,902개 TypeScript/TSX 파일)를 정적 분석하여 **Obsidian 호환 마크다운 문서 시스템**으로 구성한 프로젝트이다. 총 60개의 `.md` 파일(최상위 14개 + 하위 디렉터리 46개)로 구성되며, 이 중 분석 문서 약 39개와 튜토리얼 19장(README 포함 20개 파일)을 포함한다.

핵심 특징:
- 모든 문서가 `[[위키링크]]`로 상호 연결되어 Obsidian의 **Graph View**에서 모듈 간 관계를 시각적으로 탐색 가능
- 실제 소스코드(`./src/` 디렉터리)를 함께 포함하여 분석 문서와 원본 코드를 교차 참조 가능
- **Map of Content(MOC)** 패턴을 적용한 체계적 지식 관리 구조
- Mermaid 다이어그램 29개로 아키텍처를 시각화

README.md에서 소스코드 유출 경위를 명시하고 있다:

> "Anthropic이 npm에 배포한 `@anthropic-ai/claude-code` 패키지에 소스맵 파일(`.map`)이 포함되어 있었습니다. 해당 `.map` 파일에는 난독화되지 않은 원본 TypeScript 소스코드의 참조 경로가 기록되어 있었으며... 완전한 소스코드가 ZIP 아카이브 형태로 직접 다운로드 가능한 상태였습니다."

---

## 2. 분석 문서 구조

### Index.md -- Map of Content (MOC)

`Index.md`는 전체 지식 지도의 허브 역할을 한다. Obsidian 프론트매터(`tags: [MOC, index, architecture]`)를 포함하며, 다음을 제공한다:

- **아키텍처 개요 다이어그램**: 사용자 입력 -> Entrypoints -> REPL Screen -> Query Engine/Commands/Ink -> Tools -> Services -> Utils 순서의 계층 구조를 ASCII 아트로 표현
- **19개 섹션별 핵심 문서 링크**: 진입점, 쿼리 엔진, 도구 시스템, 서비스 레이어, UI 컴포넌트, Ink 프레임워크, 유틸리티, 훅 시스템, 명령어 시스템, 부트스트랩, 코디네이터, 키바인딩, 메모리, 마이그레이션, 네이티브 TS, 화면 시스템, 서버, 기타 모듈, 루트 파일
- **4가지 핵심 데이터 흐름**: 초기화 흐름, 쿼리 실행 흐름, 도구 실행 흐름, MCP 연결 흐름
- **6가지 설계 원칙**: 레이어드 아키텍처, 권한 우선, 확장 가능성, React 기반 터미널 UI, 멀티에이전트 지원, 멀티프로바이더 API

### README.md (235KB)

README.md는 약 235KB의 대형 문서로, 다음 내용을 포함한다:

- 소스코드 유출 배경(npm 소스맵 사건)과 기술적/커뮤니티 인사이트
- 프로젝트 규모 요약 (1,902 파일, ~160 디렉터리, 40+ 도구, 100+ 명령어, 19 서비스)
- 5-Layer Architecture를 Mermaid로 시각화 (User Input -> Entrypoint -> Engine -> Execution -> Service -> Infrastructure)
- 각 모듈 상세 분석 (13개 섹션)
- **Prompt & Agent Architecture** 섹션: 시스템 프롬프트 조립 순서, 컨텍스트 주입 흐름, API 요청 조립, 6종 내장 에이전트 타입, Coordinator 모드, CLAUDE.md 로딩 체인 등
- 소스코드 브라우즈 가이드 및 Obsidian 사용법

### 각 Overview 문서

모든 Overview 문서는 동일한 구조를 따른다:
1. Obsidian 프론트매터 (tags, created, type, status)
2. 모듈 아키텍처 다이어그램 (ASCII 또는 Mermaid)
3. 파일별 상세 분석 테이블
4. 실행 흐름 다이어그램
5. 통계 요약
6. `[[위키링크]]` 기반 관련 문서 연결

---

## 3. Components_Overview.md 핵심 내용

UI 컴포넌트 시스템은 **총 389개 파일**로 구성되며, React + Ink 프레임워크 기반 터미널 UI를 제공한다.

**주요 컴포넌트 카테고리:**

| 카테고리 | 파일 수 | 핵심 내용 |
|----------|---------|-----------|
| 메시지 시스템 | 36 | AssistantTextMessage, AssistantToolUseMessage(~45KB), UserToolResultMessage 등 |
| 권한 다이얼로그 | 32 | PermissionRequest(~33KB), BashPermissionRequest, FilePermissionDialog 등 도구별 권한 컴포넌트 |
| 디자인 시스템 | 18 | Dialog, FuzzyPicker(~40KB), Tabs(~41KB), ThemedBox/Text, ThemeProvider |
| 에이전트 UI | 16 | 에이전트 관리 컴포넌트 |
| MCP UI | 15 | MCP 서버 UI 컴포넌트 |

**컴포넌트 렌더링 계층:**

```
REPL.tsx (메인 화면)
  -> FullscreenLayout
       -> ScrollBox (스크롤 가능 영역) -> Message 컴포넌트들 + AgentProgressLine
       -> PromptInput (하단 고정 입력) -> BaseTextInput + Footer + Notifications
       -> StatusLine (상태바)
  -> 모달 다이얼로그 (PermissionRequest, QuickOpenDialog, DiffDialog)
```

특히 `SystemTextMessage.tsx`(~79KB), `AttachmentMessage.tsx`(~71KB), `ContextVisualization.tsx`(~76KB) 등 매우 대형 컴포넌트가 존재하며, 이는 복잡한 터미널 렌더링 로직의 규모를 보여준다.

---

## 4. Tools_Overview.md 핵심 내용

Claude Code는 **40개 도구**를 제공하며, 모든 도구는 `Tool<Input, Output, Progress>` 인터페이스를 구현한다.

**Tool 인터페이스의 핵심 메서드:**

```typescript
interface Tool<Input, Output, Progress> {
  name: string;
  description(): string;
  call(input: Input, context: ToolUseContext): Promise<Output>;
  checkPermissions(input: Input, context: ToolPermissionContext): PermissionResult;
  isReadOnly(): boolean;
  isConcurrencySafe(): boolean;
  isDestructive(): boolean;
}
```

**도구 카테고리 분포:**

| 카테고리 | 도구 수 | 주요 도구 |
|----------|---------|-----------|
| 파일 시스템 | 5 | FileRead, FileEdit, FileWrite, Glob, Grep |
| 태스크 관리 | 6 | TaskCreate/Update/List/Get/Stop/Output |
| MCP | 4 | MCPTool, ListMcpResources, ReadMcpResource, McpAuth |
| 에이전트/멀티에이전트 | 3 | AgentTool, SendMessage, Skill |
| 워크트리/플랜 | 4 | EnterWorktree, ExitWorktree, EnterPlanMode, ExitPlanMode |
| 스케줄링 | 3 | ScheduleCron, CronDelete, CronList |
| 셸 실행 | 2 | Bash, PowerShell |
| 웹 | 2 | WebFetch, WebSearch |
| 코드 인텔리전스 | 1 | LSP |
| 기타 특수 | 10 | Config, NotebookEdit, RemoteTrigger, Brief, AskUserQuestion 등 |

**도구 실행 흐름 핵심:**
1. 모델 응답에서 `tool_use` 블록 추출
2. `findToolByName`으로 도구 조회
3. `Tool.checkPermissions()` -> 권한 시스템 연동
4. `StreamingToolExecutor`가 동시성 안전 도구는 병렬 실행, 비안전 도구는 직렬 실행

**공통 패턴:** Zod v4 스키마 검증, 지연 스키마 평가(임포트 사이클 방지), Feature Flag 기반 가용성, `maxResultSizeChars`로 출력 크기 제한

---

## 5. Services_Overview.md 핵심 내용

서비스 레이어는 **19개 서비스, 130개 파일**로 구성된다.

**10대 핵심 서비스:**

| 서비스 | 핵심 역할 | 주요 특징 |
|--------|-----------|-----------|
| API Service | Claude API 통신 | **4개 프로바이더** (Anthropic Direct, AWS Bedrock, Google Vertex AI, Azure Foundry) |
| MCP Service | MCP 서버 연결 관리 | **4개 전송 프로토콜** (stdio, SSE, WebSocket, HTTP) |
| Compact Service | 컨텍스트 윈도우 압축 | Full Compact, Micro Compact, Session Memory 3가지 전략 |
| Tools Service | 도구 실행 오케스트레이션 | Async Generator 기반 스트리밍, 병렬/직렬 분류 |
| LSP Service | Language Server Protocol | 정의 이동, 참조 찾기, 호버, 심볼, 호출 계층 |
| OAuth Service | PKCE 기반 OAuth 2.0 | 브라우저 열기 -> 콜백 수신 -> 토큰 교환 |
| Analytics Service | 이벤트 로깅 | 제로 의존성, Datadog + 1P 싱크 |
| SessionMemory | 세션 메모리 | 포크된 서브에이전트가 백그라운드에서 메모리 파일 업데이트 |
| ExtractMemories | 메모리 추출 | 쿼리 루프 종료 시 대화에서 지속 메모리 추출 |
| Plugins Service | 플러그인 라이프사이클 | 마켓플레이스 통합, 의존성 해석, 정책 기반 차단 |

**기타 서비스:** tips(팁 레지스트리), policyLimits(레이트 리밋), remoteManagedSettings(엔터프라이즈 원격 설정), settingsSync, AgentSummary, autoDream, PromptSuggestion, MagicDocs, teamMemorySync, toolUseSummary

---

## 6. tutorial/ 19장 각 주제 요약

튜토리얼은 "초등학생부터 시니어 엔지니어까지" 대상으로, 초급/중급/고급/특별편의 4단계로 구성된다.

### 초급 (1-4장) -- 기초 다지기

| 장 | 제목 | 핵심 내용 |
|----|------|-----------|
| 1장 | 우리의 똑똑한 친구, Claude | Claude와 Claude Code CLI 소개, API 프로바이더 개념을 피자 주문 비유로 설명 |
| 2장 | 스스로 생각하는 에이전트 | ReAct(Reasoning and Acting) 패턴, 40개 도구, 6개 내장 에이전트 유형 |
| 3장 | AI를 움직이는 프롬프트 마법 | 15개 프롬프트 블록, CLAUDE.md 계층 구조, 시스템/메시지/도구의 "택배 상자" 비유 |
| 4장 | 실제 코드로 보는 구조 | 5-Stop 소스코드 가이드 투어, 핵심 파일과 개념의 연결 |

### 중급 (5-8장) -- 시스템 이해

| 장 | 제목 | 핵심 내용 |
|----|------|-----------|
| 5장 | AI의 기억 시스템 | 세션/자동/팀 메모리, 추출 에이전트, MEMORY.md 구조 |
| 6장 | 보안과 권한 시스템 | 3층 방어 구조, 12단계 권한 흐름, tree-sitter 기반 AST Bash 보안 |
| 7장 | MCP와 확장성 | MCP 4가지 전송 프로토콜, 스킬 시스템, 플러그인 아키텍처 |
| 8장 | 터미널 렌더링 엔진 | React -> Yoga -> ANSI 6단계 파이프라인, 더블 버퍼링 |

### 고급 (9-12장) -- 깊은 이해

| 장 | 제목 | 핵심 내용 |
|----|------|-----------|
| 9장 | 상태 관리와 글로벌 스토어 | 209개 getter/setter 싱글톤, 비용 추적, 반응적 Store |
| 10장 | 컨텍스트 압축과 토큰 관리 | 9항목 보존 전략, Full/Micro/Session Memory 3가지 압축 전략 |
| 11장 | 화면 시스템과 세션 관리 | REPL(4,500줄), 세션 재개, 진단 화면, Vim/음성 모드, Buddy |
| 12장 | 고급 패턴과 내부 최적화 | 8대 설계 패턴(Factory, Async Generator, Singleton Store 등), 동시성, 네이티브 TS 포트 |

### 특별편 (13-19장)

| 장 | 제목 | 핵심 내용 |
|----|------|-----------|
| 13장 | 소스코드에 숨겨진 비밀들 | autoDream, Speculation, YOLO, Buddy, Tengu 등 12개 미공개 기능 |
| 14장 | 실행 흐름 완전 해부 | Enter부터 응답까지 15단계, 루프 표시 포함 실전 예시 |
| 15장 | 리버스 엔지니어링 | 코드 스니펫 10개로 5대 설계 철학 역공학 분석 |
| 16장 | 다른 코드 에이전트와의 비교 | Codex, Cursor, Copilot, Aider, Devin과의 아키텍처 비교 |
| 17장 | 프롬프트 아키텍처 완전 해부 | 54개 이상 프롬프트 파일 전수 분석, 지시문 원문 해부 |
| 18장 | 도메인별 에이전트 적용 전략 | 웹/데이터/DevOps/보안/교육 등 8개 분야 적용 기법 |
| 19장 | Anthropic만의 비법 26가지 | 캐시 7중 최적화, DCE, Undercover, 투기적 실행, Auto-Dream 등 |

---

## 7. Stats_Report.md 통계 분석 결과

### 프로젝트 규모

| 지표 | 값 |
|------|-----|
| 총 파일 수 | 1,902 |
| 총 디렉터리 수 | ~160 |
| 주요 언어 | TypeScript (.ts, .tsx) |
| UI 프레임워크 | React (Ink 터미널 렌더링) |
| 빌드 도구 | Bun |

### 모듈별 파일 분포 (상위 10)

| 순위 | 모듈 | 파일 수 | 비율 |
|------|------|---------|------|
| 1 | utils/ | 564 | 29.7% |
| 2 | components/ | 389 | 20.5% |
| 3 | commands/ | 207 | 10.9% |
| 4 | tools/ | 184 | 9.7% |
| 5 | services/ | 130 | 6.8% |
| 6 | hooks/ | 104 | 5.5% |
| 7 | ink/ | 96 | 5.0% |
| 8 | bridge/ | 31 | 1.6% |
| 9 | constants/ | 21 | 1.1% |
| 10 | skills/ | 20 | 1.1% |

### 위키링크 기반 연결 밀도

| 모듈 | 들어오는 링크 | 나가는 링크 | 총 연결 |
|------|--------------|------------|---------|
| Index (MOC) | 0 | 25 | 25 |
| Tools_Overview | 12 | 8 | 20 |
| Query_Engine | 8 | 10 | 18 |
| Permissions_System | 10 | 4 | 14 |
| Services_Overview | 8 | 6 | 14 |

### 모듈 응집도 평가

| 모듈 | 응집도 | 이유 |
|------|--------|------|
| Ink Framework | 5/5 | 독립적 프레임워크, 명확한 경계 |
| Tools | 4/5 | 일관된 인터페이스, 도구별 격리 |
| Services | 4/5 | 서비스별 명확한 책임 |
| Utils | 3/5 | 다양한 유틸, 일부 크로스커팅 |
| Components | 3/5 | 대형 컴포넌트 존재 |

### 핵심 외부 의존성

`@anthropic-ai/sdk`, `@anthropic-ai/bedrock-sdk`, `@anthropic-ai/vertex-sdk`, `@anthropic-ai/foundry-sdk` (API 4종), `@modelcontextprotocol/sdk` (MCP), `react` + `yoga-layout` (UI), `zod` v4 (스키마), `tree-sitter` (AST 파싱), `vscode-languageserver-protocol` (LSP)

### 보안 아키텍처 통계

| 보안 계층 | 파일 수 | 기법 |
|-----------|---------|------|
| 권한 시스템 | 26 | 규칙 기반 + AI 분류 |
| Bash 보안 | 15+ | AST 파싱, 인젝션 방지 |
| 셸 검증 | 12 | 읽기전용 명령 레지스트리 |
| 경로 검증 | 5+ | 안전 경로 체크 |
| 샌드박싱 | 2 | @anthropic-ai/sandbox-runtime |

### 2차 분석에서 발견된 추가 모듈

| 모듈 | 줄 수 | 핵심 역할 |
|------|-------|-----------|
| bootstrap/state | 1,758줄 | 글로벌 상태 싱글톤 (209 getter/setter) |
| coordinator | 369줄 | 멀티워커 에이전트 오케스트레이션 |
| keybindings | 3,159줄 | 커스텀 키바인딩 + 코드 시퀀스 지원 |
| native-ts | 4,081줄 | color-diff, file-index, yoga-layout Rust/C++ -> TS 포트 |
| screens | 5,977줄 | REPL, 세션재개, 진단 화면 |
| buddy | 1,298줄 | 컴패니언 캐릭터 절차적 생성 |
| memdir | 1,736줄 | 파일 기반 영속 메모리 (AI 선택) |

---

## 8. 다른 분석 리포 대비 고유한 인사이트

### 8.1 Obsidian 네이티브 지식 관리 시스템

다른 분석 리포들이 단순 마크다운 파일 모음인 반면, leaf-kit은 **Obsidian의 위키링크(`[[]]`), 프론트매터, Graph View를 정식 활용한 지식 관리 시스템**으로 설계되었다. Index.md가 Map of Content(MOC) 역할을 하며, 총 연결 밀도(위키링크 기반)를 Stats_Report에서 정량적으로 측정한 것이 독특하다.

### 8.2 소스코드 유출 경위의 기술적 분석

README에서 npm 소스맵 유출 사건을 단순 언급이 아닌 기술적으로 분석하고 있다:
- `cli.js.map`(57MB)의 `sources[]` + `sourcesContent[]` 배열 구조
- 4,756개 소스 파일 중 Claude Code 자체 1,906개 + node_modules 2,850개
- Anthropic이 1년 전에도 동일 실수를 했다는 커뮤니티 인사이트

### 8.3 19장 계층형 튜토리얼

초등학생 수준의 비유(피자 주문, 성의 3겹 성벽)부터 시니어 엔지니어 수준의 코드 스니펫 역공학까지, **19장의 단계적 튜토리얼**은 다른 분석 리포에서 볼 수 없는 교육적 접근이다. 특히:
- 13장(숨겨진 비밀): autoDream, Speculation, YOLO, Tengu 등 문서화되지 않은 기능 12개 발굴
- 17장(프롬프트 아키텍처): **54개 이상 프롬프트 파일 전수 분석** -- 다른 리포에서 이 수준의 프롬프트 분석은 없음
- 19장(Anthropic 비법): 캐시 7중 최적화, DCE, Undercover, 투기적 실행 등 **26가지 고유 기법** 정리

### 8.4 정량적 모듈 분석과 품질 지표

Stats_Report에서 단순 파일 수 집계를 넘어:
- **모듈 응집도 5점 척도 평가** (Ink Framework 5/5, Utils 3/5 등)
- **계층 분리도 분석** ("계층 위반: 최소 -- 대체로 단방향 의존성 유지")
- **설계 패턴 분포** (Factory, Async Generator, React Hooks, Singleton Store, Observer, Strategy, Memoization, Race Condition, Forked Agent)
- **위키링크 기반 연결 밀도** 정량 측정

### 8.5 실제 소스코드 포함

분석 문서만이 아니라 `./src/` 디렉터리에 **실제 TypeScript/TSX 소스코드 1,902개 파일**을 함께 포함하여, 분석 내용을 원본 코드와 직접 대조할 수 있다. 이는 분석의 검증 가능성을 높인다.

### 8.6 2차 분석으로 발견된 소규모 모듈

다른 리포들이 주요 모듈만 다루는 반면, leaf-kit은 **2차 분석**을 통해 소규모 모듈까지 빠짐없이 다루었다: bootstrap/state(1,758줄 싱글톤), coordinator(369줄 멀티워커), buddy(1,298줄 컴패니언 캐릭터), voice(54줄 피처 게이트), moreright(25줄 스텁) 등. 특히 `useMoreRight`가 "외부 빌드용 REPL 스텁"이라는 점이나, `voiceModeEnabled`가 "음성 모드 피처 게이트"(54줄)라는 점 등은 다른 분석에서 쉽게 놓치는 디테일이다.

### 8.7 Prompt & Agent Architecture 전문 섹션

README에서 별도의 대형 섹션으로 **프롬프트 및 에이전트 아키텍처**를 다룬다:
- 시스템 프롬프트 조립 순서 (10단계)
- 6종 내장 에이전트 타입 (Main, Sub-Agent, Compact, Memory Extract, Session Memory, Coordinator Worker)
- CLAUDE.md 로딩 체인 (프로젝트 루트 -> 하위 디렉터리 -> 사용자 홈)
- End-to-End 프롬프트 흐름

이는 Claude Code를 단순 코드가 아닌 **프롬프트 시스템으로 이해**하는 관점을 제공하며, AI 에이전트 개발자에게 실질적인 참고 자료가 된다.

---

## 요약

leaf-kit/claude-analysis는 Claude Code CLI 소스코드의 **가장 체계적이고 포괄적인 분석 리포지토리**이다. Obsidian 호환 지식 관리 시스템, 19장 계층형 튜토리얼, 정량적 모듈 분석, 54개 프롬프트 전수 분석, 26가지 Anthropic 고유 기법 발굴 등은 다른 분석 리포에서 찾기 어려운 고유한 가치를 제공한다. 소스코드를 함께 포함하여 분석의 검증 가능성을 확보한 점도 차별화 요소이다.
