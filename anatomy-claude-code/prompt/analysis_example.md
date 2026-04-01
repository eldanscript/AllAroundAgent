> 클로드 코드 소스라고 유추되는 유출 코드가 돌고 있는데, 다들 그렇듯이 나도 뭔가 쓸만한 게 있을까 뒤져보고 있다. 내 관심은 프롬프트라서 클로드 코드에게 이 소스에 등장하는 프롬프트로부터 프롬프트 작성 팁을 찾아달라고 해봤다.
> 

**분류 체계**

총83개의 기법을 10개의 카테고리로 분류.

---

1. Emphasis & Attention Control (강조 및 주의 제어)

1.1 CAPS for Behavioral Locks

파일: BashTool/prompt.ts — "NEVER" 12회 이상 사용
"NEVER update the git config"
"NEVER run destructive git commands"
"NEVER skip hooks (--no-verify)"
왜 효과적인가: 대문자는 시각적/의미적 가중치를 모두 높임. 모델은 대문자 금지사항을 일반 텍스트보다 우선순위 높은 제약으로 처리함.

1.2 IMPORTANT/CRITICAL Prefix

파일: compact/prompt.ts
"CRITICAL: Respond with TEXT ONLY. Do NOT call any tools."
왜 효과적인가: "CRITICAL"은 프롬프트 시작 부분에서 사용되어 첫 읽기 주의를 잡음. 실패 결과("you will fail the task")와 결합하면 제약이 강화됨.

1.3 Triple-Redundancy Pattern (삼중 반복)

파일: compact/prompt.ts — 도구 금지가 3곳에 등장

1. Preamble: "CRITICAL: Respond with TEXT ONLY"
2. Body: 지시사항 내
3. Trailer: "REMINDER: Do NOT call any tools"
왜 효과적인가: Recency bias(최근성 편향)를 활용. 트레일러는 컨텍스트에서 가장 최근이고, 프리앰블은 첫 읽기 주의를 담당. 세 번 반복은 단일 강한
문장보다 효과적.

---

1. Structural Patterns (구조적 패턴)

2.1 Hierarchical Section Headers

파일: prompts.ts # System # Doing tasks # Using your tools # Executing actions with care # Tone and style # Output efficiency

왜 효과적인가: 명확한 섹션 헤더가 인지적 버킷을 만들어, 모델이 서로 다른 지시 영역을 독립적으로 참조하고 우선순위를 정할 수 있음.

2.2 XML Tags as Scratchpad → Strip Pattern

파일: compact/prompt.ts
"wrap your analysis in <analysis> tags"
// 이후 코드에서:
formattedSummary.replace(/<analysis>[\s\S]*?<\/analysis>/, '')
왜 효과적인가: <analysis> 태그는 이중 목적으로 사용됨 — 생성 시 추론 품질을 높이지만, 최종 출력에서는 제거됨. 강제 추론이 품질을 개선하되, 추론
자체는 사용자에게 노출되지 않는 기법.

★ Insight ─────────────────────────────────────
이것은 "chain-of-thought stripping" 패턴으로, Anthropic이 자사 제품에서 사용하는 핵심 기법입니다. 모델에게 XML 태그 안에서 사고하도록 강제한 뒤
post-processing으로 제거하면, 추론 품질은 유지하면서 출력은 깔끔하게 됩니다.
─────────────────────────────────────────────────

2.3 Cache Boundary Marker

파일: prompts.ts
export const SYSTEM_PROMPT_DYNAMIC_BOUNDARY = '**SYSTEM_PROMPT_DYNAMIC_BOUNDARY**'
왜 효과적인가: 메타-프롬프팅 — 프롬프트 자체가 자신의 처리 방법을 포함함. 정적(전역 캐시 가능) 콘텐츠와 동적(세션별) 콘텐츠를 분리하여 API 비용
최적화.

2.4 Structured Environment Context

파일: prompts.ts
<env>
Working directory: ${getCwd()}
Platform: ${env.platform}
Shell: ${shellName}
</env>
왜 효과적인가: XML 태그가 구조화된 데이터임을 신호함. 모델이 환경 정보를 쿼리 가능한 사실로 처리.

---

1. Behavioral Control (행동 제어)

3.1 When/When NOT Bifurcation

