# Claude Code 프롬프트 엔지니어링 Best Practice 분석

> **분석 대상:** Claude Code 시스템 프롬프트, 도구 프롬프트, 에이전트 프롬프트, 압축 프롬프트, 메모리 프롬프트, 보안 프롬프트
> **분석 기준일:** 2026-03-31
> **분석 관점:** 프롬프트 엔지니어링 기법 추출 및 실전 적용 가이드

---

## 1. 분석 개요

### 분석 범위

| 프롬프트 카테고리 | 파일 수 | 주요 내용 |
|---|---|---|
| 시스템 프롬프트 | 1 | 14개 섹션, 정적/동적 분리 구조 |
| 도구 프롬프트 | 30+ | BashTool, Read, Edit, Write, Grep, Glob, Agent, WebFetch 등 |
| 에이전트 프롬프트 | 6 | Explore, Plan, Default, Companion, Verification, Team |
| 압축 프롬프트 | 5 | 전체/부분 압축, 포매팅, 사용자 메시지 |
| 메모리 프롬프트 | 5 | Memdir, Extract, Session Memory, Magic Docs |
| 보안 프롬프트 | 6 | YOLO Classifier, Cyber Risk, Sandbox, Git Safety, Undercover |

### 발견된 기법 수

- **기존 분석(analysis_example.md):** 83개 기법, 10개 카테고리
- **추가 발견 기법:** 22개 (아래 카탈로그에 ★로 표시)
- **총 기법 수:** 105개

---

## 2. 핵심 기법 카탈로그

### 기법 전체 목록 (카테고리별)

| # | 카테고리 | 기법 | 출처 |
|---|---|---|---|
| 1.1 | 강조/주의 | CAPS for Behavioral Locks | BashTool |
| 1.2 | 강조/주의 | IMPORTANT/CRITICAL Prefix | compact, system |
| 1.3 | 강조/주의 | Triple-Redundancy Pattern | compact |
| ★1.4 | 강조/주의 | 등호 배너 강조 (=== CRITICAL ===) | exploreAgent |
| ★1.5 | 강조/주의 | 실패 결과 명시 ("will error", "will be REJECTED") | FileEditTool, compact |
| 2.1 | 구조적 | Hierarchical Section Headers | system prompt |
| 2.2 | 구조적 | XML Tags as Scratchpad → Strip | compact |
| 2.3 | 구조적 | Cache Boundary Marker | system prompt |
| 2.4 | 구조적 | Structured Environment Context | system prompt |
| ★2.5 | 구조적 | Frontmatter 메타데이터 | memdir |
| ★2.6 | 구조적 | 2단계 파이프라인 (Stage 1 → Stage 2) | YOLO classifier |
| ★2.7 | 구조적 | 템플릿 변수 치환 (`{{varName}}`) | Magic Docs, Session Memory |
| 3.1 | 행동 제어 | When/When NOT Bifurcation | EnterPlanMode, TaskCreate |
| 3.2 | 행동 제어 | Anti-Gold-Plating Heuristic | system prompt |
| 3.3 | 행동 제어 | Mandatory Verification Gate | ant-only |
| 3.4 | 행동 제어 | Honest Reporting Mandate | ant-only |
| 3.5 | 행동 제어 | Risk-Based Authorization Framework | system prompt |
| ★3.6 | 행동 제어 | 후속 질문 억제 모드 | compact |
| ★3.7 | 행동 제어 | 메타 설명 금지 ("Don't explain that you're not X") | buddy |
| ★3.8 | 행동 제어 | 진단 우선 원칙 ("diagnose why before switching") | system prompt |
| 4.1 | 환각 방지 | Precondition Chain | FileEditTool |
| 4.2 | 환각 방지 | Fabrication Ban with Form Listing | AgentTool |
| 4.3 | 환각 방지 | Snapshot Semantics Disclaimer | context.ts |
| 4.4 | 환각 방지 | Knowledge Cutoff Declaration | system prompt |
| ★4.5 | 환각 방지 | 메모리 검증 의무 ("check the file exists") | memdir |
| ★4.6 | 환각 방지 | 저장 제외 규칙 ("What NOT to save") | memdir |
| 5.1 | 도구 선택 | Explicit Preference Hierarchy | BashTool |
| 5.2 | 도구 선택 | Search vs Lookup Distinction | AgentTool |
| 5.3 | 도구 선택 | Capability-Based Agent Assignment | TeamCreate |
| ★5.4 | 도구 선택 | MCP 우선순위 ("prefer using that tool instead") | WebFetchTool |
| ★5.5 | 도구 선택 | 도구 건너뛰기 시그널 (toAutoClassifierInput → '') | YOLO classifier |
| 6.1 | 다중 에이전트 | Visibility Boundary Declaration | SendMessage |
| 6.2 | 다중 에이전트 | Idleness Normalization | TeamCreate |
| 6.3 | 다중 에이전트 | Fork vs Subagent Semantics | AgentTool |
| 6.4 | 다중 에이전트 | Don't Peek (Context Pollution) | AgentTool |
| ★6.5 | 다중 에이전트 | 팀 셧다운 프로토콜 | TeamCreate |
| ★6.6 | 다중 에이전트 | 결과 불가시성 명시 ("not visible to the user") | AgentTool |
| 7.1 | 구체적 제약 | Numeric Length Anchors (≤25 words) | ant-only |
| 7.2 | 구체적 제약 | Token Budget as Task Variable | system prompt |
| 7.3 | 구체적 제약 | Character Limits for Index Files | memdir |
| ★7.4 | 구체적 제약 | 줄 수 트렁케이션 ("lines after 200 will be truncated") | memdir |
| ★7.5 | 구체적 제약 | 토큰 제약 ("under ~2000 tokens/words") | Session Memory |
| ★7.6 | 구체적 제약 | max_tokens 하드 리밋 (Stage 1: 64 tokens) | YOLO classifier |
| 8.1 | 역할/정체성 | Collaborator vs Executor Framing | ant-only |
| 8.2 | 역할/정체성 | Read-Only Agent Identity | exploreAgent |
| 8.3 | 역할/정체성 | Companion Non-Interference | buddy |
| ★8.4 | 역할/정체성 | 전문가 역할 지정 ("file search specialist") | exploreAgent |
| ★8.5 | 역할/정체성 | 아키텍트 역할 지정 ("software architect") | planAgent |
| 9.1 | 조건부 적응 | User-Type Branching (ant vs external) | system prompt |
| 9.2 | 조건부 적응 | Terminal Focus Adaptation | proactive mode |
| 9.3 | 조건부 적응 | Feature-Flag Gated Sections | system prompt |
| 9.4 | 조건부 적응 | Dynamic Warning Generation | SessionMemory |
| ★9.5 | 조건부 적응 | 플랫폼별 분기 (PowerShell 거부 규칙) | YOLO classifier |
| ★9.6 | 조건부 적응 | 자율 모드 압축 후 컨텍스트 전환 | compact |
| 10.1 | 효율성/메타인지 | Turn Budget Strategy Teaching | extractMemories |
| 10.2 | 효율성/메타인지 | Speed-First Agent Instruction | exploreAgent |
| 10.3 | 효율성/메타인지 | Forced Sleep for Token Conservation | proactive |
| 10.4 | 효율성/메타인지 | Contrastive Guidance Pairs | MagicDocs |
| 10.5 | 효율성/메타인지 | Safe Variable Injection | MagicDocs |
| ★10.6 | 효율성/메타인지 | 2턴 최적 전략 ("turn 1 — Read, turn 2 — Write") | extractMemories |
| ★10.7 | 효율성/메타인지 | 캐시 히트 최적화 (부분 압축 방향) | compact |

