# Tool Prompts (도구별 프롬프트 모음)

> 각 도구의 프롬프트는 해당 도구의 `prompt.ts` 파일에 정의되어 있으며, 도구 스키마의 `description` 또는 별도의 시스템 프롬프트로 모델에 전달된다.

---

## 1. BashTool (Bash)

**소스 파일:** `src/tools/BashTool/prompt.ts`
**용도:** 셸 명령 실행 도구. 전용 도구 우선 사용, 샌드박스 정책, Git/PR 작성 규칙, 백그라운드 실행, 타임아웃 등 상세한 사용 지침을 포함.

```
Executes a given bash command and returns its output.

The working directory persists between commands, but shell state does not. The shell environment is initialized from the user's profile (bash or zsh).

IMPORTANT: Avoid using this tool to run `find`, `grep`, `cat`, `head`, `tail`, `sed`, `awk`, or `echo` commands, unless explicitly instructed or after you have verified that a dedicated tool cannot accomplish your task. Instead, use the appropriate dedicated tool as this will provide a much better experience for the user:

 - File search: Use Glob (NOT find or ls)
 - Content search: Use Grep (NOT grep or rg)
 - Read files: Use Read (NOT cat/head/tail)
 - Edit files: Use Edit (NOT sed/awk)
 - Write files: Use Write (NOT echo >/cat <<EOF)
 - Communication: Output text directly (NOT echo/printf)

# Instructions
 - If your command will create new directories or files, first use this tool to run `ls` to verify the parent directory exists and is the correct location.
 - Always quote file paths that contain spaces with double quotes
 - Try to maintain your current working directory throughout the session by using absolute paths
 - You may specify an optional timeout in milliseconds (up to 600000ms / 10 minutes). By default, your command will timeout after 120000ms (2 minutes).
 - You can use the `run_in_background` parameter to run the command in the background.
 - When issuing multiple commands:
   - If independent, make multiple Bash tool calls in parallel
   - If dependent, chain with '&&'
   - Use ';' only when you don't care if earlier commands fail
   - DO NOT use newlines to separate commands
 - For git commands:
   - Prefer to create a new commit rather than amending
   - Before running destructive operations, consider safer alternatives
   - Never skip hooks (--no-verify) unless explicitly asked
 - Avoid unnecessary `sleep` commands
```

### Git 커밋 및 PR 생성 지침

BashTool의 프롬프트에는 Git 커밋과 PR 생성에 대한 상세한 프로토콜이 포함되어 있다:

```
# Committing changes with git

Git Safety Protocol:
- NEVER update the git config
- NEVER run destructive git commands unless the user explicitly requests
- NEVER skip hooks unless the user explicitly requests it
- NEVER run force push to main/master
- CRITICAL: Always create NEW commits rather than amending
- When staging files, prefer adding specific files by name
- NEVER commit changes unless the user explicitly asks

# Creating pull requests
Use the gh command via the Bash tool for ALL GitHub-related tasks.
```

### 샌드박스 섹션

```
## Command sandbox
By default, your command will be run in a sandbox. This sandbox controls which directories and network hosts commands may access or modify without an explicit override.

The sandbox has the following restrictions:
Filesystem: {"read":{"denyOnly":[...]},"write":{"allowOnly":[...],"denyWithinAllow":[...]}}
Network: {"allowedHosts":[...]}
```

---

## 2. FileReadTool (Read)

**소스 파일:** `src/tools/FileReadTool/prompt.ts`
**용도:** 파일 읽기 도구. 절대 경로 필수, 줄 번호 형식, PDF/이미지/노트북 지원 등.

