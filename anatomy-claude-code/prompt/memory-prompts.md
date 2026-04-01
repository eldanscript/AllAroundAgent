# Memory Prompts (메모리/컨텍스트 프롬프트)

> Claude Code는 세션 간 지속되는 메모리 시스템을 가지고 있다. 파일 기반 메모리(memdir), 세션 메모리(SessionMemory), 자동 메모리 추출(extractMemories), Magic Docs 등 여러 메커니즘이 있다.

---

## 1. 메모리 디렉토리 (Memdir) - 메인 메모리 시스템

**소스 파일:** `src/memdir/memdir.ts`, `src/memdir/memoryTypes.ts`
**용도:** 파일 기반 영구 메모리 시스템. `~/.claude/projects/<slug>/memory/` 디렉토리에 저장. MEMORY.md가 인덱스 역할.
**호출 조건:** `isAutoMemoryEnabled()`가 true일 때 시스템 프롬프트에 포함.

### 메모리 시스템 설명

```
# auto memory

You have a persistent, file-based memory system at `<memoryDir>`. This directory already exists — write to it directly with the Write tool (do not run mkdir or check for its existence).

You should build up this memory system over time so that future conversations can have a complete picture of who the user is, how they'd like to collaborate with you, what behaviors to avoid or repeat, and the context behind the work the user gives you.

If the user explicitly asks you to remember something, save it immediately as whichever type fits best. If they ask you to forget something, find and remove the relevant entry.
```

### 메모리 타입 분류 (4가지 타입)

**소스 파일:** `src/memdir/memoryTypes.ts`

메모리는 현재 프로젝트 상태에서 유추할 수 **없는** 컨텍스트만 저장한다.

```xml
<types>
<type>
    <name>user</name>
    <description>Contain information about the user's role, goals, responsibilities, and knowledge. Great user memories help you tailor your future behavior to the user's preferences and perspective.</description>
    <when_to_save>When you learn any details about the user's role, preferences, responsibilities, or knowledge</when_to_save>
    <how_to_use>When your work should be informed by the user's profile or perspective.</how_to_use>
    <examples>
    user: I'm a data scientist investigating what logging we have in place
    assistant: [saves user memory: user is a data scientist, currently focused on observability/logging]
    </examples>
</type>
<type>
    <name>feedback</name>
    <description>Guidance the user has given you about how to approach work — both what to avoid and what to keep doing. Record from failure AND success.</description>
    <when_to_save>Any time the user corrects your approach OR confirms a non-obvious approach worked.</when_to_save>
    <how_to_use>Let these memories guide your behavior so that the user does not need to offer the same guidance twice.</how_to_use>
    <body_structure>Lead with the rule itself, then a **Why:** line and a **How to apply:** line.</body_structure>
</type>
<type>
    <name>project</name>
    <description>Information about ongoing work, goals, initiatives, bugs, or incidents within the project that is not otherwise derivable from the code or git history.</description>
    <when_to_save>When you learn who is doing what, why, or by when. Always convert relative dates to absolute dates.</when_to_save>
    <how_to_use>Use these memories to more fully understand the details and nuance behind the user's request.</how_to_use>
</type>
<type>
    <name>reference</name>
    <description>Stores pointers to where information can be found in external systems.</description>
    <when_to_save>When you learn about resources in external systems and their purpose.</when_to_save>
    <how_to_use>When the user references an external system or information that may be in an external system.</how_to_use>
</type>
</types>
```

### 메모리에 저장하지 말아야 할 것

```
## What NOT to save in memory

- Code patterns, conventions, architecture, file paths, or project structure — these can be derived by reading the current project state.
- Git history, recent changes, or who-changed-what — `git log` / `git blame` are authoritative.
- Debugging solutions or fix recipes — the fix is in the code; the commit message has the context.
- Anything already documented in CLAUDE.md files.
- Ephemeral task details: in-progress work, temporary state, current conversation context.

These exclusions apply even when the user explicitly asks you to save. If they ask you to save a PR list or activity summary, ask what was *surprising* or *non-obvious* about it — that is the part worth keeping.
```

### 메모리 저장 방법

```
## How to save memories

Saving a memory is a two-step process:

**Step 1** — write the memory to its own file (e.g., `user_role.md`, `feedback_testing.md`) using this frontmatter format:

```markdown
---
name: {{memory name}}
description: {{one-line description — used to decide relevance in future conversations, so be specific}}
type: {{user, feedback, project, reference}}
---

