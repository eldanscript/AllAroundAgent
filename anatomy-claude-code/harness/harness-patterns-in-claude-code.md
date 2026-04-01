# Claude Code 소스코드에서 발견된 하네스 엔지니어링 패턴 분석

> 분석 대상: `anatomy-claude-code/refs/hangsman-claude-code-source/src/`  
> 분석 일시: 2026-03-31

---

## 요약

Claude Code 소스코드는 하네스 엔지니어링의 6가지 아키텍처 패턴을 모두 네이티브로 구현하고 있다. 하네스가 `.claude/agents/`와 `.claude/skills/`에 파일을 자동 생성하는 것은 Claude Code가 이미 제공하는 에이전트 정의 시스템과 스킬 로딩 인프라 위에서 동작하는 것이다. 즉 **하네스는 Claude Code의 내장 멀티 에이전트 인프라를 활용하는 메타 스킬**이다.

---

## 1. 에이전트 오케스트레이션 패턴

### 1.1 핵심 도구 5종

Claude Code는 멀티 에이전트 조율을 위한 전용 도구를 구현한다.

| 도구 | 파일 위치 | 역할 |
|------|----------|------|
| **AgentTool** | `src/tools/AgentTool/AgentTool.tsx` | 서브 에이전트/워커 생성 (동기/비동기/백그라운드) |
| **TeamCreateTool** | `src/tools/TeamCreateTool/TeamCreateTool.ts` | 팀 생성, TeamFile에 멤버 등록 |
| **TeamDeleteTool** | `src/tools/TeamDeleteTool/TeamDeleteTool.ts` | 팀 해체, 디렉터리/워크트리 정리 |
| **SendMessageTool** | `src/tools/SendMessageTool/SendMessageTool.ts` | 에이전트 간 메시지 전달 (1:1, 브로드캐스트, 구조화된 메시지) |
| **TaskCreateTool** | `src/tools/TaskCreateTool/TaskCreateTool.ts` | 작업 항목 생성 및 추적 |
| **TaskStopTool** | `src/tools/TaskStopTool/TaskStopTool.ts` | 실행 중인 에이전트/태스크 중지 |

### 1.2 두 가지 실행 모드

하네스 문서에서 정의한 두 모드가 정확히 Claude Code에 구현되어 있다.

**에이전트 팀 모드 (Swarm):**
- `TeamCreateTool`로 팀 생성 → `SendMessageTool`로 팀원 간 메시지 교환
- `isAgentSwarmsEnabled()` 게이트로 활성화
- `CLAUDE_CODE_EXPERIMENTAL_AGENT_TEAMS=1` 환경변수 필요
- 팀 파일은 `~/.claude/teams/{team_name}/` 하위에 JSON으로 저장

```typescript
// src/tools/TeamCreateTool/TeamCreateTool.ts:157
const teamFile: TeamFile = {
  name: finalTeamName,
  description: _description,
  createdAt: Date.now(),
  leadAgentId,
  leadSessionId: getSessionId(),
  members: [{ agentId: leadAgentId, name: TEAM_LEAD_NAME, ... }],
}
```

**서브 에이전트 모드 (Coordinator):**
- `AgentTool`로 워커를 직접 스폰
- `CLAUDE_CODE_COORDINATOR_MODE=1` 환경변수로 활성화
- Coordinator 시스템 프롬프트가 자동 주입됨

```typescript
// src/coordinator/coordinatorMode.ts:116
return `You are Claude Code, an AI assistant that orchestrates software engineering tasks across multiple workers.`
```

### 1.3 Fork 서브에이전트

`src/tools/AgentTool/forkSubagent.ts`에서 세 번째 모드를 구현한다. `subagent_type`을 생략하면 부모의 전체 컨텍스트를 상속하는 "포크" 에이전트가 생성된다. 이는 coordinator 모드와 상호 배타적이다.

```typescript
// src/tools/AgentTool/forkSubagent.ts:32-36
export function isForkSubagentEnabled(): boolean {
  if (feature('FORK_SUBAGENT')) {
    if (isCoordinatorMode()) return false
    if (getIsNonInteractiveSession()) return false
    return true
  }
  return false
}
```

