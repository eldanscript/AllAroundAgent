# Compact Prompts (압축/요약 관련 프롬프트)

> 컨텍스트 윈도우 한계에 도달하면 이전 대화를 요약하여 세션을 지속할 수 있게 한다. 전체(base) 압축과 부분(partial) 압축, 두 가지 모드가 있다.

---

## 1. 전체 압축 프롬프트 (Base Compact Prompt)

**소스 파일:** `src/services/compact/prompt.ts`
**함수:** `getCompactPrompt()`
**용도:** 전체 대화를 하나의 요약으로 압축. 컨텍스트 한계 도달 시 또는 사용자가 `/compact` 실행 시.
**호출 조건:** 컨텍스트 윈도우가 가득 찼을 때 자동 실행.

### No-Tools 프리앰블

모든 압축 프롬프트 앞에 붙는 도구 사용 금지 지시:

```
CRITICAL: Respond with TEXT ONLY. Do NOT call any tools.

- Do NOT use Read, Bash, Grep, Glob, Edit, Write, or ANY other tool.
- You already have all the context you need in the conversation above.
- Tool calls will be REJECTED and will waste your only turn — you will fail the task.
- Your entire response must be plain text: an <analysis> block followed by a <summary> block.
```

### 분석 지침 (Base)

```
Before providing your final summary, wrap your analysis in <analysis> tags to organize your thoughts and ensure you've covered all necessary points. In your analysis process:

1. Chronologically analyze each message and section of the conversation. For each section thoroughly identify:
   - The user's explicit requests and intents
   - Your approach to addressing the user's requests
   - Key decisions, technical concepts and code patterns
   - Specific details like:
     - file names
     - full code snippets
     - function signatures
     - file edits
   - Errors that you ran into and how you fixed them
   - Pay special attention to specific user feedback that you received, especially if the user told you to do something differently.
2. Double-check for technical accuracy and completeness, addressing each required element thoroughly.
```

### 메인 프롬프트

```
Your task is to create a detailed summary of the conversation so far, paying close attention to the user's explicit requests and your previous actions.
This summary should be thorough in capturing technical details, code patterns, and architectural decisions that would be essential for continuing development work without losing context.

Your summary should include the following sections:

1. Primary Request and Intent: Capture all of the user's explicit requests and intents in detail
2. Key Technical Concepts: List all important technical concepts, technologies, and frameworks discussed.
3. Files and Code Sections: Enumerate specific files and code sections examined, modified, or created. Pay special attention to the most recent messages and include full code snippets where applicable.
4. Errors and fixes: List all errors that you ran into, and how you fixed them.
5. Problem Solving: Document problems solved and any ongoing troubleshooting efforts.
6. All user messages: List ALL user messages that are not tool results.
7. Pending Tasks: Outline any pending tasks that you have explicitly been asked to work on.
8. Current Work: Describe in detail precisely what was being worked on immediately before this summary request.
9. Optional Next Step: List the next step that you will take that is related to the most recent work you were doing. IMPORTANT: ensure that this step is DIRECTLY in line with the user's most recent explicit requests.
```

### No-Tools 트레일러

```
REMINDER: Do NOT call any tools. Respond with plain text only — an <analysis> block followed by a <summary> block. Tool calls will be rejected and you will fail the task.
```

---

## 2. 부분 압축 프롬프트 - "from" 방향 (Partial Compact - From)

**함수:** `getPartialCompactPrompt(customInstructions, 'from')`
**용도:** 이전 컨텍스트는 유지하고 최근 메시지 부분만 요약. 부분 컴팩트 시 사용.
**호출 조건:** 부분 압축 모드에서 최근 메시지를 요약할 때.