파일: EnterPlanModeTool/prompt.ts
"### GOOD - Use EnterPlanMode:" (7 예시)
"### BAD - Don't use EnterPlanMode:" (4 예시)
왜 효과적인가: 긍정/부정 사례를 명시적으로 분리하여, 모델이 도구를 범용 대안으로 사용하는 것을 방지.

3.2 Anti-Gold-Plating with Concrete Heuristic

파일: prompts.ts
"Three similar lines of code is better than a premature abstraction."
왜 효과적인가: 모호한 "과도하게 하지 마라" 대신 구체적 비율을 제공. 3줄 vs 추상화라는 의사결정 기준이 됨.

3.3 Mandatory Verification Gate

파일: prompts.ts (ant-only)
"Before reporting a task complete, verify it actually works: run the test,
execute the script, check the output. If you can't verify, say so explicitly
rather than claiming success."
왜 효과적인가: 완료 선언 전 명시적 게이트를 만듦. "say so explicitly"는 검증 불가 시에도 침묵이 아닌 명시적 보고를 강제.

3.4 Honest Reporting Mandate

파일: prompts.ts (ant-only)
"Never claim 'all tests pass' when output shows failures, never suppress or
simplify failing checks to manufacture a green result, and never characterize
incomplete work as done."
왜 효과적인가: 삼중 부정("never...never...never")으로 결과 미화를 원천 차단. 가짜 성공 보고라는 특정 실패 모드를 이름으로 지목.

3.5 Risk-Based Authorization Framework

파일: prompts.ts
"Carefully consider the reversibility and blast radius of actions."
왜 효과적인가: 단순 이진 규칙 대신 원칙적 프레임워크(가역성 + 영향 범위)를 제공하여, 새로운 상황에서도 모델이 적용 가능.

---

1. Anti-Hallucination (환각 방지)

4.1 Precondition Chain

파일: FileEditTool/prompt.ts
"You must use your Read tool at least once before editing.
This tool will error if you attempt an edit without reading the file."
왜 효과적인가: "will error"라는 보장된 실패를 선언함으로써, 모델이 지름길을 시도하는 것을 방지.

4.2 Fabrication Ban with Form Listing

파일: AgentTool/prompt.ts (fork mode)
"Never fabricate or predict fork results in any format — not as prose,
summary, or structured output."
왜 효과적인가: 나쁜 행동뿐 아니라 그것이 나타날 형태(산문, 요약, 구조화된 출력)까지 명시. 대안("give status, not a guess")도 제공.

4.3 Snapshot Semantics Disclaimer

파일: context.ts
"This is the git status at the start of the conversation. Note that this
status is a snapshot in time, and will not update during the conversation."
왜 효과적인가: 시간에 민감한 정보가 오래될 수 있음을 명시하여, 모델이 현실과 달라진 상태에 의존하는 것을 방지.

4.4 Knowledge Cutoff Declaration

파일: prompts.ts
"Assistant knowledge cutoff is ${cutoff}."
왜 효과적인가: 모델별 다른 cutoff 날짜(Sonnet: 2025/8, Opus: 2025/5)를 설정하여 훈련 데이터 이후 사건에 대한 환각 방지.

---

1. Tool Selection Guidance (도구 선택 가이드)

5.1 Explicit Preference Hierarchy with Negation

파일: BashTool/prompt.ts
"File search: Use Glob (NOT find or ls)
Content search: Use Grep (NOT grep or rg)
Read files: Use Read (NOT cat/head/tail)
Edit files: Use Edit (NOT sed/awk)"
왜 효과적인가: 선호 도구와 금지 도구를 동시에 나열. 모델은 "해야 할 것"과 "하지 말아야 할 것"을 한 줄에서 학습.

5.2 Search vs Lookup Distinction

파일: AgentTool/prompt.ts
"For simple, directed codebase searches → use Glob or Grep directly.
For broader codebase exploration → use Agent with subagent_type=Explore."
왜 효과적인가: 작업의 성격(특정 검색 vs 탐색)에 따라 도구를 구분, 과도한 도구 사용(Agent로 알려진 파일 읽기)과 부족한 도구 사용(Grep으로 열린
탐색)을 모두 방지.

5.3 Capability-Based Agent Assignment

파일: TeamCreateTool/prompt.ts
"Read-only agents (e.g., Explore, Plan) cannot edit or write files.
Only assign them research, search, or planning tasks."
왜 효과적인가: 에이전트 능력과 작업 요구사항을 사전에 매칭하여, 능력이 없는 에이전트에 작업 할당을 방지.