---

## 2. 에이전트 정의 시스템

### 2.1 `.claude/agents/` 디렉터리 구조

`src/tools/AgentTool/loadAgentsDir.ts`가 에이전트 정의를 로드한다. 하네스가 생성하는 `.claude/agents/*.md` 파일은 이 시스템이 파싱한다.

**에이전트 정의 소스 우선순위** (나중 것이 앞의 것을 덮어씀):
1. `built-in` — 코드에 내장된 에이전트
2. `plugin` — 플러그인에서 로드
3. `userSettings` — `~/.claude/agents/`
4. `projectSettings` — 프로젝트의 `.claude/agents/`
5. `flagSettings` — 원격 설정/기능 플래그
6. `policySettings` — 관리 정책

```typescript
// src/tools/AgentTool/loadAgentsDir.ts:296-393
export const getAgentDefinitionsWithOverrides = memoize(
  async (cwd: string): Promise<AgentDefinitionsResult> => {
    const markdownFiles = await loadMarkdownFilesForSubdir('agents', cwd)
    const customAgents = markdownFiles.map(({ filePath, frontmatter, content, source }) => {
      return parseAgentFromMarkdown(filePath, baseDir, frontmatter, content, source)
    })
    const pluginAgents = await loadPluginAgents()
    const builtInAgents = getBuiltInAgents()
    // 모든 소스를 병합하고 우선순위에 따라 덮어씀
  }
)
```

### 2.2 에이전트 정의 스키마

마크다운 프론트매터로 에이전트의 모든 속성을 정의한다. 하네스가 생성하는 `.md` 파일도 이 형식을 따른다.

```typescript
// src/tools/AgentTool/loadAgentsDir.ts:73-99
const AgentJsonSchema = z.object({
  description: z.string(),
  tools: z.array(z.string()).optional(),
  disallowedTools: z.array(z.string()).optional(),
  prompt: z.string(),
  model: z.string().optional(),
  effort: z.union([z.enum(EFFORT_LEVELS), z.number().int()]).optional(),
  permissionMode: z.enum(PERMISSION_MODES).optional(),
  mcpServers: z.array(AgentMcpServerSpecSchema()).optional(),
  hooks: HooksSchema().optional(),
  maxTurns: z.number().int().positive().optional(),
  skills: z.array(z.string()).optional(),
  initialPrompt: z.string().optional(),
  memory: z.enum(['user', 'project', 'local']).optional(),
  background: z.boolean().optional(),
  isolation: z.enum(['worktree']).optional(),
})
```

### 2.3 내장 에이전트 목록

`src/tools/AgentTool/builtInAgents.ts`에서 기본 에이전트를 등록한다:

- **general-purpose** — 범용 에이전트 (`built-in/generalPurposeAgent.ts`)
- **explore** — 코드베이스 탐색 전용 (`built-in/exploreAgent.ts`)
- **plan** — 계획 수립 전용 (`built-in/planAgent.ts`)
- **verification** — 검증 에이전트 (`built-in/verificationAgent.ts`)
- **claude-code-guide** — 사용법 안내 (`built-in/claudeCodeGuideAgent.ts`)
- **statusline-setup** — 상태바 설정 (`built-in/statuslineSetup.ts`)
- **worker** — Coordinator 모드 전용 워커 (coordinator/workerAgent.ts)

### 2.4 에이전트 UI 관리

`src/components/agents/` 디렉터리에 에이전트를 생성/편집/삭제하는 전체 UI가 구현되어 있다:
- `CreateAgentWizard.tsx` — 에이전트 생성 마법사
- `AgentEditor.tsx` — 에이전트 편집기
- `AgentsList.tsx` — 에이전트 목록
- `AgentDetail.tsx` — 상세 보기
- `validateAgent.ts` — 유효성 검증
- `agentFileUtils.ts` — 파일 CRUD

경로 상수:
```typescript
// src/components/agents/types.ts:5-7
export const AGENT_PATHS = {
  FOLDER_NAME: '.claude',
  AGENTS_DIR: 'agents',
}
```

---