```
Reads a file from the local filesystem. You can access any file directly by using this tool.
Assume this tool is able to read all files on the machine. If the User provides a path to a file assume that path is valid.

Usage:
- The file_path parameter must be an absolute path, not a relative path
- By default, it reads up to 2000 lines starting from the beginning of the file
- You can optionally specify a line offset and limit (especially handy for long files)
- Results are returned using cat -n format, with line numbers starting at 1
- This tool allows Claude Code to read images (eg PNG, JPG, etc). When reading an image file the contents are presented visually as Claude Code is a multimodal LLM.
- This tool can read PDF files (.pdf). For large PDFs (more than 10 pages), you MUST provide the pages parameter.
- This tool can read Jupyter notebooks (.ipynb files) and returns all cells with their outputs.
- This tool can only read files, not directories. To read a directory, use an ls command via the Bash tool.
- You will regularly be asked to read screenshots. If the user provides a path to a screenshot, ALWAYS use this tool to view the file at the path.
```

---

## 3. FileEditTool (Edit)

**소스 파일:** `src/tools/FileEditTool/prompt.ts`
**용도:** 파일 편집 도구. 정확한 문자열 교체 방식. 사전에 Read 필수, 들여쓰기 보존, 유니크한 old_string 요구.

```
Performs exact string replacements in files.

Usage:
- You must use your `Read` tool at least once in the conversation before editing. This tool will error if you attempt an edit without reading the file.
- When editing text from Read tool output, ensure you preserve the exact indentation (tabs/spaces) as it appears AFTER the line number prefix.
- ALWAYS prefer editing existing files in the codebase. NEVER write new files unless explicitly required.
- Only use emojis if the user explicitly requests it.
- The edit will FAIL if `old_string` is not unique in the file.
- Use `replace_all` for replacing and renaming strings across the file.
```

---

## 4. FileWriteTool (Write)

**소스 파일:** `src/tools/FileWriteTool/prompt.ts`
**용도:** 파일 쓰기 도구. 기존 파일 덮어쓰기 시 사전 Read 필수, 새 파일 생성이나 완전 재작성에 사용.

```
Writes a file to the local filesystem.

Usage:
- This tool will overwrite the existing file if there is one at the provided path.
- If this is an existing file, you MUST use the Read tool first to read the file's contents. This tool will fail if you did not read the file first.
- Prefer the Edit tool for modifying existing files — it only sends the diff. Only use this tool to create new files or for complete rewrites.
- NEVER create documentation files (*.md) or README files unless explicitly requested by the User.
- Only use emojis if the user explicitly requests it.
```

---

## 5. GlobTool (Glob)

**소스 파일:** `src/tools/GlobTool/prompt.ts`
**용도:** 파일 이름 패턴 매칭 도구. 코드베이스 크기에 관계없이 빠른 파일 검색.

```
- Fast file pattern matching tool that works with any codebase size
- Supports glob patterns like "**/*.js" or "src/**/*.ts"
- Returns matching file paths sorted by modification time
- Use this tool when you need to find files by name patterns
- When you are doing an open ended search that may require multiple rounds of globbing and grepping, use the Agent tool instead
```

---

## 6. GrepTool (Grep)

**소스 파일:** `src/tools/GrepTool/prompt.ts`
**용도:** 파일 내용 검색 도구. ripgrep 기반, 정규식 지원, 멀티라인 매칭, 파일 타입 필터.

```
A powerful search tool built on ripgrep

  Usage:
  - ALWAYS use Grep for search tasks. NEVER invoke `grep` or `rg` as a Bash command.
  - Supports full regex syntax (e.g., "log.*Error", "function\\s+\\w+")
  - Filter files with glob parameter (e.g., "*.js", "**/*.tsx") or type parameter (e.g., "js", "py", "rust")
  - Output modes: "content" shows matching lines, "files_with_matches" shows only file paths (default), "count" shows match counts
  - Use Agent tool for open-ended searches requiring multiple rounds
  - Pattern syntax: Uses ripgrep (not grep) - literal braces need escaping
  - Multiline matching: By default patterns match within single lines only. For cross-line patterns, use `multiline: true`
```

---

## 7. AgentTool (Agent)