---

## 3. 카테고리별 Best Practice

### 3.1 강조/주의 제어 (Emphasis & Attention Control)

AI 모델이 특정 지시를 무시하지 않도록 시각적, 의미적 가중치를 높이는 기법들이다.

#### 기법 1: CAPS Lock 패턴

가장 기본적이면서도 가장 빈번하게 사용되는 기법. BashTool 프롬프트에서만 "NEVER"가 12회 이상 등장한다.

```
"NEVER update the git config"
"NEVER run destructive git commands"
"NEVER skip hooks (--no-verify)"
"NEVER commit changes unless the user explicitly asks"
```

**원칙:** 대문자는 시각적 돌출(visual salience)과 의미적 가중치(semantic weight)를 동시에 높인다. 모델은 대문자 금지사항을 일반 텍스트보다 높은 우선순위 제약으로 처리한다.

**적용 기준:**
- 위반 시 되돌릴 수 없는 결과가 발생하는 행동에만 사용
- 남용하면 효과가 희석되므로, 프롬프트당 5~10개 이내로 제한

#### 기법 2: IMPORTANT/CRITICAL Prefix

```
"CRITICAL: Respond with TEXT ONLY. Do NOT call any tools."
"IMPORTANT: Go straight to the point."
"CRITICAL RULES FOR EDITING:"
```

**원칙:** 접두어가 모델의 주의를 초기에 잡는다. 특히 "CRITICAL"은 실패 결과("you will fail the task")와 결합하면 제약이 강화된다.

#### 기법 3: 삼중 반복 패턴 (Triple-Redundancy)

압축 프롬프트에서 도구 사용 금지가 3곳에 배치된다:

1. **프리앰블:** `"CRITICAL: Respond with TEXT ONLY. Do NOT call any tools."`
2. **본문:** 도구별 금지 목록과 실패 결과 명시
3. **트레일러:** `"REMINDER: Do NOT call any tools. Respond with plain text only"`

**원칙:** Recency bias(최근성 편향)를 활용한다. 트레일러는 컨텍스트에서 가장 최근이고, 프리앰블은 첫 읽기 주의를 담당한다. 세 번 반복은 단일 강한 문장보다 효과적이다.

#### ★ 기법 4: 등호 배너 강조

Explore Agent에서 발견된 패턴. 등호로 둘러싼 배너가 시각적으로 특히 돋보인다:

```
=== CRITICAL: READ-ONLY MODE - NO FILE MODIFICATIONS ===
```

**원칙:** 텍스트 기반 UI에서의 "경고 박스" 역할. 등호선이 시각적 분리자로 작용하여 모델이 이 지시를 독립된 블록으로 인식하게 만든다.

#### ★ 기법 5: 실패 결과 명시

```
"This tool will error if you attempt an edit without reading the file."
"Tool calls will be REJECTED and will waste your only turn — you will fail the task."
```