## 3. 스킬 시스템

### 3.1 스킬 로딩 계층

`src/skills/` 디렉터리가 스킬 시스템의 핵심이다.

| 모듈 | 역할 |
|------|------|
| `loadSkillsDir.ts` | `.claude/skills/` 디렉터리에서 마크다운 스킬 파일 로드 |
| `bundledSkills.ts` | 내장 스킬 등록 (`registerBundledSkill()`) |
| `mcpSkillBuilders.ts` | MCP 서버의 프롬프트를 스킬로 변환 |

**스킬 소스 종류** (`LoadedFrom` 타입):
```typescript
// src/skills/loadSkillsDir.ts:67-74
export type LoadedFrom = 'commands_DEPRECATED' | 'skills' | 'plugin' | 'managed' | 'bundled' | 'mcp'
```

### 3.2 내장 스킬 목록

`src/skills/bundled/` 디렉터리의 내장 스킬들:

| 스킬 | 파일 | 설명 |
|------|------|------|
| **batch** | `batch.ts` | 대규모 병렬 변경 오케스트레이션 (5-30 워크트리 에이전트) |
| **simplify** | `simplify.ts` | 코드 리뷰 및 단순화 |
| **verify** | `verify.ts` | 변경사항 검증 |
| **debug** | `debug.ts` | 디버깅 지원 |
| **loop** | `loop.ts` | 반복 실행 (폴링) |
| **claude-api** | `claudeApi.ts` | Claude API 사용 가이드 |
| **update-config** | `updateConfig.ts` | 설정 변경 |
| **skillify** | `skillify.ts` | 스킬 자동 생성 |
| **remember** | `remember.ts` | 기억 저장 |
| **stuck** | `stuck.ts` | 막힌 상황 해결 |

### 3.3 SkillTool — 스킬 실행 엔진

`src/tools/SkillTool/SkillTool.ts`가 스킬을 실행한다. 두 가지 실행 모드:

- **Inline**: 스킬 프롬프트를 현재 컨텍스트에 주입
- **Fork**: 별도 서브 에이전트에서 스킬 실행 (격리된 토큰 예산)

```typescript
// src/tools/SkillTool/SkillTool.ts:122
async function executeForkedSkill(command, commandName, args, context, ...) {
  const agentId = createAgentId()
  for await (const message of runAgent({
    agentDefinition, promptMessages, toolUseContext: { ...context, getAppState: modifiedGetAppState },
    canUseTool, isAsync: false, querySource: 'agent:custom', ...
  })) {
    agentMessages.push(message)
  }
}
```

---

## 4. 6가지 아키텍처 패턴의 구현 흔적

### 4.1 파이프라인 (순차 실행)

Coordinator 시스템 프롬프트에서 명시적으로 4단계 파이프라인을 정의한다:

```
// src/coordinator/coordinatorMode.ts:198-209
| Phase | Who | Purpose |
| Research | Workers (parallel) | Investigate codebase |
| Synthesis | **You** (coordinator) | Read findings, craft specs |
| Implementation | Workers | Make targeted changes, commit |
| Verification | Workers | Test changes work |
```

`/batch` 스킬은 3단계 파이프라인을 구현:
1. **Phase 1: Research and Plan** (Plan Mode) — 조사 후 독립 단위 분해
2. **Phase 2: Spawn Workers** — 병렬 워크트리 에이전트 스폰
3. **Phase 3: Track Progress** — 상태 추적 및 결과 집계

### 4.2 팬아웃/팬인 (병렬 에이전트)

Coordinator 프롬프트에서 병렬 실행을 핵심 능력으로 강조:

```
// src/coordinator/coordinatorMode.ts:212-213
**Parallelism is your superpower. Workers are async. Launch independent workers
concurrently whenever possible — don't serialize work that can run simultaneously
and look for opportunities to fan out.**
```

`/batch` 스킬이 팬아웃/팬인의 완전한 구현:
```typescript
// src/skills/bundled/batch.ts:61
// Phase 2: 모든 에이전트를 단일 메시지 블록에서 병렬 실행
// All agents must use isolation: "worktree" and run_in_background: true.
// Launch them all in a single message block so they run in parallel.
```

