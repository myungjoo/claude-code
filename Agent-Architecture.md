# Claude Code 저장소 Agent 아키텍처 분석

## 목차

1. [저장소 전체 구조](#1-저장소-전체-구조)
2. [플러그인 목록 및 역할](#2-플러그인-목록-및-역할)
3. [Agent 상세 분석](#3-agent-상세-분석)
4. [Tool 접근 제어](#4-tool-접근-제어)
5. [Agent 간 상호작용 패턴](#5-agent-간-상호작용-패턴)
6. [주요 워크플로우 흐름도](#6-주요-워크플로우-흐름도)
7. [Hook 시스템](#7-hook-시스템)
8. [핵심 설계 원칙](#8-핵심-설계-원칙)

---

## 1. 저장소 전체 구조

```
/home/user/claude-code/
├── .claude/                          # 프로젝트별 전역 설정
│   └── commands/                    # 프로젝트 전역 명령어
├── .claude-plugin/
│   └── marketplace.json             # 마켓플레이스 번들 설정
├── plugins/                         # 13개의 공식 Claude Code 플러그인
│   ├── agent-sdk-dev/
│   ├── claude-opus-4-5-migration/
│   ├── code-review/
│   ├── commit-commands/
│   ├── explanatory-output-style/
│   ├── feature-dev/
│   ├── frontend-design/
│   ├── hookify/
│   ├── learning-output-style/
│   ├── plugin-dev/
│   ├── pr-review-toolkit/
│   ├── ralph-wiggum/
│   └── security-guidance/
└── scripts/                         # 유틸리티 스크립트 (TypeScript)
```

### 표준 플러그인 내부 구조

```
plugin-name/
├── .claude-plugin/
│   └── plugin.json          # 플러그인 메타데이터
├── commands/                # 슬래시 명령어 정의 (*.md)
├── agents/                  # 자율 Agent 정의 (*.md)
├── skills/                  # 스킬 정의 (*/SKILL.md)
├── hooks/                   # 이벤트 훅 (hooks.json + 스크립트)
├── .mcp.json                # 외부 도구 설정 (선택사항)
└── README.md
```

---

## 2. 플러그인 목록 및 역할

| 플러그인 | 카테고리 | Agent 수 | 핵심 역할 |
|---|---|---|---|
| `feature-dev` | development | 3 | 기능 개발 7단계 워크플로우 |
| `pr-review-toolkit` | development | 6 | PR 다각 품질 검토 |
| `code-review` | development | 0 | 자동화 PR 검토 (명령어 기반) |
| `plugin-dev` | development | 3 | 플러그인 개발 7가지 스킬 |
| `agent-sdk-dev` | development | 2 | Agent SDK 앱 생성·검증 |
| `ralph-wiggum` | productivity | 0 | 반복 자동화 루프 (Hook 기반) |
| `commit-commands` | productivity | 0 | Git 워크플로우 명령어 |
| `hookify` | productivity | 1 | 커스텀 Hook 규칙 생성 |
| `explanatory-output-style` | learning | 0 | 교육적 설명 모드 (Hook 기반) |
| `learning-output-style` | learning | 0 | 학습 참여 모드 (Hook 기반) |
| `frontend-design` | design | 0 | 고품질 UI 설계 스킬 |
| `security-guidance` | security | 0 | 보안 패턴 경고 (Hook 기반) |
| `claude-opus-4-5-migration` | migration | 0 | 모델 마이그레이션 스킬 |

---

## 3. Agent 상세 분석

### 3.1 feature-dev 플러그인 (3개 Agent)

#### `code-explorer`
- **역할**: 기존 코드에서 기능의 진입점, 실행 흐름, 아키텍처 추적 분석
- **모델**: sonnet
- **허용 도구**: `Glob, Grep, Read, WebFetch, Bash`
- **실행 방식**: 병렬 (2-3개 동시 실행)
- **입력**: 기능 요청 설명
- **출력**: 진입점 목록, 실행 흐름, 관련 파일 목록

#### `code-architect`
- **역할**: 3가지 독립적인 아키텍처 설계 옵션 생성 및 비교
- **모델**: sonnet
- **허용 도구**: `Glob, Grep, Read`
- **실행 방식**: 병렬 (각 옵션별 독립 실행)
- **입력**: 코드베이스 분석 결과 + 기능 요구사항
- **출력**: 아키텍처 옵션 (최소 변경 / 깔끔한 설계 / 실용적 균형)

#### `code-reviewer` (feature-dev)
- **역할**: 구현 코드의 품질, DRY 원칙, 우아함, 버그 검토
- **모델**: sonnet
- **허용 도구**: `Glob, Grep, Read`
- **실행 방식**: 병렬 (3개 동시 실행, 각각 다른 관점)
- **입력**: 구현된 코드 diff
- **출력**: 이슈 목록 (0-100 신뢰도 점수 포함)

---

### 3.2 pr-review-toolkit 플러그인 (6개 Agent)

#### `code-reviewer` (pr-review-toolkit)
- **역할**: 일반 코드 품질, CLAUDE.md 준수 여부, 버그 탐지
- **모델**: opus
- **허용 도구**: `Bash(gh pr view:*, gh pr diff:*), Read, Glob, Grep`
- **출력**: 신뢰도 80점 이상 이슈만 보고

#### `pr-test-analyzer`
- **역할**: 테스트 커버리지 품질 분석 (행동 기반, 중요 간극 식별)
- **모델**: inherit (호출자 모델 상속)
- **허용 도구**: `Bash, Read, Glob, Grep`
- **출력**: 중요도 1-10 평가 + 테스트 누락 간극

#### `silent-failure-hunter`
- **역할**: 조용한 실패(silent failure), 부정확한 에러 처리 패턴 탐사
- **모델**: inherit
- **허용 도구**: `Read, Glob, Grep`
- **출력**: 심각도별 분류된 에러 처리 이슈

#### `comment-analyzer`
- **역할**: 코드 주석과 실제 동작 간 불일치, 문서 완성도 검증
- **모델**: inherit
- **허용 도구**: `Read, Glob, Grep`
- **출력**: 부정확한 주석 목록 + 문서화 누락 항목

#### `type-design-analyzer`
- **역할**: 타입 설계 평가 (캡슐화, 불변식, 유용성, 강제성)
- **모델**: inherit
- **허용 도구**: `Read, Glob, Grep`
- **출력**: 4차원 각 1-10 점수 + 개선 제안

#### `code-simplifier`
- **역할**: 불필요한 복잡도 감소, 코드 간결화 기회 식별
- **모델**: inherit
- **허용 도구**: `Read, Glob, Grep`
- **출력**: 간결화 가능 구간 + 개선 제안

---

### 3.3 plugin-dev 플러그인 (3개 Agent)

#### `agent-creator`
- **역할**: 요구사항에 맞는 Agent 정의 파일(*.md) 생성
- **모델**: sonnet
- **허용 도구**: `Write, Read`
- **출력**: 완성된 Agent .md 파일

#### `plugin-validator`
- **역할**: 생성된 플러그인의 구조, 메타데이터, 컴포넌트 유효성 검증
- **모델**: sonnet
- **허용 도구**: `Read, Grep, Glob, Bash`
- **출력**: 검증 보고서 (통과/실패 항목 목록)

#### `skill-reviewer`
- **역할**: 생성된 Skill 문서의 트리거 구문, Progressive Disclosure 준수 검토
- **모델**: sonnet
- **허용 도구**: `Read, Glob`
- **출력**: 스킬 품질 평가 및 개선 제안

---

### 3.4 agent-sdk-dev 플러그인 (2개 Agent)

#### `agent-sdk-verifier-ts`
- **역할**: TypeScript Agent SDK 앱 검증 (설치, 설정, 타입 안전성, 빌드 가능성)
- **모델**: sonnet
- **허용 도구**: `Read, Glob, Bash`
- **출력**: TypeScript SDK 검증 보고서

#### `agent-sdk-verifier-py`
- **역할**: Python Agent SDK 앱 검증 (설치, 설정, 사용 패턴, 실행 가능성)
- **모델**: sonnet
- **허용 도구**: `Read, Glob, Bash`
- **출력**: Python SDK 검증 보고서

---

### 3.5 hookify 플러그인 (1개 Agent)

#### `conversation-analyzer`
- **역할**: 현재 세션 대화에서 문제 행동 패턴 식별 및 Hook 규칙 제안
- **모델**: inherit
- **허용 도구**: `Read`
- **입력**: 세션 전사(transcript)
- **출력**: YAML 형식 Hook 규칙 제안 목록

---

## 4. Tool 접근 제어

Agent별 최소 권한 원칙(Principle of Least Privilege)이 적용됩니다.

### 도구별 사용 Agent 매트릭스

| Tool | 읽기 전용 Agent | 생성 Agent | 검증 Agent | 실행 Agent |
|---|---|---|---|---|
| `Read` | 전체 | agent-creator | plugin-validator | - |
| `Glob` | 전체 | - | plugin-validator | - |
| `Grep` | 전체 | - | plugin-validator | - |
| `Write` | - | agent-creator | - | - |
| `Bash` | - | - | plugin-validator | verifier-ts/py |
| `Bash(gh pr view:*)` | - | - | code-reviewer(pr) | - |
| `WebFetch` | code-explorer | - | - | - |

### Tool 제한 예시

```yaml
# 탐색 전용 Agent (쓰기 불가)
allowed-tools: ["Glob", "Grep", "Read", "WebFetch"]

# 생성 Agent (최소 권한)
allowed-tools: ["Write", "Read"]

# PR 검토 Agent (gh CLI 특정 명령만 허용)
allowed-tools: ["Bash(gh pr view:*)", "Bash(gh pr diff:*)", "Read", "Glob", "Grep"]

# SDK 검증 Agent (빌드 실행 허용)
allowed-tools: ["Read", "Glob", "Bash"]
```

---

## 5. Agent 간 상호작용 패턴

### 패턴 1: Fan-out 병렬 실행

동일한 Agent를 여러 개 동시 실행하여 다양한 관점을 확보합니다.

```
메인 명령어
    ├─→ Agent 인스턴스 1 (관점 A)
    ├─→ Agent 인스턴스 2 (관점 B)  ← 동시 실행
    └─→ Agent 인스턴스 3 (관점 C)
          ↓
    결과 집계 및 통합
```

**사용 예**: `feature-dev`의 Phase 2, 4, 6 (각각 2-3개 병렬 실행)

---

### 패턴 2: 조건부 디스패치

변경 내용 유형에 따라 적합한 Agent만 선택 실행합니다.

```
pr-review-toolkit:review-pr [all|tests|types|comments|errors|simplify]
    ├─→ [all] 전체 6개 Agent 실행
    ├─→ [tests] pr-test-analyzer 만 실행
    ├─→ [types] type-design-analyzer 만 실행
    ├─→ [comments] comment-analyzer 만 실행
    ├─→ [errors] silent-failure-hunter 만 실행
    └─→ [simplify] code-simplifier 만 실행
```

---

### 패턴 3: 파이프라인 직렬 실행

이전 Agent의 출력이 다음 Agent의 입력이 됩니다.

```
code-review 명령어
    ↓
[1] 검토 필요 여부 판단 (Haiku)
    ↓ (필요한 경우만)
[2] CLAUDE.md 가이드라인 수집 (Haiku)
    ↓
[3] PR 변경 요약 (Sonnet)
    ↓
[4] 4개 병렬 검토 Agent
    │  ├─→ CLAUDE.md 준수 1 (Sonnet)
    │  ├─→ CLAUDE.md 준수 2 (Sonnet)
    │  ├─→ 명백한 버그 (Opus)
    │  └─→ git blame 분석 (Opus)
    ↓
[5] 이슈별 검증 Agent (병렬, 신뢰도 계산)
    ↓
[6] 80점 미만 필터링
    ↓
결과 출력 또는 PR 댓글 게시
```

---

### 패턴 4: 메타 에이전트 (자기참조 구조)

Agent가 다른 Agent를 생성하고 검증합니다.

```
plugin-dev:create-plugin 명령어
    ↓
agent-creator Agent  ← Agent를 생성하는 Agent
    ↓ (생성된 Agent .md 파일)
plugin-validator Agent  ← Agent를 검증하는 Agent
    ↓
skill-reviewer Agent  ← Skill을 검토하는 Agent
    ↓
검증 완료
```

---

### 패턴 5: Hook 기반 자기반복 루프

Stop 훅이 종료 시도를 가로채 같은 작업을 반복합니다.

```
사용자: /ralph-loop "<prompt>" --max-iterations 20
    ↓
Claude: 작업 실행
    ↓
[종료 시도]
    ↓ (Stop 훅 가로채기)
stop-hook.sh: 반복 횟수 확인
    ├─→ [완료 조건 충족] → 정상 종료
    └─→ [미완료] → 동일 프롬프트 재제출 → Claude 재실행
```

---

## 6. 주요 워크플로우 흐름도

### 6.1 feature-dev 전체 흐름

```
사용자: /feature-dev "기능 설명"
         │
         ▼
┌─────────────────────┐
│ Phase 1: Discovery  │  기능 요구사항 명확화
└─────────────────────┘
         │
         ▼
┌──────────────────────────────────────────┐
│ Phase 2: Codebase Exploration            │
│  ┌─────────────┐ ┌─────────────┐         │
│  │code-explorer│ │code-explorer│  (병렬)  │
│  │  인스턴스 1  │ │  인스턴스 2  │         │
│  └─────────────┘ └─────────────┘         │
└──────────────────────────────────────────┘
         │
         ▼
┌─────────────────────────────┐
│ Phase 3: Clarifying Questions│  모든 모호함 제거
└─────────────────────────────┘
         │ (사용자 응답 대기)
         ▼
┌──────────────────────────────────────────────────────┐
│ Phase 4: Architecture Design                         │
│  ┌──────────────┐ ┌──────────────┐ ┌──────────────┐  │
│  │code-architect│ │code-architect│ │code-architect│  │
│  │  최소 변경   │ │ 깔끔한 설계  │ │ 실용적 균형  │  │
│  └──────────────┘ └──────────────┘ └──────────────┘  │
└──────────────────────────────────────────────────────┘
         │ (사용자 아키텍처 선택)
         ▼
┌──────────────────────┐
│ Phase 5: Implementation│  파일 수정 및 구현
└──────────────────────┘
         │ (사용자 명시적 승인)
         ▼
┌──────────────────────────────────────────────────┐
│ Phase 6: Quality Review                          │
│  ┌─────────────┐ ┌─────────────┐ ┌─────────────┐ │
│  │code-reviewer│ │code-reviewer│ │code-reviewer│ │
│  │ 코드 품질   │ │ DRY/우아함  │ │  버그/정확성│ │
│  └─────────────┘ └─────────────┘ └─────────────┘ │
└──────────────────────────────────────────────────┘
         │
         ▼
┌───────────────────┐
│ Phase 7: Summary  │  완료 문서화
└───────────────────┘
```

---

### 6.2 code-review 명령어 흐름

```
/code-review [--comment]
         │
         ▼
┌─────────────────────────────────────────┐
│ 검토 필요 여부 판단                      │
│ (closed / draft / 자동화 PR → 스킵)      │
└─────────────────────────────────────────┘
         │
         ▼
┌─────────────────────────────────────────┐
│ CLAUDE.md 가이드라인 수집               │
└─────────────────────────────────────────┘
         │
         ▼
┌─────────────────────────────────────────┐
│ PR 변경 요약 생성                        │
└─────────────────────────────────────────┘
         │
         ▼
┌───────────────────────────────────────────────────────────┐
│ 4개 병렬 검토 Agent                                        │
│  ┌───────────────┐ ┌───────────────┐                       │
│  │CLAUDE.md 준수 │ │CLAUDE.md 준수 │  ← Sonnet 모델        │
│  │   Agent 1     │ │   Agent 2     │                       │
│  └───────────────┘ └───────────────┘                       │
│  ┌───────────────┐ ┌───────────────┐                       │
│  │ 명백한 버그   │ │ git blame     │  ← Opus 모델          │
│  │   탐지        │ │ 히스토리 분석 │                       │
│  └───────────────┘ └───────────────┘                       │
└───────────────────────────────────────────────────────────┘
         │
         ▼
┌─────────────────────────────────────────┐
│ 이슈별 신뢰도 검증 (병렬)               │
│ 0-100 점수 산정                          │
└─────────────────────────────────────────┘
         │
         ▼
┌─────────────────────────────────────────┐
│ 80점 미만 필터링                         │
└─────────────────────────────────────────┘
         │
         ├─→ [--comment] PR 댓글 게시
         └─→ [기본] 콘솔 출력
```

---

## 7. Hook 시스템

Hook은 Claude Code의 이벤트에 반응하여 자동으로 실행되는 스크립트입니다.

### 7.1 Hook 이벤트 타입

| 이벤트 | 설명 | 사용 플러그인 |
|---|---|---|
| `PreToolUse` | 도구 실행 전 검증 | security-guidance, hookify |
| `PostToolUse` | 도구 실행 후 처리 | hookify |
| `Stop` | 세션 종료 시도 시 | ralph-wiggum |
| `SessionStart` | 세션 시작 시 | explanatory-output-style, learning-output-style |
| `SessionEnd` | 세션 종료 시 | hookify |
| `UserPromptSubmit` | 사용자 입력 제출 시 | hookify |
| `SubagentStop` | 서브에이전트 종료 시 | hookify |

### 7.2 Hook 설정 형식

```json
{
  "description": "훅 설명",
  "hooks": {
    "PreToolUse": [
      {
        "hooks": [
          {
            "type": "command",
            "command": "${CLAUDE_PLUGIN_ROOT}/scripts/security-check.py"
          }
        ],
        "matcher": "Edit|Write|Bash"
      }
    ],
    "Stop": [
      {
        "hooks": [
          {
            "type": "command",
            "command": "${CLAUDE_PLUGIN_ROOT}/hooks/stop-hook.sh"
          }
        ]
      }
    ]
  }
}
```

### 7.3 플러그인별 Hook 활용

#### ralph-wiggum (반복 루프)
- **이벤트**: `Stop`
- **동작**: 종료 시도를 가로채 반복 횟수 확인 후 재실행

#### security-guidance (보안 경고)
- **이벤트**: `PreToolUse`
- **모니터링 패턴**:
  - 명령어 주입 (Command Injection)
  - Cross-Site Scripting (XSS)
  - `eval()` 사용
  - 위험한 innerHTML
  - pickle 역직렬화
  - `os.system()` 호출

#### explanatory-output-style / learning-output-style (교육 모드)
- **이벤트**: `SessionStart`
- **동작**: 세션 시작 시 교육적 지침을 시스템 프롬프트에 주입

#### hookify (동적 규칙 관리)
- **이벤트**: `PreToolUse`, `PostToolUse`, `UserPromptSubmit` 등
- **동작**: YAML 파일로 정의된 정규식 기반 규칙 동적 적용

### 7.4 hookify 규칙 형식

```yaml
---
name: rule-name
enabled: true
event: bash|file|stop|prompt|all
pattern: regex_pattern
action: warn|block
conditions:
  - pattern: additional_regex
    field: tool|input|output
---

규칙 설명 (마크다운 형식)
위반 시 표시될 경고 메시지
```

---

## 8. 핵심 설계 원칙

### 8.1 최소 권한 원칙 (Principle of Least Privilege)

각 Agent는 역할에 필요한 최소한의 도구만 허용받습니다.

- **읽기 전용 Agent**: `Glob, Grep, Read` 만 허용 (탐색·분석)
- **생성 Agent**: `Write, Read` 만 허용 (파일 생성)
- **검증 Agent**: `Read, Grep, Glob, Bash` 허용 (검사 실행)
- **실행 Agent**: `Bash` 포함 허용 (빌드·테스트 실행)

### 8.2 신뢰도 기반 필터링

거짓양성(False Positive)을 줄이기 위해 신뢰도 점수로 이슈를 필터링합니다.

- `code-review`: 80점 이상만 보고
- `pr-review-toolkit`: 80점 이상만 보고
- `pr-test-analyzer`: 중요도 1-10 점수 기반 우선순위화
- `type-design-analyzer`: 4차원(캡슐화/불변식/유용성/강제성) 각 1-10 평가

### 8.3 점진적 공개 (Progressive Disclosure)

사용자에게 필요한 정보만 단계적으로 제공하여 컨텍스트 효율성을 유지합니다.

1. **메타데이터**: 짧은 설명 + 강력한 트리거 구문
2. **코어 문서**: 필수 API 참조 (~1,500-2,000 단어)
3. **참고자료**: 상세 가이드 및 예제

### 8.4 명시적 사용자 동의

중요한 결정 이전에 사용자 확인을 요구합니다.

- `feature-dev`: Phase 4 아키텍처 선택, Phase 5 구현 승인
- `ralph-wiggum`: 최대 반복 횟수 (`--max-iterations`) 사전 설정
- `code-review`: `--comment` 플래그로 PR 게시 명시적 선택

### 8.5 다단계 워크플로우 추적

복잡한 워크플로우는 `TodoWrite`로 진행 상태를 실시간 추적합니다.

```
Phase 1: Discovery          [completed]
Phase 2: Codebase Exploration [in_progress]
Phase 3: Clarifying Questions [pending]
Phase 4: Architecture Design  [pending]
Phase 5: Implementation       [pending]
Phase 6: Quality Review       [pending]
Phase 7: Summary              [pending]
```

---

## 부록: 전체 Agent 목록

| Agent | 플러그인 | 모델 | 실행 방식 | 주요 허용 도구 |
|---|---|---|---|---|
| `code-explorer` | feature-dev | sonnet | 병렬 (2-3개) | Glob, Grep, Read, WebFetch, Bash |
| `code-architect` | feature-dev | sonnet | 병렬 (3개) | Glob, Grep, Read |
| `code-reviewer` | feature-dev | sonnet | 병렬 (3개) | Glob, Grep, Read |
| `code-reviewer` | pr-review-toolkit | opus | 단일 | Bash(gh), Read, Glob, Grep |
| `pr-test-analyzer` | pr-review-toolkit | inherit | 단일/병렬 | Bash, Read, Glob, Grep |
| `silent-failure-hunter` | pr-review-toolkit | inherit | 단일 | Read, Glob, Grep |
| `comment-analyzer` | pr-review-toolkit | inherit | 단일 | Read, Glob, Grep |
| `type-design-analyzer` | pr-review-toolkit | inherit | 단일 | Read, Glob, Grep |
| `code-simplifier` | pr-review-toolkit | inherit | 단일 | Read, Glob, Grep |
| `agent-creator` | plugin-dev | sonnet | 단일 | Write, Read |
| `plugin-validator` | plugin-dev | sonnet | 단일 | Read, Grep, Glob, Bash |
| `skill-reviewer` | plugin-dev | sonnet | 단일 | Read, Glob |
| `agent-sdk-verifier-ts` | agent-sdk-dev | sonnet | 단일 | Read, Glob, Bash |
| `agent-sdk-verifier-py` | agent-sdk-dev | sonnet | 단일 | Read, Glob, Bash |
| `conversation-analyzer` | hookify | inherit | 단일 | Read |