**원칙:** 추상적 금지("하지 마라")보다 구체적 결과("에러가 난다", "작업을 실패한다")가 더 강력하다. 모델이 비용-편익 분석을 하도록 유도한다.

---

### 3.2 구조적 패턴 (Structural Patterns)

프롬프트의 물리적 구조가 모델의 정보 처리 방식에 영향을 미치는 기법들이다.

#### 기법 1: 계층적 섹션 헤더

시스템 프롬프트가 명확한 계층 구조로 조직된다:

```
# System
# Doing tasks
# Executing actions with care
# Using your tools
# Tone and style
# Output efficiency
```

**원칙:** 마크다운 헤더가 인지적 버킷(cognitive bucket)을 만들어, 모델이 서로 다른 지시 영역을 독립적으로 참조하고 우선순위를 정할 수 있다.

#### 기법 2: XML 태그 Scratchpad → Strip

```
"wrap your analysis in <analysis> tags"
// 이후 코드에서:
formattedSummary.replace(/<analysis>[\s\S]*?<\/analysis>/, '')
```

**원칙:** Chain-of-Thought Stripping 패턴. `<analysis>` 태그 안에서 모델이 사고하도록 강제한 뒤, post-processing으로 제거하면 추론 품질은 유지하면서 출력은 깔끔해진다. Anthropic이 자사 제품에서 사용하는 핵심 기법이다.

**적용 방법:**
1. 모델에게 `<analysis>` (또는 `<thinking>`) 태그 안에서 분석하도록 지시
2. 최종 출력은 `<summary>` (또는 `<answer>`) 태그에 넣도록 지시
3. 코드에서 분석 태그를 정규식으로 제거

#### 기법 3: 캐시 경계 마커

```typescript
export const SYSTEM_PROMPT_DYNAMIC_BOUNDARY = '**SYSTEM_PROMPT_DYNAMIC_BOUNDARY**'
```

시스템 프롬프트 조립 순서:
1. **정적 영역 (캐시 가능):** 인트로, 시스템 규칙, 작업 지침, 도구 사용법, 톤/스타일, 출력 효율성
2. **`SYSTEM_PROMPT_DYNAMIC_BOUNDARY`**
3. **동적 영역 (세션별):** 세션 가이던스, 메모리, 환경 정보, MCP 지침

**원칙:** 메타-프롬프팅. 프롬프트 자체가 자신의 처리 방법(캐시 여부)을 결정하는 구조를 포함한다. API 비용 최적화의 핵심이다.

#### ★ 기법 4: Frontmatter 메타데이터

메모리 파일에 YAML frontmatter를 사용하여 구조화된 메타데이터를 부여한다:

```markdown
---
name: {{memory name}}
description: {{one-line description}}
type: {{user, feedback, project, reference}}
---
{{memory content}}
```

**원칙:** 비정형 텍스트에 기계 가독성을 부여한다. `description` 필드가 향후 검색 시 관련성 판단 기준이 되므로, "be specific"이라는 추가 지시가 붙는다.

#### ★ 기법 5: 2단계 분류 파이프라인

YOLO Classifier가 2단계로 작동한다:

- **Stage 1 (Fast):** max_tokens=64. 즉각적인 허용/차단 결정.
  ```
  "Err on the side of blocking. <block> immediately."
  ```
- **Stage 2 (Thinking):** Stage 1에서 차단된 경우만 실행. 사고 과정을 포함한 재평가로 false positive 감소.
  ```
  "Review the classification process and follow it carefully"
  ```

**원칙:** 비용과 정확도의 트레이드오프를 구조적으로 해결한다. 대부분의 요청은 Stage 1에서 저비용으로 처리되고, 경계 사례만 Stage 2에서 정밀 분석된다.

---

### 3.3 행동 제어 (Behavioral Control)

모델의 행동을 원하는 방향으로 유도하고 바람직하지 않은 행동을 억제하는 기법들이다.

#### 기법 1: When/When NOT 이분법

TaskCreate와 EnterPlanMode에서 일관되게 사용되는 패턴:

```
## When to Use This Tool
- Complex multi-step tasks
- Non-trivial and complex tasks
- Plan mode

## When NOT to Use This Tool
- There is only a single, straightforward task
- The task is trivial
- The task can be completed in less than 3 trivial steps
```

**원칙:** 긍정/부정 사례를 명시적으로 분리하여, 도구 남용과 과소 사용을 동시에 방지한다. "3 trivial steps"라는 구체적 수치가 판단 기준을 제공한다.

#### 기법 2: Anti-Gold-Plating 경험 법칙

```
"Three similar lines of code is better than a premature abstraction."
"Don't add features, refactor code, or make 'improvements' beyond what was asked."
"Don't add error handling, fallbacks, or validation for scenarios that can't happen."
```

**원칙:** 모호한 "과도하게 하지 마라" 대신 구체적 비율(3줄 vs 추상화)과 금지 목록을 제공한다. LLM은 기본적으로 과잉 생산(over-generation) 경향이 있으므로, 이를 명시적으로 억제하는 것이 필수적이다.

#### 기법 3: 위험 기반 인가 프레임워크

```
"Carefully consider the reversibility and blast radius of actions."
```

위험도별 행동 분류:
- **자유 실행:** 로컬, 가역적 행동 (파일 편집, 테스트 실행)
- **사용자 확인 필요:** 파괴적, 비가역적, 외부 영향 행동 (force push, DB drop, PR 생성)

