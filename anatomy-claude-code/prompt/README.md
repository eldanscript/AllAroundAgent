# Claude Code Prompt 분석

> Claude Code 소스 코드에서 추출한 프롬프트를 카테고리별로 구조화한 문서 모음.
>
> 소스 코드 위치: `anatomy-claude-code/refs/hangsman-claude-code-source/`

---

## 파일 목록

| 파일 | 용도 | 주요 프롬프트 수 |
|------|------|-----------------|
| [`system-prompt.md`](./system-prompt.md) | 메인 시스템 프롬프트 조립 구조 | 14개 섹션 |
| [`tool-prompts.md`](./tool-prompts.md) | 도구별 프롬프트 모음 | 30개 도구 |
| [`agent-prompts.md`](./agent-prompts.md) | 에이전트 프롬프트 | 6개 에이전트 |
| [`compact-prompts.md`](./compact-prompts.md) | 압축/요약 관련 프롬프트 | 6개 프롬프트 |
| [`memory-prompts.md`](./memory-prompts.md) | 메모리/컨텍스트 프롬프트 | 5개 시스템 |
| [`security-prompts.md`](./security-prompts.md) | 보안/권한 관련 프롬프트 | 6개 프롬프트 |

---

## 각 파일 상세 설명

### 1. system-prompt.md (시스템 프롬프트)

메인 시스템 프롬프트의 전체 조립 구조. `getSystemPrompt()` 함수에서 반환하는 문자열 배열의 각 섹션을 분석.

- **정적 영역:** 인트로, 시스템 규칙, 작업 수행, 행동 신중성, 도구 사용, 톤/스타일, 출력 효율성
- **동적 영역:** 세션별 가이던스, 메모리, 환경 정보, 언어/출력 스타일, MCP 지침, 스크래치패드 등
- 주요 소스: `src/constants/prompts.ts`, `src/constants/cyberRiskInstruction.ts`

### 2. tool-prompts.md (도구별 프롬프트)

30개 도구 각각의 프롬프트 원문과 설명:

- **핵심 도구:** BashTool, FileReadTool, FileEditTool, FileWriteTool, GlobTool, GrepTool
- **에이전트/통신:** AgentTool, SendMessageTool, SkillTool, AskUserQuestionTool
- **계획:** EnterPlanModeTool, ExitPlanModeTool
- **작업 관리:** TaskCreateTool, TodoWriteTool, TaskGetTool, TaskListTool, TaskUpdateTool, TaskStopTool
- **웹:** WebFetchTool, WebSearchTool
- **기타:** NotebookEditTool, MCPTool, ToolSearchTool, LSPTool, BriefTool, ConfigTool, SleepTool, PowerShellTool
- **인프라:** EnterWorktreeTool, ExitWorktreeTool, TeamCreateTool, TeamDeleteTool, RemoteTriggerTool, ScheduleCronTool
- **MCP 리소스:** ListMcpResourcesTool, ReadMcpResourceTool

### 3. agent-prompts.md (에이전트 프롬프트)

서브에이전트로 실행되는 에이전트별 시스템 프롬프트:

- **Explore Agent:** 읽기 전용 코드베이스 탐색 전문가
- **Plan Agent:** 읽기 전용 아키텍처/구현 계획 전문가
- **Default Agent:** 범용 서브에이전트 기본 프롬프트
- **Companion (Buddy):** 사용자 인터페이스의 캐릭터 컴패니언
- **Verification Agent:** 독립 검증 에이전트 프로토콜
- **Team Workflow:** 다중 에이전트 팀 조율

### 4. compact-prompts.md (압축/요약 프롬프트)

컨텍스트 윈도우 한계 관리:

- **Base Compact:** 전체 대화를 9개 섹션으로 요약
- **Partial Compact (from):** 최근 메시지만 요약, 이전 컨텍스트 유지
- **Partial Compact (up_to):** 이전 대화를 요약, 후속 메시지를 위한 컨텍스트 보존
- **No-Tools Preamble/Trailer:** 도구 호출 금지 지시 (도구 호출 시 턴 낭비 방지)
- **Compact User Summary:** 압축 후 세션 계속 안내 메시지
- **Summary Formatting:** analysis 블록 제거, summary 태그 변환

### 5. memory-prompts.md (메모리/컨텍스트 프롬프트)

세션 간 지속되는 메모리 시스템:

- **Memdir (메인 메모리):** 4가지 타입(user, feedback, project, reference)의 영구 파일 기반 메모리. MEMORY.md 인덱스
- **Extract Memories:** 백그라운드 자동 메모리 추출 에이전트
- **Session Memory:** 세션 내 구조화된 메모. 9개 섹션 템플릿
- **Magic Docs:** 자동 업데이트 문서 시스템
- **MEMORY.md 관리:** 200줄/25KB 제한, 엔트리포인트 관리

### 6. security-prompts.md (보안/권한 프롬프트)

보안 분류 및 권한 관리:

- **YOLO Classifier:** 자동 모드 보안 분류기 (2단계 XML, tool_use, 사용자 규칙 커스터마이제이션)
- **Cyber Risk Instruction:** 보안 요청 처리 경계 (Safeguards 팀 소유)
- **Actions Section:** 되돌리기 어려운 행동에 대한 사전 확인 프로토콜
- **Sandbox Policy:** 파일시스템/네트워크 접근 제한 (BashTool/PowerShellTool)
- **Git Safety Protocol:** Git 명령 안전 규칙
- **Undercover Mode:** 미발표 모델 사용 시 정보 은닉

---

## 프롬프트 소스 파일 맵

```
src/
├── constants/
│   ├── prompts.ts              # 메인 시스템 프롬프트 조립
│   └── cyberRiskInstruction.ts # 사이버 리스크 지침
├── buddy/
│   └── prompt.ts               # Companion 프롬프트
├── memdir/
│   ├── memdir.ts               # 메모리 디렉토리 프롬프트
│   └── memoryTypes.ts          # 메모리 타입 분류 체계
├── services/
│   ├── compact/prompt.ts       # 압축/요약 프롬프트
│   ├── SessionMemory/prompts.ts # 세션 메모리 프롬프트
│   ├── extractMemories/prompts.ts # 자동 메모리 추출
│   └── MagicDocs/prompts.ts    # Magic Docs 프롬프트
├── tools/
│   ├── AgentTool/
│   │   ├── prompt.ts           # AgentTool 프롬프트
│   │   └── built-in/
│   │       ├── exploreAgent.ts # Explore 에이전트
│   │       └── planAgent.ts    # Plan 에이전트
│   ├── BashTool/prompt.ts      # Bash 도구 (+ Git/Sandbox)
│   ├── FileReadTool/prompt.ts  # Read 도구
│   ├── FileEditTool/prompt.ts  # Edit 도구
│   ├── FileWriteTool/prompt.ts # Write 도구
│   ├── GlobTool/prompt.ts      # Glob 도구
│   ├── GrepTool/prompt.ts      # Grep 도구
│   ├── WebFetchTool/prompt.ts  # WebFetch 도구
│   ├── WebSearchTool/prompt.ts # WebSearch 도구
│   ├── SkillTool/prompt.ts     # Skill 도구
│   ├── AskUserQuestionTool/prompt.ts # AskUserQuestion 도구
│   ├── EnterPlanModeTool/prompt.ts   # EnterPlanMode 도구
│   ├── ExitPlanModeTool/prompt.ts    # ExitPlanMode 도구
│   ├── TaskCreateTool/prompt.ts      # TaskCreate 도구
│   ├── TodoWriteTool/prompt.ts       # TodoWrite 도구
│   ├── NotebookEditTool/prompt.ts    # NotebookEdit 도구
│   ├── ToolSearchTool/prompt.ts      # ToolSearch 도구
│   ├── SendMessageTool/prompt.ts     # SendMessage 도구
│   ├── SleepTool/prompt.ts           # Sleep 도구
│   ├── BriefTool/prompt.ts           # Brief/SendUserMessage 도구
│   ├── LSPTool/prompt.ts             # LSP 도구
│   ├── ConfigTool/prompt.ts          # Config 도구
│   ├── PowerShellTool/prompt.ts      # PowerShell 도구
│   ├── EnterWorktreeTool/prompt.ts   # EnterWorktree 도구
│   ├── ExitWorktreeTool/prompt.ts    # ExitWorktree 도구
│   ├── TeamCreateTool/prompt.ts      # TeamCreate 도구
│   ├── TeamDeleteTool/prompt.ts      # TeamDelete 도구
│   ├── RemoteTriggerTool/prompt.ts   # RemoteTrigger 도구
│   ├── ScheduleCronTool/prompt.ts    # ScheduleCron 도구
│   ├── ListMcpResourcesTool/prompt.ts # ListMcpResources 도구
│   ├── ReadMcpResourceTool/prompt.ts  # ReadMcpResource 도구
│   └── MCPTool/prompt.ts             # MCP 도구 (런타임 오버라이드)
└── utils/
    └── permissions/
        └── yoloClassifier.ts         # 자동 모드 보안 분류기
```
