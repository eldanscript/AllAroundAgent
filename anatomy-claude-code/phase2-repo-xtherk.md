# Phase 2: xtherk/open-claude-code 리포지토리 분석

## 1. 리포지토리 개요

**리포지토리**: xtherk/open-claude-code  
**기반 버전**: `@anthropic-ai/claude-code@2.1.88`  
**핵심 특징**: 소스맵에서 추출한 소스코드를 **실제 빌드 가능한 상태**로 복원한 프로젝트

### 다른 소스맵 추출과의 차이점

단순 소스맵 추출 리포지토리들이 `.ts`/`.tsx` 파일을 그대로 나열하는 데 그치는 반면, 이 리포지토리는 다음과 같은 추가 작업을 수행했다:

1. **빌드 파이프라인 복원**: `bun build`로 단일 번들을 생성하는 `package.json` scripts 구성
2. **네이티브 모듈 대체**: Rust NAPI 모듈(`color-diff-napi`, `file-index`, `yoga-layout`)을 순수 TypeScript로 재구현
3. **빌드 매크로 호환 레이어**: `MACRO.*` 빌드 타임 상수와 `bun:bundle`의 `feature()` 함수를 런타임 심으로 대체
4. **스텁 패키지 생성**: 비공개/네이티브 의존성을 로컬 스텁으로 대체
5. **비활성 도구 팩토리**: 복원 불가능한 도구를 안전하게 비활성화하는 패턴 도입

README에 명시된 대로:
> "Codex 驱动完成" -- 이 리포의 복원, 빌드 수정, 문서화 작업은 OpenAI Codex가 수행

---

## 2. 빌드 시스템 분석

### package.json scripts

```json
{
  "scripts": {
    "build": "bun build src/entrypoints/cli.tsx --outdir ./dist --target node --sourcemap",
    "start": "bun dist/cli.js",
    "smoke": "node ./dist/cli.js --version"
  }
}
```

- **build**: Bun 번들러로 `src/entrypoints/cli.tsx`를 엔트리포인트로 하여 `dist/cli.js` 단일 번들 생성. `--target node`로 Node.js 런타임 대상, `--sourcemap`으로 디버깅용 소스맵 포함
- **start**: Bun 런타임으로 빌드 결과물 실행
- **smoke**: Node.js로 `--version` 플래그 실행하여 빌드 성공 여부를 빠르게 검증

### 빌드 환경 요구사항

```
- Node.js >= 18.0.0
- Bun (빌드 및 실행)
- npm (의존성 설치)
```

### 빌드 엔트리포인트 구조

`package.json`의 `bin` 필드:
```json
{
  "bin": {
    "claude": "dist/cli.js"
  }
}
```

빌드 후 `dist/cli.js`가 CLI 진입점이 된다.

---

## 3. 빌드 가능하게 만들기 위해 추가/수정된 부분

### 3.1 recovery 디렉토리 -- 빌드 매크로 호환 레이어

원본 Claude Code는 빌드 시 `MACRO.*` 상수를 인라인하고, `bun:bundle`의 `feature()` 함수로 조건부 코드 제거(DCE)를 수행한다. 소스맵 추출 결과에는 이 빌드 인프라가 없으므로 두 개의 심(shim) 파일을 만들었다.

**`src/recovery/macroShim.js`** -- 빌드 타임 상수 대체:

```javascript
/**
 * 恢复源码兼容层：
 * 原始构建流程会在打包时注入 MACRO.* 常量，这里改为静态对象，先保证源码可以继续编译。
 */
export const RECOVERY_MACRO = {
  "BUILD_TIME": "2026-03-31T09:28:16.558Z",
  "FEEDBACK_CHANNEL": "github",
  "ISSUES_EXPLAINER": "https://github.com/anthropics/claude-code/issues",
  "NATIVE_PACKAGE_URL": "",
  "PACKAGE_URL": "https://github.com/anthropics/claude-code",
  "VERSION": "2.1.88",
  "VERSION_CHANGELOG": "https://github.com/anthropics/claude-code/releases"
};
```