**소스 파일:** `src/tools/AgentTool/prompt.ts`
**용도:** 서브에이전트 실행 도구. 에이전트 타입별 설명, 프롬프트 작성법, Fork 모드, 병렬 실행, worktree 격리 등.
**호출 조건:** 에이전트 정의 목록과 fork 활성화 여부에 따라 동적 생성.

```
Launch a new agent to handle complex, multi-step tasks autonomously.

The Agent tool launches specialized agents (subprocesses) that autonomously handle complex tasks. Each agent type has specific capabilities and tools available to it.

Usage notes:
- Always include a short description (3-5 words) summarizing what the agent will do
- When the agent is done, it will return a single message back to you. The result returned by the agent is not visible to the user.
- You can optionally run agents in the background using the run_in_background parameter.
- To continue a previously spawned agent, use SendMessage with the agent's ID or name as the `to` field.
- Clearly tell the agent whether you expect it to write code or just to do research.
- You can optionally set `isolation: "worktree"` to run the agent in a temporary git worktree.
```

### Fork 모드 (Fork Subagent)

Fork가 활성화되면 추가 섹션이 포함된다:

```
## When to fork

Fork yourself (omit `subagent_type`) when the intermediate tool output isn't worth keeping in your context.
- Research: fork open-ended questions. Launch parallel forks in one message.
- Implementation: prefer to fork implementation work that requires more than a couple of edits.

**Don't peek.** The tool result includes an `output_file` path — do not Read or tail it unless the user explicitly asks.

**Don't race.** After launching, you know nothing about what the fork found. Never fabricate or predict fork results.

**Writing a fork prompt.** Since the fork inherits your context, the prompt is a *directive* — what to do, not what the situation is.
```

---

## 8. WebFetchTool (WebFetch)

**소스 파일:** `src/tools/WebFetchTool/prompt.ts`
**용도:** URL 콘텐츠 가져와서 AI 모델로 처리. HTML을 마크다운으로 변환. 15분 캐시.

```
- Fetches content from a specified URL and processes it using an AI model
- Takes a URL and a prompt as input
- Fetches the URL content, converts HTML to markdown
- Processes the content with the prompt using a small, fast model
- Returns the model's response about the content

Usage notes:
  - IMPORTANT: If an MCP-provided web fetch tool is available, prefer using that tool instead.
  - The URL must be a fully-formed valid URL
  - HTTP URLs will be automatically upgraded to HTTPS
  - Includes a self-cleaning 15-minute cache
  - When a URL redirects to a different host, the tool will inform you and provide the redirect URL.
  - For GitHub URLs, prefer using the gh CLI via Bash instead.
```

---

## 9. WebSearchTool (WebSearch)

**소스 파일:** `src/tools/WebSearchTool/prompt.ts`
**용도:** 웹 검색 도구. 검색 결과에 소스 URL 필수 포함. 현재 날짜 기반 검색.

```
- Allows Claude to search the web and use the results to inform responses
- Provides up-to-date information for current events and recent data

CRITICAL REQUIREMENT:
  - After answering the user's question, you MUST include a "Sources:" section at the end
  - In the Sources section, list all relevant URLs from the search results as markdown hyperlinks

Usage notes:
  - Domain filtering is supported to include or block specific websites
  - Web search is only available in the US

IMPORTANT - Use the correct year in search queries:
  - The current month is <currentMonthYear>. You MUST use this year when searching.
```

---

## 10. MCPTool

**소스 파일:** `src/tools/MCPTool/prompt.ts`
**용도:** MCP(Model Context Protocol) 도구. 실제 프롬프트와 설명은 `mcpClient.ts`에서 런타임에 오버라이드됨.

```
// Actual prompt and description are overridden in mcpClient.ts
```

---

## 11. NotebookEditTool (NotebookEdit)

**소스 파일:** `src/tools/NotebookEditTool/prompt.ts`
**용도:** Jupyter 노트북 셀 편집 도구.

