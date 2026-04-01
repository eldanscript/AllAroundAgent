# Phase 2: alanisme-claude-code-decompiled 리포지토리 심층 분석

> 분석 대상: `refs/alanisme-claude-code-decompiled` (Claude Code v2.1.88 기반 20개 심층 리포트)
> 작성일: 2026-03-31

---

## 1. 리포지토리 개요

이 리포지토리는 Anthropic의 Claude Code v2.1.88 npm 패키지를 디컴파일/리버스 엔지니어링하여 작성한 **문서 전용(documentation-only)** 연구 아카이브이다. 코드를 포함하지 않으며, 20개의 독립 분석 리포트로 구성되어 있다.

**분석된 코드베이스 규모:**

| 항목 | 수치 |
|------|------|
| TypeScript 소스 파일 | 1,884개 |
| 총 코드 라인 | 512,664줄 |
| 최대 단일 파일 | `query.ts` -- 785KB |
| 내장 도구 | 40개 이상 |
| 슬래시 명령 | 80개 이상 |
| npm 의존성 | 192개 패키지 |
| 피처 게이트 모듈 | 108개 |

리포트는 크게 두 카테고리로 나뉜다:
- **핵심 아키텍처 분석** (06~20): 에이전트 루프, 도구 시스템, 권한 모델, 시스템 프롬프트, MCP 통합, 컨텍스트 관리, 상태 관리, 브릿지 시스템, 터미널 UI, 스트리밍 전송, 슬래시 명령 등
- **탐사 및 조사 리포트** (01~05): 텔레메트리, 숨겨진 기능, 언더커버 모드, 원격 제어, 미래 로드맵

---

## 2. 20개 분석 리포트 핵심 내용 요약

### 리포트 01: 텔레메트리와 프라이버시

Claude Code는 이중 분석 파이프라인을 운영한다.

**1차 로깅 (Anthropic 자체):**
- 엔드포인트: `https://api.anthropic.com/api/event_logging/batch`
- OpenTelemetry + Protocol Buffers 사용
- 배치 크기 200개, 10초마다 플러시
- 실패 이벤트는 `~/.claude/telemetry/`에 디스크 저장 후 재시도

**3차 로깅 (Datadog):**
- 64개 사전 승인 이벤트 유형만 전송
- 엔드포인트: `https://http-intake.logs.us5.datadoghq.com/api/v2/logs`

**수집 데이터 범위:**
> "platform, platformRaw, arch, nodeVersion, terminal type, installed package managers and runtimes, CI/CD detection, GitHub Actions metadata, WSL version, Linux distro, kernel version" (metadata.ts:417-452)

- 리포지토리 원격 URL은 SHA256 해시의 처음 16자로 핑거프린팅
- `OTEL_LOG_TOOL_DETAILS=1` 설정 시 도구 입력이 잘림 없이 전체 로깅
- Bash 명령에서 파일 확장자 추출 및 로깅
- **직접 API 사용자는 1차 텔레메트리 비활성화 불가**

### 리포트 02: 숨겨진 기능과 코드네임

**동물 코드네임 체계:**

| 코드네임 | 정체 |
|---------|------|
| **Tengu** (천구) | 제품/텔레메트리 접두사, 250개 이상의 분석 이벤트에 사용 |
| **Capybara** (카피바라) | Sonnet 계열 모델, 현재 v8 |
| **Fennec** (페넥여우) | Opus 4.6의 전신 |
| **Numbat** (넘벳) | 차기 모델 출시 예정 |

**Capybara v8의 알려진 문제:**
- 정지 시퀀스 오류 트리거율 ~10%
- 허위 주장율 29-30% (v4의 16.7% 대비 거의 2배)
- 과도한 주석 달기 문제
- 검증 부족 문제

**내부 vs 외부 사용자 분리:**
> "External users get a stripped-down 'be concise' experience, while internal users get the model that actually pushes back on mistakes, verifies its own work, and provides detailed explanations."

내부 전용 도구: REPLTool, SuggestBackgroundPRTool, TungstenTool, VerifyPlanExecutionTool, 에이전트 중첩