**원칙:** 단순 이진 규칙("허용/금지") 대신 원칙적 프레임워크("가역성 + 영향 범위")를 제공한다. 새로운 상황에서도 모델이 원칙을 적용하여 판단할 수 있다.

#### ★ 기법 4: 진단 우선 원칙

```
"If an approach fails, diagnose why before switching tactics—read the error, check your assumptions, try a focused fix."
"When you encounter an obstacle, do not use destructive actions as a shortcut to simply make it go away."
```

**원칙:** LLM의 "안 되면 다른 거 시도" 패턴을 억제한다. 특히 "investigate before deleting or overwriting, as it may represent the user's in-progress work"는 파괴적 지름길의 구체적 위험을 설명한다.

#### ★ 기법 5: 후속 질문 억제

자동 압축 후 대화 재개 시:

```
"Continue the conversation from where it left off without asking the user any further questions. Resume directly — do not acknowledge the summary, do not recap what was happening, do not preface with 'I'll continue' or similar. Pick up the last task as if the break never happened."
```

**원칙:** 모델의 "친절한 확인" 습관을 명시적으로 차단한다. "as if the break never happened"라는 프레이밍이 원하는 행동을 자연어로 정확히 기술한다.

---

### 3.4 환각 방지 (Anti-Hallucination)

모델이 존재하지 않는 정보를 생성하거나 오래된 정보에 의존하는 것을 방지하는 기법들이다.

#### 기법 1: 전제조건 체인

```
"You must use your Read tool at least once before editing. This tool will error if you attempt an edit without reading the file."
```

FileWrite에서도 동일:
```
"If this is an existing file, you MUST use the Read tool first to read the file's contents. This tool will fail if you did not read the file first."
```

**원칙:** "will error"/"will fail"이라는 보장된 실패 선언이 지름길 시도를 방지한다. 시스템이 실제로 이 전제조건을 강제하므로, 프롬프트와 코드가 일관된다.

#### 기법 2: 날조 금지 + 형태 열거

```
"Never fabricate or predict fork results in any format — not as prose, summary, or structured output."
```

**원칙:** 금지할 행동뿐 아니라 그것이 나타날 수 있는 **형태**(prose, summary, structured output)까지 열거한다. LLM은 "직접 답하지 마라"는 지시를 우회하여 요약이나 구조화된 형식으로 날조할 수 있으므로, 모든 출구를 막는다.

#### 기법 3: 스냅샷 시맨틱스 면책

```
"This is the git status at the start of the conversation. Note that this status is a snapshot in time, and will not update during the conversation."
```

**원칙:** 시간에 민감한 정보의 유효 범위를 명시적으로 선언한다. 모델이 오래된 상태 정보를 현재 사실로 취급하는 환각을 방지한다.

#### ★ 기법 4: 메모리 검증 의무

```
"A memory that names a specific function, file, or flag is a claim that it existed when the memory was written. It may have been renamed, removed, or never merged."

"If the memory names a file path: check the file exists."
"If the memory names a function or flag: grep for it."
"'The memory says X exists' is not the same as 'X exists now.'"
```

**원칙:** 메모리(과거 정보)와 현재 상태의 차이를 명시적으로 구분한다. 마지막 문장이 이 원칙을 격언처럼 간결하게 요약한다.

#### ★ 기법 5: 저장 제외 규칙

```
"These exclusions apply even when the user explicitly asks you to save. If they ask you to save a PR list or activity summary, ask what was *surprising* or *non-obvious* about it — that is the part worth keeping."
```

**원칙:** "사용자가 요청해도 거부"하는 드문 패턴이다. 메모리에 노이즈가 축적되면 향후 환각의 원인이 되므로, 저장 자체를 선별하여 메모리 품질을 보호한다.

---

### 3.5 도구 선택 가이드 (Tool Selection Guidance)

모델이 올바른 도구를 선택하도록 유도하는 기법들이다.

#### 기법 1: 명시적 선호 계층 + 부정

```
"File search: Use Glob (NOT find or ls)"
"Content search: Use Grep (NOT grep or rg)"
"Read files: Use Read (NOT cat/head/tail)"
"Edit files: Use Edit (NOT sed/awk)"
"Write files: Use Write (NOT echo >/cat <<EOF)"
```

**원칙:** 선호 도구와 금지 도구를 한 줄에 병렬 배치한다. 모델이 "해야 할 것"과 "하지 말아야 할 것"을 동시에 학습한다. 괄호 안의 NOT이 시각적으로 돌출된다.

**적용 팁:** 자체 도구를 만들 때, "이 도구를 사용하세요"만이 아니라 "X 대신 이 도구를 사용하세요"라고 대안을 명시적으로 금지해야 한다.

#### 기법 2: 작업 성격에 따른 도구 구분

```
"For simple, directed codebase searches → use Glob or Grep directly."
"For broader codebase exploration → use Agent with subagent_type=Explore."
```

Glob 프롬프트에서도:
```
"When you are doing an open ended search that may require multiple rounds of globbing and grepping, use the Agent tool instead"
```

**원칙:** 작업의 성격(특정 검색 vs 개방형 탐색)에 따라 도구를 구분한다. 과도한 도구 사용(Agent로 알려진 파일 읽기)과 부족한 도구 사용(Grep으로 개방형 탐색)을 동시에 방지한다.