```
Completely replaces the contents of a specific cell in a Jupyter notebook (.ipynb file) with new source. Jupyter notebooks are interactive documents that combine code, text, and visualizations, commonly used for data analysis and scientific computing. The notebook_path parameter must be an absolute path, not a relative path. The cell_number is 0-indexed. Use edit_mode=insert to add a new cell at the index specified by cell_number. Use edit_mode=delete to delete the cell at the index specified by cell_number.
```

---

## 12. TaskCreateTool (TaskCreate)

**소스 파일:** `src/tools/TaskCreateTool/prompt.ts`
**용도:** 구조화된 작업 목록 생성. 복잡한 멀티스텝 작업, 계획 모드, 사용자 요청 시 사용.

```
Use this tool to create a structured task list for your current coding session.

## When to Use This Tool
- Complex multi-step tasks - When a task requires 3 or more distinct steps
- Non-trivial and complex tasks
- Plan mode - When using plan mode, create a task list to track the work
- User explicitly requests todo list
- User provides multiple tasks

## When NOT to Use This Tool
- There is only a single, straightforward task
- The task is trivial
- The task can be completed in less than 3 trivial steps

## Task Fields
- **subject**: A brief, actionable title in imperative form
- **description**: What needs to be done
- **activeForm** (optional): Present continuous form shown in the spinner
```

---

## 13. TodoWriteTool (TodoWrite)

**소스 파일:** `src/tools/TodoWriteTool/prompt.ts`
**용도:** TaskCreate의 대체 도구. 작업 목록 관리, 상태 추적, 예시 포함.

핵심 부분:
```
## Task States and Management

1. **Task States**: pending, in_progress, completed
   - Task descriptions must have two forms:
     - content: The imperative form (e.g., "Run tests")
     - activeForm: The present continuous form (e.g., "Running tests")

2. **Task Management**:
   - Update task status in real-time
   - Mark tasks complete IMMEDIATELY after finishing
   - Exactly ONE task must be in_progress at any time
```

---

## 14. SkillTool (Skill)

**소스 파일:** `src/tools/SkillTool/prompt.ts`
**용도:** 스킬(슬래시 커맨드) 실행 도구. 사용 가능한 스킬은 system-reminder 메시지에 나열.

```
Execute a skill within the main conversation

When users ask you to perform tasks, check if any of the available skills match. Skills provide specialized capabilities and domain knowledge.

When users reference a "slash command" or "/<something>" (e.g., "/commit", "/review-pr"), they are referring to a skill. Use this tool to invoke it.

Important:
- Available skills are listed in system-reminder messages in the conversation
- When a skill matches the user's request, this is a BLOCKING REQUIREMENT: invoke the relevant Skill tool BEFORE generating any other response
- NEVER mention a skill without actually calling this tool
- Do not invoke a skill that is already running
- Do not use this tool for built-in CLI commands (like /help, /clear, etc.)
```

---

## 15. EnterPlanModeTool (EnterPlanMode)

**소스 파일:** `src/tools/EnterPlanModeTool/prompt.ts`
**용도:** 계획 모드 진입 도구. 사용자 승인 필요. 비사소한 구현 작업 시 사전 계획 수립.
**호출 조건:** 구현 작업 시작 전, 다중 접근 방식이 가능할 때.

```
Use this tool proactively when you're about to start a non-trivial implementation task.

## When to Use This Tool
1. New Feature Implementation
2. Multiple Valid Approaches
3. Code Modifications affecting existing behavior
4. Architectural Decisions
5. Multi-File Changes (more than 2-3 files)
6. Unclear Requirements
7. User Preferences Matter

## When NOT to Use This Tool
- Single-line or few-line fixes
- Tasks where the user has given very specific instructions
- Pure research/exploration tasks
```

---

## 16. ExitPlanModeTool (ExitPlanMode)