### 리포트 03: 언더커버 모드

Anthropic 직원(`USER_TYPE === 'ant'`)이 외부 리포지토리에서 작업할 때 자동 활성화되는 모드.

> "Do not blow your cover." -- 모델이 은밀한 요원으로 프레이밍됨

핵심 지시:
```
Write commit messages as a human developer would -- describe only what the code change does.
```

- **강제 해제 불가**: "There is NO force-OFF. This guards against model codename leaks"
- Co-Authored-By 라인 제거, AI 생성 마커 제거
- 외부 빌드에서는 데드코드 제거됨
- 모델명 마스킹 함수로 코드네임 유출 방지: `capybara-v2-fast` -> `cap*****-v2-fast`

### 리포트 04: 원격 제어와 킬스위치

**원격 관리 설정:**
- 매 1시간마다 `GET /api/claude_code/settings` 폴링
- "위험한" 변경 시 수락-또는-종료 다이얼로그: 거부하면 앱 종료
- 캐시된 설정은 서버 다운 시에도 유지

**주요 킬스위치:**

| 메커니즘 | 대상 | 사용자 알림 |
|---------|------|-----------|
| 원격 관리 설정 | Enterprise/Team | 수락-또는-종료 대화상자 |
| GrowthBook 피처 플래그 | 전체 사용자 | 알림 없음 |
| 킬스위치 | 전체 사용자 | 알림 없음 |
| 모델 오버라이드 | 내부(ant) | 알림 없음 |
| Fast 모드 제어 | 전체 사용자 | 알림 없음 |

> "Fast mode can be **permanently** disabled for a specific user. Not temporarily. Permanently."

### 리포트 05: 미래 로드맵

**KAIROS -- 자율 에이전트 모드:**
> "You are running autonomously. You will receive <tick> prompts that keep you alive between turns. If you have nothing useful to do, call SleepTool. Bias toward action."

- 터미널 포커스 감지: 사용자 부재 시 더 독립적으로 행동
- SleepTool, PushNotificationTool, SubscribePRTool, BriefTool 등 전용 도구
- GitHub PR 웹훅 모니터링, 독립적 커밋/푸시 가능

**미출시 도구들:**
- WebBrowserTool (코드네임 "bagel") -- 내장 브라우저 자동화
- TerminalCaptureTool, WorkflowTool, MonitorTool, SnipTool
- ListPeersTool -- Unix 도메인 소켓으로 피어 에이전트 발견
- Coordinator Mode -- 다중 에이전트 협업

**버디 시스템 (가상 펫):**
- 18종, 5단계 희귀도, 7종 모자, 5개 스탯 (DEBUGGING, PATIENCE, CHAOS, WISDOM, SNARK)
- 사용자 ID 해시 기반 결정론적 생성

### 리포트 06: 에이전트 루프 심층 분석

`query.ts` (1,729줄)가 시스템의 심장부.

**핵심 구조:**
- `query()` 함수는 async generator로 5가지 이벤트 타입 스트림 반환
- `queryLoop()`은 `while (true)` 루프로 매 반복이 Claude API 왕복 1회
- 전환 이유를 기록하는 `transition` 필드: `'next_turn'`, `'reactive_compact_retry'`, `'max_output_tokens_recovery'` 등

**스트리밍 도구 실행:**
> "The model is still streaming tokens while the first tool calls are already running. This is a major latency win."

- 동시성 안전 도구(읽기, 검색)는 병렬 실행, 비동시성 도구(쓰기, bash)는 독점 실행
- Bash 오류 시 형제 도구 중단, 읽기 도구 오류는 독립적

**오류 복구 메커니즘 7가지:**
1. 모델 폴백 (FallbackTriggeredError)
2. 최대 출력 토큰 복구 (단계적 재시도)
3. 반응적 압축 (prompt-too-long 시 자동 요약)
4. 스트리밍 폴백 톰스톤
5. 서브에이전트 아키텍처
6. 자동 압축
7. 비용 추적 및 예산 관리