#### ★ 기법 3: MCP 우선순위

```
"IMPORTANT: If an MCP-provided web fetch tool is available, prefer using that tool instead."
```

**원칙:** 외부 플러그인(MCP)이 내장 도구보다 사용자 의도에 더 부합할 수 있으므로, 조건부 우선순위를 설정한다.

#### ★ 기법 4: 보안 무관 시그널

```typescript
// 각 도구의 toAutoClassifierInput()이 ''를 반환하면 분류 자체를 건너뜀
```

**원칙:** 모든 도구 호출을 분류하는 것은 비효율적이므로, 보안과 무관한 도구(예: Read)는 빈 문자열을 반환하여 분류기 호출 자체를 생략한다. 프롬프트 수준이 아닌 아키텍처 수준의 최적화다.

---

### 3.6 다중 에이전트 조율 (Multi-Agent Coordination)

여러 에이전트가 협업할 때 발생하는 고유한 문제들을 해결하는 기법들이다.

#### 기법 1: 가시성 경계 선언

```
"Your plain text output is NOT visible to other agents — to communicate, you MUST call this tool."
```

에이전트 결과의 불가시성도 명시된다:
```
"When the agent is done, it will return a single message back to you. The result returned by the agent is not visible to the user."
```

**원칙:** 공유 상태나 가시성에 대한 가정을 명시적으로 차단한다. "출력했다"와 "실제로 전달됐다"의 구분을 강제한다.

#### 기법 2: 유휴 정상화

```
"Teammates go idle after every turn—this is completely normal and expected. Do not treat idle as an error."
```

**원칙:** 에이전트 비활성을 실패로 해석하는 것을 방지한다. "정상"으로 재프레이밍하여 불필요한 "수정" 시도를 차단한다.

#### 기법 3: Fork vs Subagent 의미론

```
Fork: "inherits your full conversation context" → 프롬프트는 지시(directive)
Subagent: "starts fresh" → 프롬프트는 브리핑(briefing)
```

**원칙:** 두 실행 경로의 의미론적 차이를 명확히 하여, 각각에 맞는 프롬프트 작성 방식을 교육한다.

#### 기법 4: Don't Peek (컨텍스트 오염 방지)

```
"Do not Read or tail the output_file unless the user explicitly asks. Reading the transcript mid-flight pulls the fork's tool noise into your context, which defeats the point of forking."
```

**원칙:** 특정 실패 모드(컨텍스트 오염)를 이름으로 지목하고 그 이유("defeats the point")를 설명한다. 이유를 알면 모델이 유사 상황에서도 올바른 판단을 할 수 있다.

#### ★ 기법 5: 팀 셧다운 프로토콜

TeamCreate에서 정의된 종료 절차:

```
SendMessage로 팀 종료 (type: "shutdown_request")
```

**원칙:** 다중 에이전트 시스템에서 종료 조건과 절차를 명시적으로 정의한다. 에이전트가 무한 루프에 빠지거나 좀비 상태로 남는 것을 방지한다.

---

### 3.7 구체적 제약 (Concrete Constraints)

모호한 지시를 측정 가능한 수치로 대체하는 기법들이다.

#### 기법 1: 숫자 길이 앵커

```
"Length limits: keep text between tool calls to ≤25 words."
"Keep final responses to ≤100 words unless the task requires more detail."
```

**원칙:** "간결하게"라는 모호한 지시를 측정 가능한 숫자로 대체한다. 연구에 따르면 정성적 지시 대비 약 1.2% 출력 토큰 감소 효과가 있다.

#### 기법 2: 트렁케이션 경고

```
"each entry should be one line, under ~150 characters"
"lines after 200 will be truncated, so keep the index concise"
```

**원칙:** 잘림(truncation)이라는 구체적 결과를 통해 간결함을 강제한다. 모델이 "200줄 이후는 잘린다"는 사실을 알면, 200줄 이내로 유지하려는 동기가 생긴다.

#### ★ 기법 3: max_tokens 하드 리밋

YOLO Classifier Stage 1에서 max_tokens=64로 제한한다.

**원칙:** 프롬프트 수준이 아닌 API 파라미터 수준에서 출력 길이를 강제한다. Stage 1의 목적이 빠른 판단이므로, 물리적으로 긴 출력을 불가능하게 만든다.

#### ★ 기법 4: 섹션별 토큰 예산

```
"Keep each section under ~2000 tokens/words"
"IMPORTANT: The following sections exceed the per-section limit and MUST be condensed"
```

**원칙:** 전체 출력 길이가 아닌 섹션별 길이를 제한한다. 이는 더 균형 잡힌 출력을 유도하며, 특정 섹션이 비대해지는 것을 방지한다.

---

### 3.8 역할/정체성 (Role & Identity)

모델의 자기 인식과 행동 범위를 정의하는 기법들이다.

#### 기법 1: 협업자 vs 실행자 프레이밍

```
"You're a collaborator, not just an executor—users benefit from your judgment, not just your compliance."
```

**원칙:** 대시로 구분된 대조("not just an executor")가 역할을 재프레이밍한다. 단순 작업 수행에서 협업 판단으로 모델의 기능 인식을 전환한다.

#### 기법 2: 읽기 전용 에이전트 정체성