팬인 (수집):
```
// Phase 3: 완료 알림 파싱하여 상태표 업데이트
// As background-agent completion notifications arrive, parse the PR: <url> line
```

### 4.3 전문가 풀 (도구 선택)

AgentTool의 `subagent_type` 매개변수가 전문가 풀 패턴을 구현한다. 에이전트 유형에 따라 다른 도구/프롬프트/모델이 할당된다:

```typescript
// src/tools/AgentTool/prompt.ts:202-206
// Available agent types and the tools they have access to:
// ${effectiveAgents.map(agent => formatAgentLine(agent)).join('\n')}
```

각 에이전트는 `tools` 화이트리스트와 `disallowedTools` 블랙리스트로 도구 접근을 제어:
```typescript
// src/tools/AgentTool/loadAgentsDir.ts:106-112
export type BaseAgentDefinition = {
  agentType: string
  tools?: string[]           // 허용된 도구만
  disallowedTools?: string[] // 차단된 도구
  skills?: string[]          // 사전 로드할 스킬
  mcpServers?: AgentMcpServerSpec[]  // 에이전트별 MCP 서버
}
```

### 4.4 생성-검증 (코드 리뷰)

Coordinator 프롬프트에서 생성과 검증을 명시적으로 분리:

```
// src/coordinator/coordinatorMode.ts:220-226
### What Real Verification Looks Like
- Run tests **with the feature enabled**
- Run typechecks and **investigate errors**
- Be skeptical — if something looks off, dig in
- **Test independently** — prove the change works, don't rubber-stamp
```

검증 전용 내장 에이전트:
```typescript
// src/tools/AgentTool/builtInAgents.ts:65-68
if (feature('VERIFICATION_AGENT') &&
    getFeatureValue_CACHED_MAY_BE_STALE('tengu_hive_evidence', false)) {
  agents.push(VERIFICATION_AGENT)
}
```

Coordinator는 검증 시 새 에이전트를 스폰하도록 권장:
```
// src/coordinator/coordinatorMode.ts:289
| Verifying code a different worker just wrote | **Spawn fresh** | Verifier should see the code with fresh eyes |
```

### 4.5 감독자 (Coordinator)

`src/coordinator/coordinatorMode.ts` 전체가 감독자 패턴의 구현이다. Coordinator는:

- 직접 도구를 사용하지 않고 워커에게 위임
- 결과를 `<task-notification>` XML로 수신
- 실패 시 `SendMessage`로 동일 워커를 계속하거나 새 워커를 스폰
- 워커의 접근 가능 도구를 동적으로 컨텍스트에 포함

```typescript
// src/coordinator/coordinatorMode.ts:88-108
export function getCoordinatorUserContext(mcpClients, scratchpadDir?) {
  let content = `Workers spawned via the ${AGENT_TOOL_NAME} tool have access to these tools: ${workerTools}`
  if (mcpClients.length > 0) {
    content += `\n\nWorkers also have access to MCP tools from connected MCP servers: ${serverNames}`
  }
  if (scratchpadDir && isScratchpadGateEnabled()) {
    content += `\n\nScratchpad directory: ${scratchpadDir}\nWorkers can read and write here without permission prompts.`
  }
}
```

### 4.6 계층적 위임 (서브에이전트)

AgentTool이 재귀적 서브 에이전트 스폰을 지원한다. 다만 무한 재귀 방지를 위해 비동기 에이전트에서는 AgentTool 자체를 차단:

```typescript
// src/constants/tools.ts:90-93
// BLOCKED FOR ASYNC AGENTS:
// - AgentTool: Blocked to prevent recursion
// - TaskOutputTool: Blocked to prevent recursion
```

Fork 에이전트도 재귀 포크를 감지하여 차단:
```typescript
// src/tools/AgentTool/forkSubagent.ts:77-78
export function isInForkChild(messages: MessageType[]): boolean {
  return messages.some(m => { /* fork boilerplate tag 검사 */ })
}
```