```
Your task is to create a detailed summary of the RECENT portion of the conversation — the messages that follow earlier retained context. The earlier messages are being kept intact and do NOT need to be summarized. Focus your summary on what was discussed, learned, and accomplished in the recent messages only.

Your summary should include the following sections:

1. Primary Request and Intent: Capture the user's explicit requests and intents from the recent messages
2. Key Technical Concepts: List important technical concepts discussed recently.
3. Files and Code Sections: Enumerate specific files and code sections examined, modified, or created.
4. Errors and fixes: List errors encountered and how they were fixed.
5. Problem Solving: Document problems solved and ongoing troubleshooting.
6. All user messages: List ALL user messages from the recent portion.
7. Pending Tasks: Outline pending tasks from the recent messages.
8. Current Work: Describe precisely what was being worked on immediately before this summary request.
9. Optional Next Step: List the next step related to the most recent work.
```

---

## 3. 부분 압축 프롬프트 - "up_to" 방향 (Partial Compact - Up To)

**함수:** `getPartialCompactPrompt(customInstructions, 'up_to')`
**용도:** 이전 대화를 요약하되, 이 요약 뒤에 최근 메시지가 이어질 것을 전제. 새로운 메시지를 이해하기 위한 컨텍스트 보존.
**호출 조건:** 캐시 히트 최적화를 위해 요약 대상을 모델이 이미 본 메시지로 제한할 때.

```
Your task is to create a detailed summary of this conversation. This summary will be placed at the start of a continuing session; newer messages that build on this context will follow after your summary (you do not see them here). Summarize thoroughly so that someone reading only your summary and then the newer messages can fully understand what happened and continue the work.

Your summary should include the following sections:

1. Primary Request and Intent
2. Key Technical Concepts
3. Files and Code Sections
4. Errors and fixes
5. Problem Solving
6. All user messages
7. Pending Tasks
8. Work Completed: Describe what was accomplished by the end of this portion.
9. Context for Continuing Work: Summarize any context, decisions, or state that would be needed to understand and continue the work in subsequent messages.
```

---

## 4. 압축 요약 사용자 메시지 (Compact User Summary Message)

**함수:** `getCompactUserSummaryMessage()`
**용도:** 압축 완료 후 사용자에게 보여지는 요약 메시지. 이전 대화가 압축되었음을 안내.
**호출 조건:** 압축 후 새 대화 시작 시.

```
This session is being continued from a previous conversation that ran out of context. The summary below covers the earlier portion of the conversation.

<formatted_summary>

If you need specific details from before compaction (like exact code snippets, error messages, or content you generated), read the full transcript at: <transcriptPath>

Recent messages are preserved verbatim.
```

### 후속 질문 억제 모드

`suppressFollowUpQuestions`가 true일 때 (자동 압축 시):

```
Continue the conversation from where it left off without asking the user any further questions. Resume directly — do not acknowledge the summary, do not recap what was happening, do not preface with "I'll continue" or similar. Pick up the last task as if the break never happened.
```

### 자율 모드에서의 압축 후 메시지

```
You are running in autonomous/proactive mode. This is NOT a first wake-up — you were already working autonomously before compaction. Continue your work loop: pick up where you left off based on the summary above. Do not greet the user or ask what to work on.
```

---

## 5. 요약 포매팅 (formatCompactSummary)

**함수:** `formatCompactSummary()`
**용도:** 원시 요약에서 `<analysis>` 블록(드래프팅 스크래치패드)을 제거하고 `<summary>` 태그를 읽기 쉬운 형식으로 변환.

동작:
1. `<analysis>...</analysis>` 섹션 제거 (요약 품질 향상을 위한 사고 과정이므로 최종 출력에 불필요)
2. `<summary>...</summary>` 태그를 `Summary:` 헤더로 교체
3. 연속된 빈 줄 정리

---

## 6. 사용자 정의 압축 지침

사용자가 CLAUDE.md 등에서 다음과 같은 형식으로 커스텀 압축 지침을 제공할 수 있다:

```
## Compact Instructions
When summarizing the conversation focus on typescript code changes and also remember the mistakes you made and how you fixed them.
```

```
# Summary instructions
When you are using compact - please focus on test output and code changes. Include file reads verbatim.
```

이 지침은 `Additional Instructions:` 섹션으로 프롬프트 끝에 추가된다.