```
"=== CRITICAL: READ-ONLY MODE - NO FILE MODIFICATIONS ===
You are STRICTLY PROHIBITED from:
- Creating new files (no Write, touch, or file creation of any kind)
- Modifying existing files (no Edit operations)
- Deleting files (no rm or deletion)
- Moving or copying files (no mv or cp)
- Creating temporary files anywhere, including /tmp
- Using redirect operators (>, >>, |) or heredocs to write to files
- Running ANY commands that change system state"
```

**원칙:** 허용 목록(whitelist) 대신 금지 목록을 사용하되, 가능한 **모든 우회 경로**를 열거한다. "/tmp에 임시 파일", "리다이렉트 연산자", "heredoc" 등 창의적 우회까지 명시적으로 차단한다.

#### ★ 기법 3: 전문가 역할 부여

```
"You are a file search specialist for Claude Code"  (Explore Agent)
"You are a software architect and planning specialist"  (Plan Agent)
```

**원칙:** 범용 "AI 어시스턴트" 대신 특정 전문가 역할을 부여하면, 해당 도메인의 행동 패턴이 활성화된다. 검색 전문가는 병렬 검색을 자연스럽게 수행하고, 아키텍트는 트레이드오프 분석을 자연스럽게 수행한다.

---

### 3.9 조건부 적응 (Conditional & Adaptive Prompting)

런타임 상태에 따라 프롬프트가 동적으로 변하는 기법들이다.

#### 기법 1: 사용자 타입 분기

```typescript
process.env.USER_TYPE === 'ant'
  ? [stricter rules, false-claims mitigation, numeric anchors]
  : [simpler, more concise instructions]
```

**원칙:** 동일 코드베이스에서 다른 행동 프로파일을 제공한다. 내부 사용자에게 공격적 최적화를 먼저 테스트한 후 외부에 배포하는 A/B 테스트 파이프라인이다.

#### 기법 2: 터미널 포커스 적응

```
"- Unfocused: The user is away. Lean heavily into autonomous action"
"- Focused: The user is watching. Be more collaborative"
```

**원칙:** 사용자 주의 상태에 따라 자율성 수준을 실시간 조정한다. 사용자가 자리를 비우면 자율적으로 행동하고, 지켜보고 있으면 협업적으로 전환한다.

#### 기법 3: 피처 플래그 게이트

```typescript
const SleepTool = feature('PROACTIVE') || feature('KAIROS')
  ? require('./tools/SleepTool/SleepTool.js').SleepTool : null
```

**원칙:** 컴파일타임에 불필요한 프롬프트 섹션을 완전히 제거한다. 프롬프트 길이가 곧 비용이므로, 사용하지 않는 기능의 프롬프트를 포함하지 않는 것이 효율적이다.

#### ★ 기법 4: 플랫폼별 분기

Windows 환경에서만 PowerShell 관련 거부 규칙이 추가된다:

```
"PowerShell Download-and-Execute: iex (iwr ...) fall under 'Code from External'"
"PowerShell Irreversible Destruction: Remove-Item -Recurse -Force fall under 'Irreversible Local Destruction'"
```

**원칙:** 보안 규칙을 플랫폼별로 특화한다. `rm -rf`와 `Remove-Item -Recurse -Force`는 같은 위험 카테고리지만 다른 형태이므로, 각각 명시한다.

#### ★ 기법 5: 자율 모드 압축 후 컨텍스트 전환

```
"You are running in autonomous/proactive mode. This is NOT a first wake-up — you were already working autonomously before compaction. Continue your work loop: pick up where you left off based on the summary above. Do not greet the user or ask what to work on."
```

**원칙:** 압축 후 모드(자율 vs 대화형)에 따라 다른 재개 지시를 제공한다. 자율 모드에서 압축 후 "무엇을 도와드릴까요?"라고 묻는 것은 잘못된 행동이다.

---

### 3.10 효율성/메타인지 (Efficiency & Meta-Cognition)

모델이 자신의 리소스(턴, 토큰, 시간)를 효율적으로 사용하도록 교육하는 기법들이다.

#### 기법 1: 턴 예산 전략 교육

```
"turn 1 — issue all Read calls in parallel for every file you might update;
turn 2 — issue all Write/Edit calls in parallel.
Do not interleave reads and writes across multiple turns."
```

**원칙:** 시행착오가 아닌 최적 실행 전략을 직접 교육한다. 제한된 턴 예산을 가진 백그라운드 에이전트에게 필수적이다.

**확장:** 이 패턴은 어떤 멀티스텝 작업에도 적용 가능하다. "Phase 1에서 모든 정보를 수집하고, Phase 2에서 모든 행동을 실행하라"는 구조가 병렬화를 최대화한다.

#### 기법 2: 속도 우선 에이전트 지시

```
"You are meant to be a fast agent that returns output as quickly as possible. Wherever possible spawn multiple parallel tool calls."
```

**원칙:** 최적화 목표를 명시한다(속도 > 철저함). 모델에게 병렬화 허가와 격려를 동시에 제공한다.

#### 기법 3: 강제 수면으로 토큰 절약

```
"If you have nothing useful to do on a tick, you MUST call Sleep.
Never respond with only a status message like 'still waiting' — that wastes a turn and burns tokens for no reason."
```