이 파일은 **59개 소스 파일**에서 `import { RECOVERY_MACRO } from '../recovery/macroShim.js'`로 참조된다. `RECOVERY_MACRO.VERSION`, `RECOVERY_MACRO.PACKAGE_URL` 등으로 원본의 `MACRO.VERSION`, `MACRO.PACKAGE_URL`을 대체한다.

**`src/recovery/bunBundleShim.js`** -- feature flag 런타임 대체:

```javascript
/**
 * 恢复源码兼容层：
 * 原始源码依赖 bun:bundle 的 feature() 做编译期开关裁剪。
 * 这里改为通过环境变量 CLAUDE_CODE_FEATURES=FLAG_A,FLAG_B 进行运行时判定，
 * 以保证恢复后的源码至少可以继续被编译。
 */
const enabledFeatures = new Set(
  (process.env.CLAUDE_CODE_FEATURES ?? '')
    .split(',')
    .map((item) => item.trim())
    .filter(Boolean),
);

export function feature(name) {
  return enabledFeatures.has(name);
}
```

원본에서 `import { feature } from 'bun:bundle'`를 사용하는 파일이 **196개**에 달한다. 이 심을 통해 `ABLATION_BASELINE`, `DUMP_SYSTEM_PROMPT` 같은 내부 feature flag가 환경변수 `CLAUDE_CODE_FEATURES`로 런타임 제어 가능해진다.

CLI 엔트리포인트(`src/entrypoints/cli.tsx`)에서의 사용 예:

```typescript
import { RECOVERY_MACRO } from '../recovery/macroShim.js';
import { feature } from 'bun:bundle';

// Fast-path for --version
if (args.length === 1 && (args[0] === '--version' || args[0] === '-v')) {
    console.log(`${RECOVERY_MACRO.VERSION} (Claude Code)`);
    return;
}

// Feature flag로 내부 전용 기능 게이팅
if (feature('DUMP_SYSTEM_PROMPT') && args[0] === '--dump-system-prompt') { ... }
```

### 3.2 stubs 디렉토리 -- 비공개 의존성 대체

원본에서 npm 레지스트리에 공개되지 않았거나 네이티브 바이너리가 필요한 3개 패키지를 로컬 스텁으로 대체했다.

**`stubs/ant-claude-for-chrome-mcp/`** -- Chrome 확장 MCP 서버:

```javascript
// index.js
export const BROWSER_TOOLS = [];
export function createClaudeForChromeMcpServer() {
  return {
    async connect() {},
    async close() {},
  };
}
```
```json
// package.json
{ "name": "@ant/claude-for-chrome-mcp", "version": "0.0.0-recovered", "type": "module", "main": "./index.js" }
```

**`stubs/color-diff-napi/`** -- 네이티브 색상 차이 모듈:

```javascript
export class ColorDiff {}
export class ColorFile {}
export function getSyntaxTheme() { return null; }
```

**`stubs/modifiers-napi/`** -- 키보드 수정자 키 감지:

```javascript
export function prewarm() {}
export function isModifierPressed() { return false; }
```

`package.json`에서 이 스텁들을 로컬 경로로 참조:

```json
"@ant/claude-for-chrome-mcp": "./stubs/ant-claude-for-chrome-mcp",
"color-diff-napi": "./stubs/color-diff-napi",
"modifiers-napi": "./stubs/modifiers-napi"
```

### 3.3 native-ts 디렉토리 -- Rust NAPI 모듈의 TypeScript 재구현

이 리포의 가장 주목할 만한 기술적 성과다. 원본 Claude Code가 성능을 위해 Rust NAPI로 구현한 3개의 핵심 모듈을 순수 TypeScript로 재구현했다.

**`src/native-ts/color-diff/index.ts`** (999줄) -- 구문 강조 + 인라인 diff 렌더링:
- 원본: Rust의 syntect + bat + similar 크레이트 기반
- 재구현: highlight.js + diff 패키지의 `diffArrays` 기반
- `ColorDiff`, `ColorFile`, `getSyntaxTheme` API를 원본과 동일하게 유지
- 주석에서 차이점을 명시: "plain identifiers and operators like `=` `:` aren't scoped, so they render in default fg instead of white/pink"