인프로세스 팀메이트는 추가 도구를 가진다:
```typescript
// src/constants/tools.ts:77-88
export const IN_PROCESS_TEAMMATE_ALLOWED_TOOLS = new Set([
  TASK_CREATE_TOOL_NAME, TASK_GET_TOOL_NAME, TASK_LIST_TOOL_NAME,
  TASK_UPDATE_TOOL_NAME, SEND_MESSAGE_TOOL_NAME,
])
```

---

## 5. 에러 핸들링 프로토콜

### 5.1 워커 실패 복구

Coordinator 프롬프트에서 실패 처리 전략을 명시:

```
// src/coordinator/coordinatorMode.ts:229-232
### Handling Worker Failures
When a worker reports failure (tests failed, build errors, file not found):
- Continue the same worker with SendMessage — it has the full error context
- If a correction attempt fails, try a different approach or report to the user
```

### 5.2 워커 중지 및 방향 전환

`TaskStopTool`로 잘못된 방향의 워커를 중지하고 `SendMessage`로 재지시:

```
// src/coordinator/coordinatorMode.ts:236-249
${TASK_STOP_TOOL_NAME}({ task_id: "agent-x7q" })
// User clarifies: "Actually, keep sessions — just fix the null pointer"
${SEND_MESSAGE_TOOL_NAME}({ to: "agent-x7q", message: "Stop the JWT refactor. Instead, fix..." })
```

### 5.3 Shutdown 프로토콜

`SendMessageTool`에 구조화된 셧다운 요청/응답 프로토콜이 구현되어 있다:

```typescript
// src/tools/SendMessageTool/SendMessageTool.ts:47-65
const StructuredMessage = z.discriminatedUnion('type', [
  z.object({ type: z.literal('shutdown_request'), reason: z.string().optional() }),
  z.object({ type: z.literal('shutdown_response'), request_id: z.string(), approve: semanticBoolean(), reason: z.string().optional() }),
  z.object({ type: z.literal('plan_approval_response'), request_id: z.string(), approve: semanticBoolean(), feedback: z.string().optional() }),
])
```

- **shutdown_request**: 리더가 팀원에게 종료 요청
- **shutdown_response**: 팀원이 승인/거부 (거부 시 사유 필수)
- **plan_approval_response**: 리더가 팀원의 계획을 승인/거부

### 5.4 에이전트 자동 재개

중지된 에이전트에 SendMessage를 보내면 자동으로 재개된다:

```typescript
// src/tools/SendMessageTool/SendMessageTool.ts:822-835
if (task.status !== 'running') {
  // task exists but stopped — auto-resume
  const result = await resumeAgentBackground({
    agentId, prompt: input.message, toolUseContext: context, ...
  })
  return { data: { success: true, message: `Agent "${input.to}" was stopped; resumed it in the background...` } }
}
```

---

## 6. Worktree 격리

### 6.1 EnterWorktreeTool / ExitWorktreeTool

에이전트별 독립 작업 공간을 git worktree로 생성한다.

- `src/tools/EnterWorktreeTool/EnterWorktreeTool.ts` — 워크트리 생성 및 세션 전환
- `src/tools/ExitWorktreeTool/ExitWorktreeTool.ts` — 워크트리 종료 (keep/remove)
- `src/utils/worktree.ts` — 핵심 유틸리티 (생성, 정리, 심링크, 변경 감지)

```typescript
// src/tools/EnterWorktreeTool/EnterWorktreeTool.ts:92
const worktreeSession = await createWorktreeForSession(getSessionId(), slug)
process.chdir(worktreeSession.worktreePath)
setCwd(worktreeSession.worktreePath)
```

### 6.2 AgentTool의 isolation 옵션

AgentTool에서 `isolation: "worktree"`를 지정하면 에이전트가 격리된 워크트리에서 실행된다:

```typescript
// src/tools/AgentTool/AgentTool.tsx:99
isolation: z.enum(['worktree']).optional().describe(
  'Isolation mode. "worktree" creates a temporary git worktree so the agent works on an isolated copy of the repo.'
)
```

Worktree는 에이전트 완료 시 자동 정리:
```typescript
// src/tools/AgentTool/AgentTool.tsx:42
import { createAgentWorktree, hasWorktreeChanges, removeAgentWorktree } from '../../utils/worktree.js'
```