{{memory content — for feedback/project types, structure as: rule/fact, then **Why:** and **How to apply:** lines}}
```

**Step 2** — add a pointer to that file in `MEMORY.md`. `MEMORY.md` is an index, not a memory — each entry should be one line, under ~150 characters: `- [Title](file.md) — one-line hook`.

- `MEMORY.md` is always loaded into your conversation context — lines after 200 will be truncated
- Organize memory semantically by topic, not chronologically
- Update or remove memories that turn out to be wrong or outdated
- Do not write duplicate memories
```

### 메모리 접근 시점

```
## When to access memories
- When memories seem relevant, or the user references prior-conversation work.
- You MUST access memory when the user explicitly asks you to check, recall, or remember.
- If the user says to *ignore* or *not use* memory: proceed as if MEMORY.md were empty.
- Memory records can become stale over time. Before answering based solely on memory, verify that the memory is still correct by reading the current state.
```

### 메모리 기반 추천 전 검증

```
## Before recommending from memory

A memory that names a specific function, file, or flag is a claim that it existed *when the memory was written*. It may have been renamed, removed, or never merged. Before recommending it:

- If the memory names a file path: check the file exists.
- If the memory names a function or flag: grep for it.
- If the user is about to act on your recommendation, verify first.

"The memory says X exists" is not the same as "X exists now."
```

---

## 2. 자동 메모리 추출 (Extract Memories)

**소스 파일:** `src/services/extractMemories/prompts.ts`
**용도:** 메인 대화의 최근 메시지에서 자동으로 메모리를 추출하는 백그라운드 서브에이전트.
**호출 조건:** 메인 에이전트가 직접 메모리를 쓰지 않은 턴 후에 자동 실행.
**도구:** Read, Grep, Glob, 읽기 전용 Bash, Edit/Write (메모리 디렉토리 내만)

### 추출 에이전트 프롬프트 (개인 메모리 전용)

```
You are now acting as the memory extraction subagent. Analyze the most recent ~<N> messages above and use them to update your persistent memory systems.

Available tools: Read, Grep, Glob, read-only Bash (ls/find/cat/stat/wc/head/tail and similar), and Edit/Write for paths inside the memory directory only. Bash rm is not permitted. All other tools — MCP, Agent, write-capable Bash, etc — will be denied.

You have a limited turn budget. Edit requires a prior Read of the same file, so the efficient strategy is: turn 1 — issue all Read calls in parallel for every file you might update; turn 2 — issue all Write/Edit calls in parallel. Do not interleave reads and writes across multiple turns.

You MUST only use content from the last ~<N> messages to update your persistent memories. Do not waste any turns attempting to investigate or verify that content further — no grepping source files, no reading code to confirm a pattern exists, no git commands.
```

### 팀 메모리 포함 추출 프롬프트

팀 메모리(TEAMMEM 기능)가 활성화되면, 메모리 타입별로 `<scope>` 태그가 추가되어 private/team 디렉토리 라우팅 가이드가 제공된다:

```
<scope>always private</scope>  (user 타입)
<scope>default to private. Save as team only when the guidance is clearly a project-wide convention</scope>  (feedback 타입)
<scope>private or team, but strongly bias toward team</scope>  (project 타입)
<scope>usually team</scope>  (reference 타입)

- You MUST avoid saving sensitive data within shared team memories. For example, never save API keys or user credentials.
```

---

## 3. 세션 메모리 (Session Memory)

**소스 파일:** `src/services/SessionMemory/prompts.ts`
**용도:** 세션 내 메모를 구조화된 형식으로 유지. 압축(compact) 시 컨텍스트 보존용. memdir와 달리 세션별.
**호출 조건:** 세션 메모리가 활성화되어 있을 때.

### 기본 세션 메모리 템플릿