```typescript
/**
 * Pure TypeScript port of vendor/color-diff-src.
 * API matches vendor/color-diff-src/index.d.ts exactly so callers don't change.
 */
export class ColorDiff {
  render(themeName: string, width: number, dim: boolean): string[] | null { ... }
}
export class ColorFile {
  render(themeName: string, width: number, dim: boolean): string[] | null { ... }
}
```

**`src/native-ts/file-index/index.ts`** (371줄) -- 퍼지 파일 검색 인덱스:
- 원본: Rust의 nucleo 크레이트 (helix 에디터의 퍼지 매처) 기반
- 재구현: 비트맵 필터링 + indexOf 스캔 + nucleo 스타일 스코어링
- `FileIndex.loadFromFileList()`, `FileIndex.search()` API 유지

```typescript
/**
 * Pure-TypeScript port of vendor/file-index-src (Rust NAPI module).
 * Score semantics: lower = better. Paths containing "test" get a 1.05x penalty.
 */
export class FileIndex {
  loadFromFileList(fileList: string[]): void { ... }
  search(query: string, limit: number): SearchResult[] { ... }
}
```

**`src/native-ts/yoga-layout/index.ts`** (2579줄) -- 터미널 UI 레이아웃 엔진:
- 원본: Meta의 Yoga (C++ WASM 바인딩)
- 재구현: 순수 TypeScript flexbox 엔진
- flex-direction, flex-grow/shrink/basis, align-items, justify-content, margin/padding/border/gap, wrap/wrap-reverse, baseline alignment 등 구현
- 주석에서 미구현 항목 명시: "aspect-ratio, box-sizing: content-box, RTL direction"

### 3.4 recovery/createRecoveredDisabledTool -- 복원 불가 도구 처리

원본의 일부 도구는 소스맵에 포함되지 않았거나 내부 인프라에 의존한다. 이를 위한 팩토리 패턴:

```javascript
// src/tools/recovery/createRecoveredDisabledTool.js
export function createRecoveredDisabledTool({ name, searchHint, unavailableMessage, userFacingName }) {
  return buildTool({
    name,
    isEnabled() { return false },
    async call() {
      return { data: { status: 'unavailable', message: unavailableMessage } }
    },
    // ...
  })
}
```

이 팩토리를 사용하는 도구 4개:

| 도구 | 비활성화 사유 |
|------|-------------|
| `TungstenTool` | tmux/terminal 관리 구현 미복원 |
| `SuggestBackgroundPRTool` | 내부 인프라 의존 |
| `VerifyPlanExecutionTool` | 내부 인프라 의존 |
| `REPLTool` | 구현 미복원 |

TungstenTool의 예시:

```javascript
const UNAVAILABLE_MESSAGE = 'Tungsten 在当前恢复版中不可用：原始 tmux/terminal 管理实现未被还原。'
export const TungstenTool = createRecoveredDisabledTool({
  name: 'Tungsten',
  searchHint: 'manage tmux sessions in tungsten',
  unavailableMessage: UNAVAILABLE_MESSAGE,
})
```

---

## 4. 소스 구조와 다른 리포 대비 차이점

### 4.1 전체 규모

| 항목 | 수치 |
|------|------|
| TypeScript/TSX 파일 수 | 1,892개 |
| 총 코드 라인 수 | ~513,000줄 |
| commands/ 하위 항목 | 103개 |
| components/ 하위 항목 | 144개 |
| hooks/ 하위 항목 | 85개 |
| utils/ 하위 항목 | 330개 |
| tools/ 하위 항목 | 48+ 도구 디렉토리 |
| services/ 하위 항목 | 37+ 서비스 |

### 4.2 디렉토리 구조