### 6.3 `/batch` 스킬과 Worktree

`/batch` 스킬은 모든 워커에 `isolation: "worktree"`를 강제한다:

```
// src/skills/bundled/batch.ts:61
All agents must use isolation: "worktree" and run_in_background: true.
```

각 워커가 독립 브랜치에서 작업하고 개별 PR을 생성하는 구조다.

### 6.4 Worktree 안전 장치

변경사항이 있는 워크트리 삭제 시 명시적 확인을 요구:

```typescript
// src/tools/ExitWorktreeTool/ExitWorktreeTool.ts:191-219
if (input.action === 'remove' && !input.discard_changes) {
  const summary = await countWorktreeChanges(session.worktreePath, session.originalHeadCommit)
  if (changedFiles > 0 || commits > 0) {
    return { result: false, message: `Worktree has ${parts.join(' and ')}. Removing will discard...` }
  }
}
```

---

## 7. 공유 스크래치패드

### 7.1 Coordinator 모드의 스크래치패드

Coordinator가 워커들 간 데이터 공유를 위한 스크래치패드 디렉터리를 제공한다:

```typescript
// src/coordinator/coordinatorMode.ts:104-106
if (scratchpadDir && isScratchpadGateEnabled()) {
  content += `\n\nScratchpad directory: ${scratchpadDir}\n
  Workers can read and write here without permission prompts.
  Use this for durable cross-worker knowledge — structure files however fits the work.`
}
```

### 7.2 세션별 스크래치패드

권한 없이 파일을 읽고 쓸 수 있는 격리된 임시 디렉터리:

```typescript
// src/utils/permissions/filesystem.ts:381-385
// Path format: /tmp/claude-{uid}/{sanitized-cwd}/{sessionId}/scratchpad/
export function getScratchpadDir(): string {
  return join(getProjectTempDir(), getSessionId(), 'scratchpad')
}
```

시스템 프롬프트에 스크래치패드 사용법이 주입된다:

```typescript
// src/constants/prompts.ts:806-818
`IMPORTANT: Always use this scratchpad directory for temporary files instead of /tmp:
${scratchpadDir}
The scratchpad directory is session-specific, isolated from the user's project,
and can be used freely without permission prompts.`
```

### 7.3 메일박스 시스템

`src/utils/teammateMailbox.ts`가 파일 기반 에이전트 간 메시징을 구현한다:

```typescript
// src/utils/teammateMailbox.ts:56
// Structure: ~/.claude/teams/{team_name}/inboxes/{agent_name}.json
export function getInboxPath(agentName: string, teamName?: string): string {
  const team = teamName || getTeamName() || 'default'
  return join(getTeamsDir(), safeTeam, 'inboxes', `${safeAgentName}.json`)
}
```

잠금 메커니즘으로 동시 접근 안전성을 보장:
```typescript
// src/utils/teammateMailbox.ts:35-41
const LOCK_OPTIONS = {
  retries: { retries: 10, minTimeout: 5, maxTimeout: 100 },
}
```

---

## 8. 하네스와 Claude Code의 관계 정리

### 8.1 하네스가 활용하는 Claude Code 인프라

| 하네스 기능 | Claude Code 인프라 |
|------------|-------------------|
| `.claude/agents/` 에이전트 정의 생성 | `loadAgentsDir.ts`의 마크다운 파싱 시스템 |
| `.claude/skills/` 스킬 생성 | `loadSkillsDir.ts`의 스킬 로딩 시스템 |
| 에이전트 팀 오케스트레이션 | `TeamCreateTool` + `SendMessageTool` |
| 서브 에이전트 직접 호출 | `AgentTool` (subagent_type 지정) |
| 에이전트 간 통신 | `teammateMailbox.ts` 파일 기반 메시징 |
| 작업 격리 | `EnterWorktreeTool` / 워크트리 시스템 |
| 공유 데이터 | 스크래치패드 디렉터리 |

### 8.2 하네스의 6가지 패턴 ↔ Claude Code 구현 매핑