### 리포트 07: 도구 시스템 아키텍처

**`buildTool` 팩토리 패턴:**
> "having one factory that produces complete Tool objects from partial definitions means there is exactly one place where defaults are established"

- 기본값은 fail-closed: `isConcurrencySafe` -> `false`, `isReadOnly` -> `false`
- 도구 인터페이스 60개 이상 필드/메서드
- Zod 스키마로 입력 검증, `lazySchema()`로 순환 임포트 방지

**도구 필터링 파이프라인:**
1. `getAllBaseTools()` -- 피처 플래그/환경변수 게이팅
2. `getTools()` -- 차단 도구 필터, 단순 모드 처리
3. `filterToolsByDenyRules()` -- 거부 규칙 매칭
4. `assembleToolPool()` -- 내장 + MCP 도구 병합, 프롬프트 캐시 안정성 위한 정렬

**BashTool 특별 처리:**
- tree-sitter로 AST 파싱하여 보안 분석
- `sed -i` 인터셉트: diff 미리보기 제공 후 시뮬레이션 경로로 실행
- 자동 백그라운드화 (15초 이상 실행 시)

### 리포트 08: 권한 및 보안 모델

**권한 모드:**

| 모드 | 설명 |
|------|------|
| `default` | 표준 대화형, 쓰기 작업마다 승인 요청 |
| `plan` | 읽기 전용 계획 모드 |
| `acceptEdits` | 파일 편집/특정 bash 명령 자동 승인 |
| `bypassPermissions` | YOLO 모드, 모든 도구 자동 승인 |
| `dontAsk` | 승인 필요 작업 자동 거부 (헤드리스 CI용) |
| `auto` | AI 분류기 중재 모드 (내부 전용) |
| `bubble` | 다중 에이전트 스웜 조정용 (내부 전용) |

**Bash 보안 계층:**
1. 명령 치환 감지 (`$()`, `${}`, 백틱 등)
2. Zsh 위험 명령 차단 (`zmodload`, `sysopen`, `ztcp` 등)
3. Heredoc 안전성 분석
4. 읽기 전용 명령 검증
5. 복합 명령 분리 (최대 50개 서브커맨드)
6. 경로 검증

**샌드박스:**
- macOS seatbelt, Linux bubblewrap 기반
- 맨 git 리포지토리 공격 방지
- 설정 파일 자체 보호 (설정 파일에 대한 쓰기 무조건 거부)

**SSRF 가드:**
> "Blocks private, link-local, and non-routable address ranges... Notably, loopback is explicitly allowed because local dev servers are a primary HTTP hook use case."

### 리포트 09: 시스템 프롬프트 엔지니어링

**15,000+ 토큰의 시스템 프롬프트 조립 파이프라인:**

정적/동적 분리 마커 `__SYSTEM_PROMPT_DYNAMIC_BOUNDARY__` 사용:
- 마커 이전: `cacheScope: 'global'` -- 전체 Anthropic fleet에서 공유
- 마커 이후: 세션별, 사용자별, 프로젝트별 동적 콘텐츠

**20개 이상의 프롬프트 섹션:**
1. IntroSection -- 정체성과 안전 지침
2. SystemSection -- 시스템 작동 방식
3. DoingTasksSection -- 코딩 행동 지침
4. ActionsSection -- 안전 가드레일
5. UsingYourToolsSection -- 도구 사용 정책
6. ToneAndStyleSection -- 톤과 스타일
7. OutputEfficiencySection -- 출력 효율성
8. 동적 섹션들: 세션 가이던스, 메모리, 환경 정보, 언어, MCP 지침 등

**내부 vs 외부 차이:**
- 외부: "Be extra concise"
- 내부: 다단락 에세이, 역피라미드 구조 설명, "users may have stepped away and lost the thread" 반영

### 리포트 10: MCP 통합

**6가지 전송 계층 지원:**
- stdio, SSE, Streamable HTTP, WebSocket, SDK Control, In-Process, claude.ai Proxy

