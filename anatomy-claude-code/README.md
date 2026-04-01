# Anatomy of Claude Code

Claude Code 내부 아키텍처 완전 분석 프로젝트

## 개요

2026년 3월 31일, npm 패키지 `@anthropic-ai/claude-code` v2.1.88에 포함된 `.map` 소스맵 파일을 통해
Claude Code의 전체 내부 소스코드(4,600+ 파일)가 노출되었다.
공식 오픈소스 리포지토리에는 279개의 플러그인 파일만 공개되어 있었으나,
실제 핵심 엔진은 완전히 별도의 상용 코드베이스였다.

이 프로젝트는 공개된 코드와 분석 자료를 종합하여 Claude Code의 아키텍처를 
**동일 기능 복제가 가능한 수준**으로 분석한다.

## 참조 리포지토리

| 디렉토리 | 출처 | 설명 |
|----------|------|------|
| `refs/anthropics-claude-code` | [anthropics/claude-code](https://github.com/anthropics/claude-code) | Anthropic 공식 (플러그인/문서만 포함) |
| `refs/chatgptprojects-claude-code` | [chatgptprojects/claude-code](https://github.com/chatgptprojects/claude-code) | npm 공개 내용 기반 정리본 |
| `refs/nirholas-claude-code` | [nirholas/claude-code](https://github.com/nirholas/claude-code) | 완전한 소스 + 웹 UI + MCP 서버 + 문서 |
| `refs/hangsman-claude-code-source` | [hangsman/claude-code-source](https://github.com/hangsman/claude-code-source) | 소스맵에서 복원한 원본 TypeScript (4,776 파일) |
| `refs/alanisme-claude-code-decompiled` | [alanisme/claude-code-decompiled](https://github.com/alanisme/claude-code-decompiled) | 20개 심층 분석 리포트 (다국어) |
| `refs/xtherk-open-claude-code` | [xtherk/open-claude-code](https://github.com/xtherk/open-claude-code) | 빌드 가능한 복원 소스코드 |
| `refs/leaf-kit-claude-analysis` | [leaf-kit/claude-analysis](https://github.com/leaf-kit/claude-analysis) | 소스코드 + 39개 분석문서 + 16장 튜토리얼 (Obsidian 호환) |

## 분석 문서

- [ANALYSIS.md](./ANALYSIS.md) - 전체 아키텍처 분석 (복제 수준 상세 분석)