| 하네스 패턴 | Claude Code 구현 | 핵심 코드 위치 |
|------------|-----------------|---------------|
| **파이프라인** | Coordinator의 4단계 워크플로우, `/batch`의 3단계 | `coordinatorMode.ts:198-209`, `batch.ts` |
| **팬아웃/팬인** | 병렬 AgentTool 호출 + `<task-notification>` 수집 | `coordinatorMode.ts:212-213`, `batch.ts:61` |
| **전문가 풀** | `subagent_type` + 에이전트별 도구 화이트리스트 | `AgentTool.tsx:85`, `loadAgentsDir.ts:106` |
| **생성-검증** | 구현 워커 → 별도 검증 워커 (fresh spawn) | `coordinatorMode.ts:220-226,289` |
| **감독자** | Coordinator 모드 전체 | `coordinatorMode.ts` |
| **계층적 위임** | 에이전트 내 서브 에이전트 스폰 (재귀 제한 있음) | `constants/tools.ts:90-93` |

### 8.3 하네스가 "메타 스킬"인 이유

하네스 자체는 Claude Code의 스킬 시스템(`registerBundledSkill` 또는 `.claude/skills/`) 위에서 동작하는 **프롬프트 기반 스킬**이다. "하네스 구성해줘"라는 명령은 결국:

1. 도메인을 분석하여 적합한 아키텍처 패턴을 선택
2. `.claude/agents/`에 에이전트 정의 마크다운 파일을 **Write** 도구로 생성
3. `.claude/skills/`에 오케스트레이션 스킬 파일을 생성
4. 이 파일들이 Claude Code의 `loadAgentsDir.ts`와 `loadSkillsDir.ts`에 의해 자동으로 로드됨

즉 하네스는 새로운 런타임을 만드는 것이 아니라, **Claude Code가 이미 갖추고 있는 멀티 에이전트 인프라의 설정 파일을 자동 생성하는 메타 스킬**이다.

---

## 부록: 주요 파일 경로 요약

```
src/coordinator/
  coordinatorMode.ts          — Coordinator 모드 시스템 프롬프트 및 설정

src/tools/AgentTool/
  AgentTool.tsx               — 에이전트 생성/실행 메인 로직
  loadAgentsDir.ts            — .claude/agents/ 파싱 및 에이전트 정의 로딩
  builtInAgents.ts            — 내장 에이전트 등록
  forkSubagent.ts             — Fork 에이전트 (컨텍스트 상속)
  runAgent.ts                 — 에이전트 실행 엔진
  prompt.ts                   — AgentTool 프롬프트 생성
  built-in/                   — 내장 에이전트 정의들

src/tools/TeamCreateTool/     — 팀 생성
src/tools/TeamDeleteTool/     — 팀 해체
src/tools/SendMessageTool/    — 에이전트 간 메시징
src/tools/TaskCreateTool/     — 작업 생성
src/tools/TaskStopTool/       — 작업 중지
src/tools/SkillTool/          — 스킬 실행 엔진
src/tools/EnterWorktreeTool/  — 워크트리 진입
src/tools/ExitWorktreeTool/   — 워크트리 퇴장

src/skills/
  loadSkillsDir.ts            — .claude/skills/ 스킬 로딩
  bundledSkills.ts            — 내장 스킬 레지스트리
  mcpSkillBuilders.ts         — MCP 프롬프트→스킬 변환
  bundled/batch.ts            — /batch 스킬 (팬아웃/팬인 구현)
  bundled/simplify.ts         — 코드 리뷰 스킬
  bundled/verify.ts           — 검증 스킬

src/utils/
  worktree.ts                 — Git worktree 유틸리티
  teammateMailbox.ts          — 파일 기반 에이전트 메시징
  swarm/teamHelpers.ts        — 팀 파일 관리 (TeamFile JSON)
  swarm/inProcessRunner.ts    — 인프로세스 팀메이트 실행
  permissions/filesystem.ts   — 스크래치패드 경로/권한 관리

src/components/agents/        — 에이전트 관리 UI
  types.ts                    — AGENT_PATHS 상수 정의
```