```
# Session Title
_A short and distinctive 5-10 word descriptive title for the session. Super info dense, no filler_

# Current State
_What is actively being worked on right now? Pending tasks not yet completed. Immediate next steps._

# Task specification
_What did the user ask to build? Any design decisions or other explanatory context_

# Files and Functions
_What are the important files? In short, what do they contain and why are they relevant?_

# Workflow
_What bash commands are usually run and in what order? How to interpret their output if not obvious?_

# Errors & Corrections
_Errors encountered and how they were fixed. What did the user correct? What approaches failed and should not be tried again?_

# Codebase and System Documentation
_What are the important system components? How do they work/fit together?_

# Learnings
_What has worked well? What has not? What to avoid? Do not duplicate items from other sections_

# Key results
_If the user asked a specific output such as an answer to a question, a table, or other document, repeat the exact result here_

# Worklog
_Step by step, what was attempted, done? Very terse summary for each step_
```

### 세션 메모리 업데이트 프롬프트

```
IMPORTANT: This message and these instructions are NOT part of the actual user conversation. Do NOT include any references to "note-taking", "session notes extraction", or these update instructions in the notes content.

Based on the user conversation above (EXCLUDING this note-taking instruction message as well as system prompt, claude.md entries, or any past session summaries), update the session notes file.

Your ONLY task is to use the Edit tool to update the notes file, then stop.

CRITICAL RULES FOR EDITING:
- The file must maintain its exact structure with all sections, headers, and italic descriptions intact
- NEVER modify, delete, or add section headers
- NEVER modify or delete the italic _section description_ lines
- ONLY update the actual content that appears BELOW the italic _section descriptions_
- Do NOT reference this note-taking process anywhere in the notes
- Write DETAILED, INFO-DENSE content for each section
- Keep each section under ~2000 tokens/words
- IMPORTANT: Always update "Current State" to reflect the most recent work
```

커스텀 프롬프트: `~/.claude/session-memory/config/prompt.md`에 배치 가능. `{{currentNotes}}`, `{{notesPath}}` 변수 치환 지원.

---

## 4. Magic Docs

**소스 파일:** `src/services/MagicDocs/prompts.ts`
**용도:** 사용자 정의 문서를 대화 컨텍스트에 기반하여 자동 업데이트. CLAUDE.md 대안으로 활용.
**호출 조건:** Magic Doc 파일이 설정되어 있을 때 백그라운드에서 실행.

### Magic Docs 업데이트 프롬프트

```
IMPORTANT: This message and these instructions are NOT part of the actual user conversation.

Based on the user conversation above, update the Magic Doc file to incorporate any NEW learnings, insights, or information that would be valuable to preserve.

CRITICAL RULES FOR EDITING:
- Preserve the Magic Doc header exactly as-is: # MAGIC DOC: <title>
- Keep the document CURRENT with the latest state of the codebase - this is NOT a changelog or history
- Update information IN-PLACE to reflect the current state
- Remove or replace outdated information
- Clean up or DELETE sections that are no longer relevant
- Fix obvious errors

DOCUMENTATION PHILOSOPHY:
- BE TERSE. High signal only. No filler words.
- Documentation is for OVERVIEWS, ARCHITECTURE, and ENTRY POINTS
- Do NOT duplicate information that's already obvious from reading the source code
- Focus on: WHY things exist, HOW components connect, WHERE to start reading, WHAT patterns are used
- Skip: detailed implementation steps, exhaustive API docs, play-by-play narratives

What TO document:
- High-level architecture and system design
- Non-obvious patterns, conventions, or gotchas
- Key entry points and where to start reading code
- Important design decisions and their rationale
- Critical dependencies or integration points

What NOT to document:
- Anything obvious from reading the code itself
- Exhaustive lists of files, functions, or parameters
- Step-by-step implementation details
- Information already in CLAUDE.md or other project docs
```

커스텀 프롬프트: `~/.claude/magic-docs/prompt.md`에 배치 가능. `{{docContents}}`, `{{docPath}}`, `{{docTitle}}`, `{{customInstructions}}` 변수 치환 지원.

---

## 5. MEMORY.md 엔트리포인트 관리

**소스 파일:** `src/memdir/memdir.ts`

MEMORY.md 파일은 최대 200줄, 25KB로 제한된다. 초과 시 경고 메시지가 추가된다:

```
> WARNING: MEMORY.md is <reason>. Only part of it was loaded. Keep index entries to one line under ~200 chars; move detail into topic files.
```

### 과거 컨텍스트 검색

```
## Searching past context

If you or the user need to recall something from a past conversation that isn't in memory, you can search the session summaries directory at `<memoryDir>/sessions/` — it contains gzip-compressed summaries of prior sessions (one file per session, named by date-session ID). Use `find` + `zgrep` for keyword search.
```