---

1. Multi-Agent Coordination (다중 에이전트 조율)

6.1 Visibility Boundary Declaration

파일: SendMessageTool/prompt.ts
"Your plain text output is NOT visible to other agents — to communicate,
you MUST call this tool."
왜 효과적인가: 공유 상태나 가시성에 대한 가정을 명시적으로 차단. "말했다"와 "실제로 전달됐다"의 구분을 강제.

6.2 Idleness Normalization

파일: TeamCreateTool/prompt.ts
"Teammates go idle after every turn—this is completely normal and expected.
Do not treat idle as an error."
왜 효과적인가: 에이전트 비활성을 실패로 해석하는 것을 방지. "정상"으로 재프레이밍하여 불필요한 "수정" 시도 차단.

6.3 Fork vs Subagent Semantics

파일: AgentTool/prompt.ts
Fork: "inherits your full conversation context" → 프롬프트는 지시(directive)
Subagent: "starts fresh" → 프롬프트는 브리핑(briefing)
왜 효과적인가: 두 경로의 의미론을 명확히 하여, 각각에 맞는 프롬프트 작성 방식을 교육.

6.4 Don't Peek (Context Pollution Prevention)

파일: AgentTool/prompt.ts
"Do not Read or tail the output_file unless the user explicitly asks.
Reading the transcript mid-flight pulls the fork's tool noise into your
context, which defeats the point of forking."
왜 효과적인가: 특정 실패 모드(컨텍스트 오염)를 이름으로 지목하고 그 이유를 설명. "defeats the point"라는 표현이 행동의 결과를 명확히 함.

---

1. Concrete Constraints (구체적 제약)

7.1 Numeric Length Anchors

파일: prompts.ts (ant-only)
"Length limits: keep text between tool calls to ≤25 words.
Keep final responses to ≤100 words unless the task requires more detail."
왜 효과적인가: 모호한 "간결하게"를 측정 가능한 숫자로 대체. 연구에 따르면 정성적 지시 대비 ~1.2% 출력 토큰 감소.

7.2 Token Budget as Task Variable

파일: prompts.ts
"The target is a hard minimum, not a suggestion. If you stop early,
the system will automatically continue you."
왜 효과적인가: 토큰 예산을 효율성 제약이 아닌 채워야 할 목표로 재프레이밍. "하드 미니멈"이라는 표현이 무시를 방지.

7.3 Character Limits for Index Files

파일: memdir.ts
"each entry should be one line, under ~150 characters"
"lines after 200 will be truncated, so keep the index concise"
왜 효과적인가: 잘림(truncation)이라는 구체적 결과를 통해 간결함을 강제. 측정 가능한 제약이 모호한 "be concise"보다 강력.

---

1. Role & Identity (역할 및 정체성)

8.1 Collaborator vs Executor Framing

파일: prompts.ts (ant-only)
"You're a collaborator, not just an executor—users benefit from your
judgment, not just your compliance."
왜 효과적인가: 대시로 구분된 대조("not just an executor")가 역할을 재프레이밍. 모델의 기능 인식을 단순 작업 수행에서 협업 판단으로 전환.

8.2 Read-Only Agent Identity

파일: exploreAgent.ts
"=== CRITICAL: READ-ONLY MODE - NO FILE MODIFICATIONS ===
You are STRICTLY PROHIBITED from:

- Creating new files
- Modifying existing files
- Deleting files..."
왜 효과적인가: 허용 목록(whitelist) 대신 금지 목록 열거를 사용. 창의적 우회("/tmp에 임시 파일" 등)까지 명시적으로 차단.

8.3 Companion Non-Interference

파일: buddy/prompt.ts
"Your job is to stay out of the way: respond in ONE line or less.
Don't explain that you're not ${name} — they know."
왜 효과적인가: 공유 UI에서 역할 분리를 교육. "they know"라는 가정이 불필요한 메타 설명을 방지.

---

1. Conditional & Adaptive Prompting (조건부 적응 프롬프팅)

9.1 User-Type Branching

파일: prompts.ts, EnterPlanModeTool/prompt.ts
process.env.USER_TYPE === 'ant'
? [stricter rules, false-claims mitigation, numeric anchors]
: [simpler, more concise instructions]
왜 효과적인가: 동일 코드베이스에서 다른 행동 프로파일을 제공. 내부 사용자에게 공격적 최적화를 먼저 테스트한 후 외부에 배포하는 A/B 테스트
파이프라인.