**서버 라이프사이클:**
- 설정 소스 우선순위: Plugin < User < Project < Local (Enterprise는 독점)
- 자동 재연결: 지수 백오프 (최초 1초, 최대 30초, 5회 시도)
- 도구 호출 타임아웃: ~27.8시간 (기본값)

**OAuth 통합:**
- RFC 6749 인증 코드 플로우 + PKCE
- macOS Keychain을 통한 토큰 저장
- XAA(Cross-App Access)로 엔터프라이즈 SSO 지원

**보안 모델:**
> "MCP tools are namespaced as `mcp__serverName__toolName`. This namespacing prevents a malicious MCP tool from masquerading as a built-in tool."

### 리포트 11: 컨텍스트 윈도우 관리

**4단계 압축 전략:**

1. **Snip Compaction** -- 오래된 저가치 메시지 제거 (API 호출 없음)
2. **Microcompact** -- 대형 도구 결과 개별 압축
3. **Context Collapse** -- 메시지 그룹을 요약으로 아카이브 (가역적)
4. **Auto-Compact** -- 전체 대화 요약 (핵 옵션)

**자동 압축 임계값:** 200K 윈도우 기준 ~167K 토큰 (83%)

**서킷 브레이커:**
> "Before the circuit breaker, some sessions were hammering the API with over 3,000 doomed compaction attempts. At scale, that was a quarter million wasted API calls per day."

최대 3회 연속 실패 후 중단.

**캐시 경제학:**
- 캐시된 Microcompact: `cache_edits`로 서버 사이드 캐시 무효화 없이 도구 결과 삭제
- 시간 기반 Microcompact: 캐시 이미 만료 시에만 로컬 메시지 직접 수정

### 리포트 12: 상태 관리

**이중 레이어 구조:**
1. **부트스트랩 싱글턴** (`bootstrap/state.ts`) -- React 마운트 전 필요한 상태, 100개 이상 필드
2. **리액티브 스토어** (`state/store.ts`) -- 34줄의 미니멀 구현, Redux/Zustand 없음

**대화 기록:**
- JSONL 파일로 저장 (`~/.claude/projects/<sanitized-cwd>/<sessionId>.jsonl`)
- UUID 기반 부모-자식 체인으로 분기 이력 지원
- 사이드체인으로 탐색적 막다른 길 보존

**세션 복구:**
- 중단 감지: `interrupted_prompt` / `interrupted_turn` / `none`
- 비용 상태, 워크트리 상태, 에이전트 설정, 컨텍스트 붕괴 커밋 로그 복원

### 리포트 13: 아키텍처 개요

**7개 주요 레이어:**
1. 진입 및 세션 부트스트랩
2. 프롬프트 조립 및 환경 주입
3. 메인 에이전트 루프
4. 도구 런타임 및 권한 적용
5. 컨텍스트 관리 및 압축
6. 영속성 및 메모리
7. 통합 표면 및 원격 제어

핵심 통찰:
> "Claude Code is best understood as a **production-grade agent runtime** with a terminal interface, not as a terminal UI with a model attached."

최적화 우선순위:
1. 개념적 깔끔함보다 실제 견고성
2. 순수 채팅 품질보다 도구 사용 효과성
3. 짧은 턴 단순성보다 긴 세션 연속성
4. 정적 제품 동작보다 운영 제어
5. 단일 신뢰 메커니즘보다 안전 레이어

### 리포트 14: BashTool 보안 및 실행

BashTool은 자체적인 미니 보안 아키텍처:

**보안 파이프라인:**
1. 명령 파싱 및 정규화
2. 위험 구문 패턴 감지
3. 복합 명령 세그먼트 분리
4. 서브커맨드 읽기전용/변형/알수없음 분류
5. 경로 및 파일시스템 영향 검증
6. 허용/거부 규칙 적용
7. 샌드박싱 적용 여부 결정
8. 불확실성 시 사용자 승인 요청

> "Claude Code does not try to make Bash perfectly safe. That would be unrealistic. Instead, it tries to make Bash analyzable enough, permissionable enough, sandboxable enough, monitorable enough, predictable enough."