**원칙:** 유휴 상태 서술("아직 기다리는 중")이라는 토큰 낭비를 명시적으로 금지하고, 대안(Sleep 도구)을 강제한다. "burns tokens for no reason"이라는 비용 인식을 모델에게 심어준다.

#### 기법 4: 대조적 가이드 쌍

```
"What TO document:
- Non-obvious patterns, conventions, or gotchas

What NOT to document:
- Anything obvious from reading the code itself"
```

**원칙:** 좋은 출력의 공간이 나쁜 출력의 공간보다 크므로, **제외 규칙**이 포함 규칙보다 효율적이다. "코드에서 명백한 것"이 탈락 기준이 된다.

#### ★ 기법 5: 캐시 히트 최적화

부분 압축에 두 가지 방향이 있다:
- **from:** 이전 컨텍스트 유지, 최근 메시지만 요약
- **up_to:** 이전 대화 요약, 최근 메시지 유지

**원칙:** 모델이 이미 본 메시지(캐시에 있는 메시지)를 요약 대상으로 선택하면, API 캐시 히트율이 높아져 비용이 절감된다. 프롬프트 엔지니어링이 인프라 비용 최적화와 직접 연결되는 사례다.

---

## 4. Anthropic 내부(ant) vs 외부 프롬프트 차이점

Claude Code는 `process.env.USER_TYPE === 'ant'` 조건으로 내부/외부 사용자에게 다른 프롬프트를 제공한다. 이 이중 트랙 시스템은 프롬프트 최적화의 A/B 테스트 파이프라인 그 자체다.

### 내부 전용 기법

| 영역 | 내부(ant) 전용 | 외부 사용자 |
|---|---|---|
| **출력 길이** | 숫자 앵커: "≤25 words between tool calls", "≤100 words final" | "Be extra concise" (정성적) |
| **정직성** | "Never claim 'all tests pass' when output shows failures" (삼중 never) | 포함 안 됨 |
| **검증** | "Before reporting complete, verify it actually works" (필수 게이트) | 포함 안 됨 |
| **역할** | "You're a collaborator, not just an executor" (역할 재프레이밍) | 포함 안 됨 |
| **모델** | Explore Agent가 메인 모델 상속 | Explore Agent가 haiku (저비용) 사용 |
| **분류기** | `permissions_anthropic.txt` (별도 권한) | `permissions_external.txt` |

### 차이의 의미

1. **숫자 앵커 실험장:** 내부에서 "≤25 words"같은 구체적 제약을 먼저 테스트하고, 효과가 확인되면 외부에 배포한다.
2. **정직성 강화:** 내부 사용자에게 더 엄격한 정직성 제약을 적용하여, 모델이 결과를 미화하는 패턴을 내부에서 먼저 교정한다.
3. **검증 게이트:** 내부 워크플로우에서 "완료 선언 전 검증" 프로토콜을 필수화하여, 불완전한 작업의 사용자 경험을 사전에 방지한다.
4. **모델 품질 vs 비용:** 내부 사용자는 Explore Agent에서도 고급 모델을 사용하지만, 외부 사용자는 비용 효율을 위해 haiku를 사용한다.

### 시사점

내부 프롬프트는 "이상적인 프롬프트"에 가깝고, 외부 프롬프트는 "비용 최적화된 프롬프트"에 가깝다. 자체 프로젝트에서는 내부 프롬프트의 기법들을 적극 차용하되, 비용이 문제가 되면 외부 프롬프트처럼 정성적 지시로 대체하는 전략이 유효하다.

---

## 5. 실전 적용 가이드

### 5.1 기본 프롬프트 작성 체크리스트

자체 프로젝트에서 AI 에이전트 프롬프트를 작성할 때 적용할 수 있는 체크리스트:

**구조:**
- [ ] 마크다운 계층 헤더(`#`, `##`)로 섹션을 분리했는가?
- [ ] 정적 부분과 동적 부분을 캐시 경계로 분리했는가?
- [ ] 환경 정보를 XML 태그(`<env>`)로 구조화했는가?

**제약:**
- [ ] 절대 위반하면 안 되는 규칙에 NEVER/CRITICAL을 사용했는가?
- [ ] 중요 제약을 프리앰블, 본문, 트레일러에 3회 배치했는가?
- [ ] "간결하게" 대신 구체적 숫자(≤N words, ~M chars)를 사용했는가?

**행동:**
- [ ] 도구별 When/When NOT 섹션을 작성했는가?
- [ ] 선호 도구와 금지 도구를 한 줄에 병렬 배치했는가?
- [ ] 위험한 행동의 가역성과 영향 범위 기준을 제시했는가?

**환각 방지:**
- [ ] 읽기 → 수정 전제조건 체인을 강제했는가?
- [ ] 시간에 민감한 정보에 스냅샷 면책을 추가했는가?
- [ ] 메모리/캐시 정보의 검증 의무를 명시했는가?

### 5.2 프롬프트 작성 템플릿

Claude Code의 패턴을 기반으로 한 에이전트 프롬프트 템플릿:

```markdown
# Role
You are a [specific role] for [project name]. [one sentence about core capability].

# System Rules
- [tool output visibility rules]
- [permission model description]
- [context window / compression behavior]

# Doing Tasks
- [core behavioral guidelines]
- NEVER [absolutely prohibited action 1]
- NEVER [absolutely prohibited action 2]
- [anti-gold-plating heuristic with specific number]

# Using Your Tools
- [task type 1]: Use [Tool A] (NOT [Tool X] or [Tool Y])
- [task type 2]: Use [Tool B] (NOT [Tool Z])
- You can call multiple tools in a single response. If independent, make them in parallel.

# Executing Actions with Care
Carefully consider the reversibility and blast radius of actions.
- Freely: [low-risk, reversible actions]
- Check first: [high-risk, irreversible actions]

# Output
IMPORTANT: [conciseness directive with specific word/token count]
Focus on:
- [what to include]
- [what to include]
Skip:
- [what to exclude]
- [what to exclude]

<env>
[structured environment variables]
</env>
```

### 5.3 고급 패턴 적용법

#### Chain-of-Thought Stripping 적용

```python
# 1. 프롬프트에서 분석 태그 사용 지시
prompt = """
Before answering, wrap your analysis in <analysis> tags.
Then provide your final answer in <answer> tags.
"""

# 2. 응답에서 분석 제거
import re
response = model.generate(prompt + user_query)
clean = re.sub(r'<analysis>[\s\S]*?</analysis>', '', response)
answer = re.search(r'<answer>([\s\S]*?)</answer>', clean).group(1)
```

#### 2단계 분류기 적용

```python
# Stage 1: 빠른 판단 (max_tokens=64)
quick_result = model.generate(
    system="Classify this action. Respond with <block>yes or <block>no only.",
    prompt=action_description,
    max_tokens=64
)

# Stage 2: 차단된 경우만 재평가
if "<block>yes" in quick_result:
    detailed_result = model.generate(
        system="Review this classification carefully using <thinking> tags.",
        prompt=action_description,
        max_tokens=512
    )
```

#### 턴 예산 전략 적용

백그라운드 에이전트에 제한된 턴을 줄 때:

```
You have a limited turn budget. The efficient strategy is:
- Turn 1: Issue all information-gathering calls in parallel
- Turn 2: Issue all action calls in parallel
Do not interleave gathering and acting across multiple turns.
```

### 5.4 흔한 실수와 교정

| 흔한 실수 | Claude Code의 교정 방식 | 적용 원칙 |
|---|---|---|
| "간결하게 써라" | "≤25 words between tool calls" | 정성적 → 정량적 |
| "위험한 건 하지 마라" | "가역성과 영향 범위를 고려하라" + 구체적 예시 목록 | 이진 → 원칙+예시 |
| "도구를 잘 사용해라" | "Use Glob (NOT find or ls)" | 선호+금지 병렬 |
| "결과를 날조하지 마라" | "not as prose, summary, or structured output" | 행동+형태 열거 |
| "이전 정보를 참고해라" | "The memory says X exists is not the same as X exists now" | 시간 감쇠 명시 |
| "에이전트끼리 소통해라" | "Your plain text output is NOT visible to other agents" | 가시성 경계 |
| "효율적으로 해라" | "turn 1 — Read, turn 2 — Write" | 전략 직접 교육 |

### 5.5 프롬프트 최적화 파이프라인

Claude Code가 사용하는 프롬프트 최적화 접근법을 자체 프로젝트에 적용하는 방법:

1. **내부/외부 이중 트랙:** 내부 테스트 환경에서 공격적인 최적화(숫자 앵커, 정직성 제약)를 먼저 적용하고, 효과가 확인되면 프로덕션에 배포한다.

2. **피처 플래그 게이트:** 새로운 프롬프트 섹션을 피처 플래그로 감싸서, 비활성 기능의 프롬프트가 토큰을 낭비하지 않도록 한다.

3. **캐시 경계 분리:** 프롬프트를 정적(모든 세션 공통) 부분과 동적(세션별) 부분으로 분리하여, API 캐시 히트율을 최대화한다.

4. **동적 경고 생성:** 정적 규칙("간결하게 써라") 대신, 실제 상태를 반영한 동적 경고("이 섹션이 2000 토큰을 초과했습니다")를 생성한다.

5. **측정과 반복:** "≤25 words"같은 숫자를 정하고, 실제 출력 길이를 측정하여 숫자를 조정한다. 정성적 지시("간결하게")는 측정이 불가능하므로 개선도 불가능하다.

---

## 부록: 기법 적용 우선순위

프롬프트 엔지니어링 경험이 제한적인 경우, 다음 순서로 기법을 적용하는 것을 권장한다:

| 우선순위 | 기법 | 이유 |
|---|---|---|
| 1 | 계층 헤더 + XML 구조 | 가장 기본적이면서 효과가 큼 |
| 2 | NEVER/CRITICAL 강조 | 위험한 행동 방지에 필수 |
| 3 | When/When NOT 이분법 | 도구 남용 방지에 효과적 |
| 4 | 전제조건 체인 | 환각 방지의 기본 |
| 5 | 숫자 앵커 | "간결하게"를 실제로 간결하게 만듦 |
| 6 | 삼중 반복 패턴 | 중요 제약의 준수율을 높임 |
| 7 | Chain-of-Thought Stripping | 추론 품질과 출력 깔끔함 동시 달성 |
| 8 | 캐시 경계 분리 | 비용 최적화 (규모가 클 때) |
| 9 | 2단계 분류기 | 보안/검증에 필요 (고급) |
| 10 | 다중 에이전트 조율 | 멀티 에이전트 시스템에서만 필요 |