```
xtherk-open-claude-code/
  package.json          -- 빌드 설정 + 의존성 정의
  package-lock.json     -- 의존성 잠금 (368KB)
  sdk-tools.d.ts        -- SDK 도구 입출력 타입 정의
  src/
    entrypoints/        -- CLI, MCP, SDK 진입점
      cli.tsx           -- CLI 메인 엔트리포인트
      init.ts           -- 초기화 로직 (텔레메트리, 설정, 인증)
      mcp.ts            -- MCP 서버 모드 엔트리포인트
      sdk/              -- SDK 타입 및 스키마
    recovery/           -- [추가됨] 빌드 호환 레이어
      macroShim.js      -- MACRO.* 상수 대체
      bunBundleShim.js  -- bun:bundle feature() 대체
    native-ts/          -- [추가됨] Rust NAPI 모듈 TS 재구현
      color-diff/       -- 구문 강조 + diff 렌더링 (999줄)
      file-index/       -- 퍼지 파일 검색 (371줄)
      yoga-layout/      -- Flexbox 레이아웃 엔진 (2579줄)
    tools/recovery/     -- [추가됨] 비활성 도구 팩토리
    main.tsx            -- 메인 앱 로직 (804KB, ~57,000줄)
    query.ts            -- API 쿼리 루프 (68KB)
    QueryEngine.ts      -- 쿼리 엔진 (46KB)
    commands.ts         -- 슬래시 커맨드 레지스트리
    tools.ts            -- 도구 레지스트리
    ink/                -- 커스텀 Ink 터미널 UI 프레임워크
    ...
  stubs/                -- [추가됨] 비공개 의존성 스텁
  vendor/               -- 플랫폼별 바이너리
    ripgrep/            -- rg 바이너리 (darwin/linux/win32, x64/arm64)
    audio-capture/      -- 음성 캡처 .node (6개 플랫폼)
```

### 4.3 다른 리포 대비 고유한 점