**sed 인터셉션:** `sed -i` 명령을 파싱하여 diff 미리보기 제공 후 시뮬레이션된 편집 경로로 실행 -- "do not trust raw shell behavior if it can be transformed into a more inspectable structured operation"

### 리포트 15: 메모리 및 지시 시스템

**3가지 메모리 카테고리:**

1. **지시적 메모리** -- CLAUDE.md, rules/*.md (행동 지시)
2. **영속적 작업 메모리** -- MEMORY.md + 주제 파일 (세션 간 사실 보존)
3. **세션 메모리** -- 런타임 메모리 추출 (압축 및 연속성)

**MEMORY.md 설계:**
> "If the always-loaded memory entrypoint were allowed to grow without limits, it would quickly become a context-budget sink. By forcing MEMORY.md to act as a lightweight index, the system preserves navigability without spending all of its prompt budget on persistent notes."

- 200줄 / 25KB 하드 리밋
- 인덱스 역할, 실제 내용은 개별 주제 파일에 저장

### 리포트 16: 트랜스크립트, 압축, 세션 재개

**트랜스크립트 구조:**
- UUID 기반 부모-자식 체인 (단순 평면 로그 아님)
- 사이드체인으로 분기 이력 보존
- 세션 파일 지연 생성 (유령 세션 방지)

**압축 경계:**
> "Compaction is not just a token-saving transformation. It changes the shape of the stored conversation."
- SystemCompactBoundaryMessage로 의미적 경계 표시
- 재개/계속/포크는 서로 다른 연속성 연산

### 리포트 17: 브릿지 시스템 및 원격 세션

**두 가지 배포 형태:**
- 독립 브릿지 (`bridgeMain.ts`): 장기 실행 프로세스, 미니 오케스트레이터
- REPL 브릿지: 기존 세션에 피기백

**세션 라이프사이클:**
- 등록 -> 폴링 -> 확인 -> 하트비트 -> 완료 -> 종료
- 30초 대기 후 SIGKILL 에스컬레이션
- 3계층 인증: OAuth, Work Secret (세션별 JWT), Trusted Device Token

**멀티세션 관리:**
- 3가지 스폰 모드: single-session, same-dir, worktree
- 최대 32개 동시 세션 (멀티세션 모드)
- 수면 감지: 틱 간격이 2배 초과 시 에러 예산 리셋

### 리포트 18: React/Ink 터미널 UI

**기술 스택:**
- Ink 커스텀 포크 + React Reconciler + Yoga 레이아웃 (순수 TypeScript)
- 346개 TSX 컴포넌트
- 60fps 목표, 16ms 프레임 간격

**핵심 기술:**
- 더블 버퍼링 (GPU 렌더링과 동일 원리)
- 스타일 인터닝 (StylePool, CharPool)
- blit 최적화 -- 변경되지 않은 서브트리는 이전 프레임에서 직접 복사
- 하드웨어 스크롤 영역 (DECSTBM + SU/SD)
- React Compiler 사용 (자동 메모이제이션)

**브라우저 DOM 이벤트 모델 충실 포팅:**
- 캡처/버블 2단계
- 마우스 히트 테스팅
- 포커스 매니저 (탭 순환, 자동 포커스)

### 리포트 19: 스트리밍 및 전송 레이어

**3가지 전송 구현:**

| 전송 | 읽기 | 쓰기 | 세대 |
|------|------|------|------|
| WebSocket | WS | WS | v1 (원본) |
| Hybrid | WS | HTTP POST | v1.5 (과도기) |
| SSE | SSE | HTTP POST | v2 (최신) |

**SerialBatchEventUploader:**
- 직렬 배치 업로드, 백프레셔, 지수 백오프
- 스트림 이벤트 100ms 버퍼링으로 토큰별 HTTP POST 방지
- 직렬화 불가능 아이템 자동 드롭

**NDJSON 안전성:**
> "U+2028 (LINE SEPARATOR) and U+2029 (PARAGRAPH SEPARATOR) ... When the NDJSON protocol splits on line boundaries, a raw U+2028 inside a JSON string will split the message in half."

**수면/깨우기 감지:** setInterval 틱 간격 60초 초과 시 소켓 사망 판정

### 리포트 20: 슬래시 명령 및 비용 추적

**80개 이상 슬래시 명령:**
- 3계층: 정적 임포트, 피처 게이트 임포트, 동적 소스 (스킬/플러그인/MCP/워크플로)
- 3가지 명령 타입: `local` (동기 실행), `local-jsx` (React 렌더), `prompt` (모델에 주입)
- Fuse.js 퍼지 검색 (명령명 가중치 3, 설명 가중치 0.5)

**비용 추적:**
- 모델별 7차원 사용량: 입력/출력/캐시읽기/캐시생성 토큰, 웹검색 요청, USD, 컨텍스트 윈도우
- 가격 티어: Sonnet급 $3/$15, Opus 4.5/4.6 $5/$25, Opus 4.6 Fast $30/$150
- 세션 재개 시 비용 상태 복원 (세션 ID 매칭 필수)
- advisor 사용량 재귀적 추적

---

## 3. 각 리포트에서 발견된 주요 아키텍처 인사이트

### 설계 원칙

1. **Fail-Closed 철학**: 기본값이 안전한 방향. `isConcurrencySafe` 기본값 `false`, 분류기 실패 시 거부, 파싱 불가 명령은 승인 요청.

2. **계층적 방어 (Defense in Depth)**: Bash 명령 하나가 보안 검사(bashSecurity.ts) -> 읽기전용 검증 -> 모드 기반 확인 -> 경로 검증 -> 권한 규칙 -> 분류기(자동 모드 시) -> 샌드박스를 모두 통과해야 함.

3. **프롬프트 캐시 인식**: 시스템 전반에 걸쳐 캐시 히트율을 극도로 의식. 정적/동적 경계 마커, SDK 소비자용 어시스턴트 메시지 복제, 도구 정렬 안정성 등.

4. **장기 세션 우선 설계**: 자동 압축, 세션 메모리, 트랜스크립트 체인, 사이드체인, 워크트리 격리 등 모든 것이 세션이 오래 지속될 것을 전제.

5. **점진적 인프라 마이그레이션**: 전송 계층의 v1(WS) -> v1.5(Hybrid) -> v2(SSE) 진화, 피처 플래그 뒤에서 동시 운영.

### 핵심 아키텍처 패턴

- **Async Generator 기반 에이전트 루프**: `query()`가 스트리밍 상태 머신으로 동작, 단순 요청/응답이 아닌 복합 이벤트 스트림
- **buildTool 팩토리**: 40개 이상 도구의 일관된 등록/검증/실행 계약
- **리액티브 스토어 (34줄)**: Redux/Zustand 없이 `Object.is` 체크와 클로저로 구현
- **트랜스크립트 체인**: UUID 기반 부모-자식 링크로 분기/합류/압축 지원
- **서브에이전트 풀 피델리티**: 자식 에이전트가 부모와 동일한 `query()` 함수 사용, 격리된 도구 풀과 MCP 서버

---

## 4. 코드네임과 피처 플래그 정리

### 모델 코드네임

| 코드네임 | 대상 | 비고 |
|---------|------|------|
| Tengu (천구) | 제품/텔레메트리 접두사 | 모든 GrowthBook 플래그에 `tengu_` 접두사 |
| Capybara (카피바라) | Sonnet 계열 (현재 v8) | `capybara-v2-fast[1m]` |
| Fennec (페넥여우) | Opus 4.6 전신 | `fennec-latest` -> `opus` 마이그레이션 |
| Numbat (넘벳) | 차기 모델 | "Remove this section when we launch numbat" |
| Penguin (펭귄) | Fast 모드 | `tengu_penguins_off` |
| Bagel (베이글) | WebBrowserTool | 내부 코드네임 |

### 주요 피처 플래그

| 플래그 | 기능 |
|--------|------|
| `tengu_onyx_plover` | Auto Dream (백그라운드 메모리 통합) |
| `tengu_coral_fern` | Memdir 기능 |
| `tengu_herring_clock` | Team 메모리 |
| `tengu_sedge_lantern` | Away Summary |
| `tengu_frond_boric` | 분석 킬스위치 |
| `tengu_amber_quartz_disabled` | 음성 모드 킬스위치 |
| `tengu_amber_flint` | 에이전트 팀 |
| `tengu_hive_evidence` | 검증 에이전트 |
| `tengu_penguins_off` | Fast 모드 비활성화 |
| `tengu_marble_sandcastle` | Fast 모드 관련 |
| `tengu_ant_model_override` | 내부 사용자 모델 오버라이드 |
| `tengu_bridge_repl_v2` | env-less 브릿지 |
| `tengu_ccr_bridge_multi_session` | 멀티세션 브릿지 |
| `tengu_harbor_ledger` | 채널 서버 허용 목록 |

**명명 규칙:** `tengu_` + 임의 단어 쌍 (형용사/재료 + 자연/사물). 의도적으로 기능 추론을 방지하는 운영 보안 설계.

### 예정된 모델 버전
- Opus 4.7, Sonnet 4.8 (언더커버 모드 지시문에서 발견)
- `@[MODEL LAUNCH]` 마커 20개 이상이 코드베이스에 산재

---

## 5. 보안 모델 분석 요약

### 잘 설계된 부분

1. **계층적 방어**: 단일 레이어에 의존하지 않음. Bash 명령이 보안 검사, 읽기전용 검증, 모드 기반 확인, 경로 검증, 권한 규칙, 분류기, 샌드박스를 모두 통과
2. **Fail-Closed 철학**: 분류기 API 실패 시 거부, 파싱 불가 명령은 승인 요청, 모호한 상황은 프롬프트
3. **TOCTOU 인식**: 셸 확장 구문, 틸다 변형, 심볼릭 링크 순회, macOS/Windows 대소문자 정규화 처리
4. **Zsh 공격 표면 커버리지**: `zmodload`, `sysopen`, `ztcp`, `zf_*` 빌트인 등 차단
5. **맨 git 리포지토리 공격 방지**: `is_git_directory()` + `core.fsmonitor` 메커니즘 처리
6. **설정 파일 자체 보호**: 샌드박스가 자체 설정 파일과 `.claude/skills` 쓰기를 무조건 거부
7. **SSRF 가드**: DNS rebinding 공격 방지를 위한 `dns.lookup` 콜백 검증

### 잠재적 우려

1. **복잡성 자체가 공격 표면**: bashSecurity.ts만 23개 이상의 보안 검사 카테고리 처리
2. **분류기가 보안 경계**: AI 모델이 보안 결정을 내리는 확률적 시스템
3. **외부 빌드 스텁**: 외부 사용자는 분류기 보호 없이 정적 규칙 + 샌드박스에만 의존
4. **excludedCommands는 보안 경계 아님**: 사용자 편의 기능으로 명시적 문서화
5. **프리승인 도메인에 업로드 가능 사이트 포함**: nuget.org 등

### 다른 에이전트 시스템과의 비교
- vs Cursor/Windsurf: Claude Code는 OS 수준 샌드박싱 제공 (대부분의 IDE 기반 에이전트는 미제공)
- vs Devin: Claude Code의 권한 규칙이 더 세분화 (도구 수준, 접두사 기반, 와일드카드)
- vs OpenAI Codex CLI: Claude Code의 bash 보안 분석이 훨씬 더 철저

---

## 6. 미공개 기능 분석 요약

### 출시 예정 기능

| 기능 | 상태 | 설명 |
|------|------|------|
| **KAIROS** | 피처 게이트 | 자율 에이전트 모드, 터미널 포커스 감지, 푸시 알림 |
| **Voice Mode** | 구현 완료, 게이트 | 푸시투토크 음성 입력, OAuth 전용 |
| **WebBrowserTool** ("bagel") | 구현, 게이트 | 내장 브라우저 자동화 |
| **Coordinator Mode** | 개발중 | 다중 에이전트 협업 |
| **Buddy System** | 완전 구현 | 가상 펫 (18종, 가챠 시스템) |
| **Dream Task** | 피처 게이트 | 백그라운드 메모리 통합 |
| **Numbat** | 계획 | 차기 모델 |
| **Opus 4.7 / Sonnet 4.8** | 계획 | 차기 모델 버전 |

### 미출시 도구

| 도구 | 피처 플래그 | 기능 |
|------|-----------|------|
| SleepTool | KAIROS/PROACTIVE | 자율 행동 간 페이싱 |
| PushNotificationTool | KAIROS | 기기 푸시 알림 |
| SubscribePRTool | KAIROS_GITHUB_WEBHOOKS | GitHub PR 웹훅 구독 |
| TerminalCaptureTool | TERMINAL_PANEL | 터미널 패널 캡처/모니터링 |
| WorkflowTool | WORKFLOW_SCRIPTS | 사전 정의 워크플로 실행 |
| MonitorTool | MONITOR_TOOL | 시스템/프로세스 모니터링 |
| ListPeersTool | UDS_INBOX | 피어 에이전트 발견 |
| RemoteTriggerTool | AGENT_TRIGGERS_REMOTE | 원격 에이전트 트리거 |

### 전략적 방향

1. **보조 -> 자율**: KAIROS가 상시 가동 자율 개발 에이전트로의 전환을 나타냄
2. **멀티모달 확장**: 음성 입력, 브라우저 자동화, 워크플로 오케스트레이션
3. **다중 에이전트**: Coordinator Mode, ListPeersTool, UDS 기반 에이전트 간 통신

---

## 7. 텔레메트리/프라이버시 분석 요약

### 수집 데이터

| 카테고리 | 세부 항목 |
|---------|---------|
| 환경 핑거프린트 | OS, 아키텍처, Node 버전, 터미널 타입, 패키지 매니저, CI/CD, WSL, 리눅스 배포판 |
| 프로세스 메트릭 | uptime, RSS, heapTotal, heapUsed, CPU 사용량 |
| 사용자/세션 추적 | 모델명, 세션 ID, 사용자 ID, 기기 ID, 계정 UUID, 조직 UUID, 구독 티어 |
| 리포지토리 핑거프린팅 | git remote URL의 SHA256 해시 처음 16자 |
| 도구 입력 | 기본: 512자 잘림, JSON 4096자, 배열 20개, 중첩 2레벨 |
| 파일 확장자 | bash 명령에서 사용된 파일 확장자 추출 |

### 옵트아웃 문제

> "The first-party logging pipeline **cannot be disabled** if you're using the direct Anthropic API."

- 비활성화 가능 조건: 테스트 환경, 3자 클라우드 제공자(Bedrock, Vertex), 비공개 글로벌 옵트아웃 플래그
- 사용자 인터페이스에 노출된 옵트아웃 방법 없음
- `OTEL_LOG_TOOL_DETAILS=1` 시 전체 도구 입력 무잘림 로깅 (백도어)

### A/B 테스트

- GrowthBook 통합으로 사용자를 실험 그룹에 배정
- 사용자 속성 전송: id, sessionId, deviceID, platform, organizationUUID, subscriptionType
- 가시적 표시기 없음

### 원격 제어 투명성 문제

> "There's no audit log, no notification system, no way for a user to know when their Claude Code instance has been remotely modified by a feature flag change."

**결론:** Claude Code의 텔레메트리는 제품 분석 수준의 광범위한 환경 프로파일링을 수행하며, 직접 API 사용자에게 실질적인 옵트아웃 방법을 제공하지 않는다. 소스 코드 유출 증거는 없으나, 수집 범위와 옵트아웃 부재는 프라이버시 관점에서 주목할 만하다. 원격 제어 인프라는 사용자에게 투명하지 않은 방식으로 동작을 변경할 수 있는 광범위한 능력을 갖추고 있다.