9.2 Terminal Focus Adaptation

파일: prompts.ts (proactive mode)
"- **Unfocused**: The user is away. Lean heavily into autonomous action

- **Focused**: The user is watching. Be more collaborative"
왜 효과적인가: 사용자 주의 상태에 따라 자율성 수준을 실시간 조정하는 피드백 루프.

9.3 Feature-Flag Gated Sections

파일: prompts.ts
const SleepTool = feature('PROACTIVE') || feature('KAIROS')
? require('./tools/SleepTool/SleepTool.js').SleepTool : null
왜 효과적인가: Bun의 dead code elimination을 활용한 컴파일타임 프롬프트 최적화. 불필요한 섹션이 빌드에서 완전히 제거됨.

9.4 Dynamic Warning Generation

파일: SessionMemory/prompts.ts
if (oversizedSections.length > 0) {
parts.push(`IMPORTANT: The following sections exceed the per-section limit     and MUST be condensed:\\n${oversizedSections.join('\\n')}`)
}
왜 효과적인가: 정적 지시가 아닌, 실제 상태 기반의 동적 경고를 생성. "이전 출력이 너무 깁니다"라는 구체적 피드백이 추상적 규칙보다 강력.

---

1. Efficiency & Meta-Cognition (효율성 및 메타인지)

10.1 Turn Budget Strategy Teaching

파일: extractMemories/prompts.ts
"turn 1 — issue all Read calls in parallel for every file you might update;
turn 2 — issue all Write/Edit calls in parallel.
Do not interleave reads and writes across multiple turns."
왜 효과적인가: 시행착오가 아닌 최적 실행 전략을 직접 교육. 제한된 턴 예산을 가진 백그라운드 에이전트에 필수적.

10.2 Speed-First Agent Instruction

파일: exploreAgent.ts
"You are meant to be a fast agent that returns output as quickly as possible.
Wherever possible spawn multiple parallel tool calls."
왜 효과적인가: 최적화 목표를 명시(속도 > 철저함). 모델에게 병렬화 허가와 격려를 동시에 제공.

10.3 Forced Sleep for Token Conservation

파일: prompts.ts (proactive mode)
"If you have nothing useful to do on a tick, you MUST call Sleep.
Never respond with only a status message like 'still waiting' —
that wastes a turn and burns tokens for no reason."
왜 효과적인가: 유휴 상태 서술("아직 기다리는 중")이라는 토큰 낭비를 명시적으로 금지하고 대안(Sleep 도구)을 강제.

10.4 Contrastive Guidance Pairs (대조적 가이드 쌍)

파일: MagicDocs/prompts.ts
"What TO document:

- Non-obvious patterns, conventions, or gotchas

What NOT to document:

- Anything obvious from reading the code itself"
왜 효과적인가: 좋은 출력의 공간이 나쁜 출력의 공간보다 크므로, 제외 규칙이 포함 규칙보다 효율적. "코드에서 명백한 것"이 탈락 기준이 됨.

10.5 Safe Variable Injection

파일: MagicDocs/prompts.ts
// Single-pass replacement avoids: (1) $ backreference corruption
// (2) double-substitution when user content contains {{varName}}
template.replace(/\{\{(\w+)\}\}/g, (match, key) => variables[key] ?? match)
왜 효과적인가: 사용자 콘텐츠에 변수명이 포함된 경우의 프롬프트 인젝션 방지. 단일 패스로 이중 치환과 $ 참조 오류를 동시에 방지.

---

★ Insight ─────────────────────────────────────
가장 핵심적인 3가지 발견:

1. Chain-of-thought stripping (<analysis> → strip) — 추론 강제 후 제거로 품질과 깔끔함을 동시 달성
2. 삼중 반복 패턴 — 중요한 제약을 프리앰블, 본문, 트레일러에 3회 배치하여 recency bias 활용
3. 구체적 숫자 (≤25 words, ~150 chars, 200 line limit) — 정성적 "간결하게"보다 측정 가능한 제약이 ~1.2% 더 효과적

이 기법들은 Anthropic이 자체 제품에서 프롬프트 엔지니어링을 어떻게 하는지 보여주는 일급 자료입니다. 특히 "내부(ant) vs 외부" 이중 트랙은
프롬프트 최적화의 A/B 테스트 파이프라인 그 자체입니다.
─────────────────────────────────────────────────