1. **recovery/ + native-ts/ + stubs/ + tools/recovery/**: 이 4개 디렉토리가 이 리포만의 고유 추가분이다. 소스맵 원본 추출에는 없는, 빌드 가능성을 확보하기 위한 작업물이다.

2. **vendor/ 바이너리 포함**: ripgrep과 audio-capture의 멀티플랫폼 네이티브 바이너리가 포함되어 있어, 소스맵만으로는 얻을 수 없는 런타임 의존성도 확보했다.

3. **package-lock.json 포함 (368KB)**: 의존성 버전을 고정하여 재현 가능한 빌드 환경을 보장한다.

4. **중국어 주석/메시지**: recovery 파일과 비활성 도구 메시지가 중국어로 작성되어 있어, 중국 개발자 커뮤니티(LinuxDo 포럼 등) 기반의 프로젝트임을 알 수 있다.

---

## 5. Smoke 테스트 및 검증 방법

### 5.1 기본 검증 흐름

```bash
# 1. 의존성 설치
npm install

# 2. 빌드
npm run build

# 3. 스모크 테스트 -- 버전 출력 확인
npm run smoke
# 예상 출력: "2.1.88 (Claude Code)"

# 4. 직접 실행
node ./dist/cli.js
```

### 5.2 Smoke 테스트의 동작 원리

`npm run smoke`는 `node ./dist/cli.js --version`을 실행한다.

`cli.tsx`의 fast-path에서:

```typescript
if (args.length === 1 && (args[0] === '--version' || args[0] === '-v')) {
    console.log(`${RECOVERY_MACRO.VERSION} (Claude Code)`);
    return;
}
```

이 fast-path는 **모듈 로딩 없이** 즉시 반환하므로, 빌드 결과물의 기본적인 정합성(번들 생성 성공, 매크로 심 작동)을 최소 비용으로 검증한다.

### 5.3 README에 포함된 스크린샷

리포에 `images/snapshot1.png` (시작 화면), `images/snapshot2.png` (대화 화면) 스크린샷이 포함되어 있어, 실제 실행 결과가 검증되었음을 보여준다.

---

## 6. 의존성 분석

### 6.1 주요 런타임 의존성

| 카테고리 | 패키지 | 용도 |
|----------|--------|------|
| **AI/API** | `@anthropic-ai/sdk` ^0.80.0 | Anthropic API 클라이언트 |
| | `@anthropic-ai/bedrock-sdk` ^0.26.4 | AWS Bedrock 통합 |
| | `@anthropic-ai/vertex-sdk` ^0.14.4 | GCP Vertex AI 통합 |
| | `@anthropic-ai/foundry-sdk` ^0.2.3 | Foundry 통합 |
| **MCP** | `@modelcontextprotocol/sdk` ^1.29.0 | MCP 프로토콜 SDK |
| | `@anthropic-ai/mcpb` ^2.1.2 | MCP 브릿지 |
| **UI/터미널** | `react` ^19.2.4 | Ink 기반 터미널 UI |
| | `react-reconciler` ^0.33.0 | 커스텀 React 렌더러 |
| | `chalk` ^5.6.2 | ANSI 색상 |
| | `chokidar` ^5.0.0 | 파일 시스템 감시 |
| **AWS** | `@aws-sdk/client-bedrock` ^3.1020.0 | Bedrock 클라이언트 |
| | `@aws-sdk/credential-providers` ^3.1020.0 | AWS 인증 |
| **텔레메트리** | `@opentelemetry/*` ^0.214.0 / ^2.6.1 | 분산 추적, 메트릭, 로깅 (12개 패키지) |
| **유틸리티** | `zod` ^4.3.6 | 스키마 검증 |
| | `lodash-es` ^4.17.23 | 유틸리티 함수 |
| | `yaml` ^2.8.3 | YAML 파싱 |
| | `marked` ^17.0.5 | 마크다운 렌더링 |

### 6.2 로컬/스텁 의존성 (3개)

```json
"@ant/claude-for-chrome-mcp": "./stubs/ant-claude-for-chrome-mcp",
"color-diff-napi": "./stubs/color-diff-napi",
"modifiers-napi": "./stubs/modifiers-napi"
```

이들은 `npm install` 시 로컬 경로에서 심볼릭 링크로 설치된다.

### 6.3 선택적 의존성 (sharp 플랫폼별 바이너리)

```json
"optionalDependencies": {
    "@img/sharp-darwin-arm64": "^0.34.2",
    "@img/sharp-darwin-x64": "^0.34.2",
    "@img/sharp-linux-arm64": "^0.34.2",
    // ... 9개 플랫폼
}
```

이미지 처리(`sharp`)를 위한 플랫폼별 네이티브 바이너리. 설치 실패해도 빌드에 영향 없음.

### 6.4 vendor 바이너리 의존성

| 바이너리 | 플랫폼 | 용도 |
|----------|--------|------|
| `vendor/ripgrep/rg` | darwin/linux/win32 (x64/arm64) | GrepTool의 백엔드 |
| `vendor/audio-capture/*.node` | darwin/linux/win32 (x64/arm64) | 음성 모드 오디오 캡처 |

### 6.5 의존성 특이사항

- **OpenTelemetry 12개 패키지**: gRPC, HTTP, Proto 3가지 전송 방식 모두 지원. 프로덕션 텔레메트리 인프라의 규모를 보여줌
- **`@azure/identity`**: Azure 인증 지원 (Bedrock/Vertex 외에 Azure OpenAI도 지원 가능)
- **`google-auth-library`**: GCP Vertex AI 인증
- **`sharp`**: 이미지 리사이징/변환 (스크린샷 첨부 기능)
- **`ws`**: WebSocket (원격 세션, 브릿지 모드)
- **`vscode-jsonrpc`**: LSP 통신 (코드 인텔리전스)
- **`@commander-js/extra-typings`**: CLI 인자 파싱 (TypeScript 타입 자동 추론)

---

## 요약

xtherk/open-claude-code는 Claude Code v2.1.88의 소스맵 추출물을 **실제로 빌드하고 실행할 수 있는 상태**로 복원한 유일한 리포지토리다. 핵심 기여는:

1. **빌드 매크로/feature flag 심 레이어** (macroShim.js, bunBundleShim.js)로 빌드 타임 의존성 제거
2. **3개 Rust NAPI 모듈을 순수 TypeScript로 재구현** (color-diff 999줄, file-index 371줄, yoga-layout 2579줄)
3. **3개 비공개 패키지를 스텁으로 대체**, 4개 복원 불가 도구를 안전하게 비활성화
4. **멀티플랫폼 vendor 바이너리** (ripgrep, audio-capture) 포함

총 ~513,000줄, 1,892개 소스 파일 규모로, Claude Code의 전체 아키텍처를 로컬에서 탐색하고 연구할 수 있는 기반을 제공한다.