**소스 파일:** `src/tools/ExitPlanModeTool/prompt.ts`
**용도:** 계획 모드 종료 및 사용자 승인 요청. 계획 파일을 이미 작성한 후 사용.

```
Use this tool when you are in plan mode and have finished writing your plan to the plan file and are ready for user approval.

## How This Tool Works
- You should have already written your plan to the plan file
- This tool does NOT take the plan content as a parameter - it will read the plan from the file
- This tool simply signals that you're done planning and ready for the user to review

IMPORTANT: Only use this tool when the task requires planning the implementation steps of a task that requires writing code. For research tasks — do NOT use this tool.
```

---

## 17. AskUserQuestionTool (AskUserQuestion)

**소스 파일:** `src/tools/AskUserQuestionTool/prompt.ts`
**용도:** 사용자에게 다중 선택 질문. 선호도 수집, 모호한 지시 명확화, 구현 선택.

```
Use this tool when you need to ask the user questions during execution. This allows you to:
1. Gather user preferences or requirements
2. Clarify ambiguous instructions
3. Get decisions on implementation choices as you work
4. Offer choices to the user about what direction to take.

Usage notes:
- Users will always be able to select "Other" to provide custom text input
- Use multiSelect: true to allow multiple answers
- If you recommend a specific option, make that the first option and add "(Recommended)"

Plan mode note: In plan mode, use this tool to clarify requirements BEFORE finalizing your plan. Do NOT use this tool to ask "Is my plan ready?" - use ExitPlanMode for plan approval.
```

---

## 18. ToolSearchTool (ToolSearch)

**소스 파일:** `src/tools/ToolSearchTool/prompt.ts`
**용도:** 지연 로드(deferred) 도구의 스키마를 가져오는 도구. MCP 도구는 기본적으로 지연 로드됨.

```
Fetches full schema definitions for deferred tools so they can be called.

Deferred tools appear by name in <system-reminder> messages. Until fetched, only the name is known — there is no parameter schema, so the tool cannot be invoked.

Result format: each matched tool appears as one <function>...</function> line inside the <functions> block.

Query forms:
- "select:Read,Edit,Grep" — fetch these exact tools by name
- "notebook jupyter" — keyword search, up to max_results best matches
- "+slack send" — require "slack" in the name, rank by remaining terms
```

---

## 19. EnterWorktreeTool (EnterWorktree)

**소스 파일:** `src/tools/EnterWorktreeTool/prompt.ts`
**용도:** Git worktree를 생성하고 격리된 작업 환경으로 전환.

```
Use this tool ONLY when the user explicitly asks to work in a worktree.

## When to Use
- The user explicitly says "worktree"

## When NOT to Use
- The user asks to create a branch, switch branches — use git commands instead
- Never use this tool unless the user explicitly mentions "worktree"

## Behavior
- Creates a new git worktree inside `.claude/worktrees/` with a new branch based on HEAD
- Switches the session's working directory to the new worktree
```

---

## 20. ExitWorktreeTool (ExitWorktree)

**소스 파일:** `src/tools/ExitWorktreeTool/prompt.ts`
**용도:** Worktree 세션 종료 및 원래 디렉토리 복귀.

```
Exit a worktree session created by EnterWorktree and return the session to the original working directory.

## Parameters
- `action` (required): `"keep"` or `"remove"`
- `discard_changes` (optional, default false): only meaningful with `action: "remove"`.
```

---

## 21. SendMessageTool (SendMessage)

**소스 파일:** `src/tools/SendMessageTool/prompt.ts`
**용도:** 다른 에이전트(팀메이트)에게 메시지 전송. 팀 협업, 프로토콜 응답.

```
# SendMessage

Send a message to another agent.

| `to` | |
|---|---|
| `"researcher"` | Teammate by name |
| `"*"` | Broadcast to all teammates |

Your plain text output is NOT visible to other agents — to communicate, you MUST call this tool.
```

---

## 22. SleepTool (Sleep)

**소스 파일:** `src/tools/SleepTool/prompt.ts`
**용도:** 지정 시간 대기. 자율 모드에서 tick 사이 대기, 할 일이 없을 때 사용.

```
Wait for a specified duration. The user can interrupt the sleep at any time.

Use this when the user tells you to sleep or rest, when you have nothing to do, or when you're waiting for something.

You may receive <tick> prompts — these are periodic check-ins. Look for useful work to do before sleeping.

Prefer this over `Bash(sleep ...)` — it doesn't hold a shell process.
```

---

## 23. LSPTool (LSP)

**소스 파일:** `src/tools/LSPTool/prompt.ts`
**용도:** Language Server Protocol 연동. 심볼 정의, 참조 찾기, 호버 정보, 호출 계층 등.

```
Interact with Language Server Protocol (LSP) servers to get code intelligence features.

Supported operations:
- goToDefinition: Find where a symbol is defined
- findReferences: Find all references to a symbol
- hover: Get hover information (documentation, type info) for a symbol
- documentSymbol: Get all symbols in a document
- workspaceSymbol: Search for symbols across the entire workspace
- goToImplementation: Find implementations of an interface or abstract method
- prepareCallHierarchy: Get call hierarchy item at a position
- incomingCalls: Find all functions/methods that call the function at a position
- outgoingCalls: Find all functions/methods called by the function at a position
```

---

## 24. BriefTool (SendUserMessage)

**소스 파일:** `src/tools/BriefTool/prompt.ts`
**용도:** 자율 모드에서 사용자에게 메시지 전송. 상태(normal/proactive) 분류.
**호출 조건:** `feature('KAIROS') || feature('KAIROS_BRIEF')` 활성 시.

```
Send a message the user will read. Text outside this tool is visible in the detail view, but most won't open it — the answer lives here.

`message` supports markdown. `attachments` takes file paths for images, diffs, logs.

`status` labels intent: 'normal' when replying to what they just asked; 'proactive' when you're initiating.
```

---

## 25. PowerShellTool (PowerShell)

**소스 파일:** `src/tools/PowerShellTool/prompt.ts`
**용도:** Windows 환경용 PowerShell 명령 실행. PS 5.1/7+ 에디션 감지 및 구문 차이 안내.
**호출 조건:** Windows 플랫폼일 때 활성화.

---

## 26. ConfigTool (Config)

**소스 파일:** `src/tools/ConfigTool/prompt.ts`
**용도:** Claude Code 설정 읽기/쓰기. 테마, vim 모드, 모델, 권한 모드 등.

---

## 27. TaskGetTool / TaskListTool / TaskUpdateTool / TaskStopTool

**소스 파일:** `src/tools/Task*Tool/prompt.ts`
**용도:** 작업 목록 CRUD. TaskGet(상세 조회), TaskList(전체 목록), TaskUpdate(상태/소유자 변경), TaskStop(중지).

---

## 28. TeamCreateTool / TeamDeleteTool

**소스 파일:** `src/tools/TeamCreateTool/prompt.ts`, `src/tools/TeamDeleteTool/prompt.ts`
**용도:** 팀(에이전트 그룹) 생성/삭제. 팀 워크플로우, 작업 배분, 메시지 전달 프로토콜 정의.

---

## 29. RemoteTriggerTool / ScheduleCronTool

**소스 파일:** `src/tools/RemoteTriggerTool/prompt.ts`, `src/tools/ScheduleCronTool/prompt.ts`
**용도:** 원격 트리거 API 호출 및 크론 작업 스케줄링.

---

## 30. ListMcpResourcesTool / ReadMcpResourceTool

**소스 파일:** `src/tools/ListMcpResourcesTool/prompt.ts`, `src/tools/ReadMcpResourceTool/prompt.ts`
**용도:** MCP 서버의 리소스 목록 조회 및 읽기.
