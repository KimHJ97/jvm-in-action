# AI Agent 개발

## 1. AI 에이전트 선택

### Claude Code 디렉토리 구조

 - `CLAUDE.md`: 프로젝트 지침 및 코딩 컨벤션
 - `.claude/skills/`: 자동 활성화되는 프롬프트 템플릿
 - `.claude/agents/`: 특화된 작업을 위한 커스텀 서브에이전트 정의
 - `.mcp.json`: DB, API, 브라우저 등 외부 도구 연결
 - `.claude/settings.json`: Hooks 및 프로젝트 설정
 - `.claude/settings.local.json`: 로컬 프로젝트 설정

<br/>

## 2. AI 에이전트 세팅

 - AI 에이전트 설치
    - `Gemini CLI`: https://geminicli.com/
    - `Claude Code`: https://claude.com/product/claude-code
    - `Codex`: https://openai.com/ko-KR/codex/
```bash
# Gemini CLI
npm install -g @google/genini-cli
gemini

# Claude Code
# npm install -> deprecated
npm install -g @claude
npm install -g @anthropic-ai/claude-code # deprecated
curl -fsSL https://claude.ai/install.sh | bash # Linux
irm https://claude.ai/install.ps1 | iex # Windows PowerShell
curl -fsSL https://claude.ai/install.cmd -o install.cmd && install.cmd && del install.cmd # Windows CMD
gemini

# OpenAI Codex
npm i -g @openai/codex
codex
```

<br/>

## 3. Claude Code 활용

### 3-1. Skills

Claude Code Skills는 Claude의 능력을 확장하는 재사용 가능한 "플러그인" 시스템입니다. __특정 작업이나 워크플로우를 한 번 정의해두면, 필요할 때마다 Claude가 자동으로 해당 지침을 불러와 사용합니다.__

 - 공식 문서: https://code.claude.com/docs/ko/skills

<br/>

#### Skill 구조 만들기

```bash
# Personal Skills (모든 프로젝트에서 사용)
mkdir -p ~/.claude/skills/skill-name

# Project Skills (특정 프로젝트에만 사용)
mkdir -p .claude/skills/skill-name
```
<br/>

#### Skill 파일 작성

Frontmatter란 마크다운 문서에서 메타데이터 역할을 하도록 만들어진 특별한 규칙이다.

 - `프론트매터 참조`
    - 모든 필드는 선택사항이며, Claude가 기술을 사용할 시기를 알 수 있도록 description만 권장된다.
    - name: Skill 식별자(Slash Command 이름)
    - description: Claude가 언제 Skill을 사용할지 이해하는 주요 트리거 메커니즘
        - Skill이 무엇을 하는지, 언제 사용하는지
    - disable-model-invocatio: 자동 호출 방지 옵션, true인 경우 사용자가 직접 Slash Command로 호출해야 함
    - user-invocable: false인 경우 '/' 자동 완성 제외 (메뉴 가시성 제어 옵션)
    - model: 이 기술이 활성화되었을 때 사용할 모델
    - allowed-tools: 이 기술이 활성화되었을 때 Claude가 권한을 요청하지 않고 사용할 도구 목록 정의
 - `본문`
    - Claude가 따를 지침을 정의
```markdown
---
name: explain-code
description: Use when explaining how code works, teaching about a codebase, or when the user asks "how does this work?"
---

When explaining code, always include:
1. **Start with an analogy**: Compare the code to something from everyday life
2. **Draw a diagram**: Use ASCII art to show the flow
3. **Walk through the code**: Explain step-by-step
4. **Highlight a gotcha**: Common mistakes or misconceptions
```
<br/>

#### 공식 Skills 저장소

 - 깃허브: https://github.com/anthropics/skills/tree/main
```bash
/plugin marketplace add anthropics/skills

/plugin install pptx@anthropic-agent-skills
/plugin install pdf@anthropic-agent-skills
```
<br/>

#### Skill 실행 흐름

 - `초기 설정 단계`: 대화 시작 시 Claude는 시스템 프롬프트에 모든 __Skills의 메타데이터만 로드__ 한다.
 - `사용자 요청`: Claude는 Skills의 __메타데이터(설명 및 요약)를 스캔__ 해서 관련 있는 것을 찾는다.
 - `Skill 호출`: 관련 Skill이 발견되면 Claude는 Skill 도구를 호출한다.
    - Skill을 호출하면 시스템은 마크다운 파일(SKILL.md)을 로드하고, 상세 지침으로 확장하고, 이를 새로운 사용자 메시지로 대화 컨텍스트에 주입한다.
    - __Skills는 별도 프로세스, 서브에이전트, 외부 도구가 아니라 주입된 지침이다.__
        - 대화 컨텍스트 수정: SKILL.md의 지침 추가
        - 실행 컨텍스트 수정: 허용된 도구, 모델 선택 변경
        - 점진적 로딩: 필요한 스크립트나 추가 파일만 로딩

<br/>

#### Skill 작성 예시

 - `Skill 메타 정보`
    - __name__
        - 이 Skill의 고유 이름
        - Spring Boot REST API를 만들 때 적용할 규칙 묶음
    - __description__
        - 이 Skill이 언제 사용되어야 하는지를 명확히 정의
        - 사용자가 Spring API 하나 만들어줘 / 엔드포인트 추가해줘
    - __allowed-tools__
        - 이 Skill을 실행할 때 사용 가능한 도구
        - Read / Write / Edit → 파일 생성·수정 가능
        - Glob / Grep → 파일 탐색 가능
        - Bash → 스크립트 실행 가능
        - LSP → Java/Spring 코드 구조 이해 가능
 - `본문`
    - Package Structure: 패키지 구조 규칙
    - Common Rules: 전역 규칙
    - Controller, DTO, Domain, Entity, Repository, Service, .. : 개별 규칙
```md
---
name: spring-api-rules
description: Define controllers, services, repositories, entities, DTOs for Spring Boot REST API. Use when user mentions API, endpoint, controller, service, repository, entity, DTO, CRUD, domain, feature, function, or REST creation.
allowed-tools: Read, Write, Edit, Glob, Grep, Bash, LSP
---

# Spring API Development Rules

Standard rules for Spring Boot REST API development in this project.

## Package Structure

com.apiece.springboot_sns_sample
├── controller/              # REST API controllers
│   └── dto/                 # Request/Response DTOs
├── domain/                  # Domain packages
│   └── user/                # User domain
│       ├── User.java        # Entity
│       ├── UserRepository.java
│       ├── UserService.java
│       └── UserException.java
└── config/                  # Configuration classes

## Common Rules

- Constructor injection using `@RequiredArgsConstructor` where possible
- No field injection (`@Autowired` on fields)
- `@ConfigurationProperties` classes should be written as records
- Use Lombok `@Getter` actively for entities and classes (except DTOs which use records)

## Controller

- Use `@RestController`
- Do not use `@RequestMapping` at class level; write full endpoint path on each method
- Return type: `ResponseEntity<T>`
- Naming: `*Controller`

## DTO

- Only controller DTOs go in `controller/dto` package
- Use Java `record`
- Request: `toEntity()` method
- Response: `from(Entity)` static factory

## Domain

Each domain is organized under `domain/{domainName}/` package:

- `{Domain}.java` - Entity
- `{Domain}Repository.java` - Data access
- `{Domain}Service.java` - Business logic
- `{Domain}Exception.java` - Domain exception

### Entity

- `protected` default constructor
- `@GeneratedValue(strategy = GenerationType.IDENTITY)`
- Associations: `FetchType.LAZY` by default
- No FK constraints in database; use `@JoinColumn` without FK constraints

### Repository

- Extends `JpaRepository<Entity, ID>`
- Follow Spring Data JPA query method naming conventions

### Service

- Use `@Transactional` only when necessary:
  - Use when multiple write operations must be in a single transaction
  - Use when Dirty Checking is needed (entity modification without explicit save)
  - Do NOT use for single Repository operations (they handle transactions automatically)
  - Do NOT use for simple read operations

## Exception Handling

- Domain exceptions: `domain/{domainName}/{Domain}Exception.java`
- Global handling with `@RestControllerAdvice`

## API Shell Script

When creating a new API, create a shell script in `src/main/resources/http/`:

- File naming: lowercase with resource name (e.g., `post.sh`, `follow.sh`)
- Include curl commands for all endpoints (POST, GET, PUT, DELETE)
- Use `BASE_URL="http://localhost:8080"` variable
- Use `-b cookies.txt` for authenticated requests
- Add descriptive echo statements before each curl command
- Start with `#!/bin/bash` shebang
```
<br/>

### 3-2. Hooks

__Hooks는 Claude Code의 특정 시점에서 자동으로 실행되는 커스텀 트리거입니다.__ "만약 X가 일어나면 Y를 실행한다"는 규칙을 설정할 수 있습니다.

Claude Code의 동작에 대해 결정론적 제어를 제공하여 LLM이 실행하도록 선택하는 것에 의존하기보다는 특정 작업이 항상 발생하도록 보장한다.

 - `공식 문서`
    - https://code.claude.com/docs/ko/hooks
    - https://code.claude.com/docs/ko/hooks-guide

<br/>

#### Hooks 사용법

`.claude/settings.json` 혹은 `.claude/settings.local.json` 클로드 설정 파일에 특정 이벤트가 발생했을 때, 사용자 정의 셸 명령을 수행하도록 할 수 있다.

`~/.claude/settings.json`은 사용자 설정, `.claude/settings.json` - 프로젝트 설정, `.claude/settings.local.json` 로컬 프로젝트 설정 (커밋되지 않음)

 - `Hooks 이벤트 정의`
    - Claude Code 설정 파일(`.claude/settings.json`)에 Hooks 이벤트를 정의한다.
    - __Hooks 이벤트 종류__
        - PreToolUse: 도구 호출 전에 실행 (차단 가능)
        - PermissionRequest: 권한 대화상자가 표시될 때 실행 (허용 또는 거부 가능)
        - PostToolUse: 도구 호출 완료 후 실행
        - UserPromptSubmit: 사용자가 프롬프트를 제출할 때 Claude가 처리하기 전에 실행
        - Notification: Claude Code가 알림을 보낼 때 실행
        - Stop: Claude Code가 응답을 마칠 때 실행
        - SubagentStop: 서브에이전트 작업이 완료될 때 실행
        - PreCompact: Claude Code가 컴팩트 작업을 실행하려고 할 때 실행
        - SessionStart: Claude Code가 새 세션을 시작하거나 기존 세션을 재개할 때 실행
        - SessionEnd: Claude Code 세션이 종료될 때 실행
    - __matcher__
        - 도구 이름을 일치시키는 패턴으로 대소문자를 구분한다. (정규식 지원)
        - PreToolUse, PermissionRequest, PostToolUse에만 적용 가능
        - `*`를 사용하면 모든 도구를 일치시킬 수 있다.
        - 빈 문자열을 사용하거나, matcher를 비워둘 수 있다.
        - 예시: Write, Edit|Write, Notebook.*
    - __hooks__: 패턴이 일치할 때 실행할 hooks의 배열
        - type: Hook 실행 유형(command - Bash 명령어, prompt - LLM 기반 평가)
        - command: type이 command인 경우 실행할 Bash 명령어
        - prompt: type이 prompt인 경우 실행할 LLM 프롬프트
        - timeout: hook이 실행되어야 하는 시간, 초과하면 특정 hook이 취소된다.
```json
{
  "hooks": {
    "EventName": [
      {
        "matcher": "ToolPattern",
        "hooks": [
          {
            "type": "command",
            "command": "your-command-here"
          }
        ]
      }
    ]
  }
}
```

<br/>

#### Hooks 사용 사례

 - `알림`:  Claude Code가 입력을 기다리거나 무언가를 실행할 권한을 기다릴 때 알림을 받는 방식을 사용자 정의한다.
 - `자동 포맷팅`: 모든 파일 편집 후 파일 포맷팅을 수행할 수 있다. (Gradle - Spotless, TypeScript - prettier, Go - gofmt)
 - `로깅`: 규정 준수 또는 디버깅을 위해 실행된 모든 명령을 추적하고 계산한다.
 - `피드백`: Claude Code가 코드베이스 규칙을 따르지 않는 코드를 생성할 때 자동화된 피드백을 제공한다.

<br/>

#### Hooks 사용 예시

 - `Hooks로 실행될 Bash 명령어 정의`
    - .claude/hooks 디렉토리의 파일을 생성 (관례)
```bash
#!/bin/bash

# PostToolUse hook: Run spotlessApply on project files after Edit/Write/MultiEdit

INPUT=$(cat)

# Extract file_path from tool_input
FILE_PATH=$(echo "$INPUT" | jq -r '.tool_input.file_path // empty')

# Check if it's a file type that Spotless handles
case "$FILE_PATH" in
  *.java|*.kts|*.json|*.yaml|*.yml|*.md|*.properties|*.xml|*.sql)
    echo "[lint hook] Running spotlessApply on: $FILE_PATH"
    cd "$CLAUDE_PROJECT_DIR" || exit 0
    ./gradlew spotlessApply -q 2>/dev/null
    ;;
esac

exit 0
```

 - `build.gradle.kts`
```groovy
spotless {
    java {
        target("src/**/*.java")
        removeUnusedImports()
        trimTrailingWhitespace()
        endWithNewline()
    }

    kotlinGradle {
        target("*.gradle.kts")
        ktlint()
        trimTrailingWhitespace()
        endWithNewline()
    }

    json {
        target("src/**/*.json", ".claude/**/*.json")
        gson()
            .indentWithSpaces(2)
            .sortByKeys()
        trimTrailingWhitespace()
        endWithNewline()
    }

    yaml {
        target("src/**/*.yaml", "src/**/*.yml")
        jackson()
            .yamlFeature("WRITE_DOC_START_MARKER", false)
            .yamlFeature("MINIMIZE_QUOTES", true)
        trimTrailingWhitespace()
        endWithNewline()
    }

    format("markdown") {
        target("*.md", ".claude/**/*.md")
        trimTrailingWhitespace()
        endWithNewline()
    }

    format("misc") {
        target("src/**/*.properties", "src/**/*.xml", "src/**/*.sql", "src/**/*.sh")
        trimTrailingWhitespace()
        endWithNewline()
    }
}
```

 - `Hooks 이벤트 정의`
    - .claude/settings.json 파일에 Hooks 이벤트 정의
    - PostToolUse: 도구 호출 완료 후 실행
    - matcher: Edit, Write, MultiEdit일 때 훅을 실행
    - hooks.type: Bash 커맨드 명령을 수행
    - hooks.command: .claude/hooks/lint.sh 스크립트를 수행
```bash
{
  "hooks": {
    "PostToolUse": [
      {
        "hooks": [
          {
            "command": "\"$CLAUDE_PROJECT_DIR\"/.claude/hooks/lint.sh",
            "timeout": 60,
            "type": "command"
          }
        ],
        "matcher": "Edit|Write|MultiEdit"
      }
    ]
  }
}
```
<br/>

### 3-3. Subagents

서브에이전트는 특정 유형의 작업을 처리하는 특화된 AI 어시스턴트입니다. __각 서브에이전트는 자신의 컨텍스트 윈도우에서 실행되며 사용자 정의 시스템 프롬프트, 특정 도구 액세스 및 독립적인 권한을 가진다.__ Claude가 서브에이전트의 설명과 일치하는 작업을 만나면 해당 서브에이전트에 위임하고, __서브에이전트는 독립적으로 작동하여 결과를 반환__ 한다.

 - 공식 문서: https://code.claude.com/docs/ko/sub-agents
 - 컨텍스트 보존 - 탐색 및 구현을 주 대화에서 분리하여 유지
 - 제약 조건 적용 - 서브에이전트가 사용할 수 있는 도구 제한
 - 구성 재사용 - 사용자 수준 서브에이전트를 통해 프로젝트 간 재사용
 - 동작 특화 - 특정 도메인을 위한 집중된 시스템 프롬프트
 - 비용 제어 - Haiku와 같은 더 빠르고 저렴한 모델로 작업 라우팅

<br/>

#### Subagents 정의

 - `명령어 사용`
```bash
/agents
```

 - `수동 파일 생성`
    - .claude/agents 디렉토리에 MarkDown 파일 생성
    - `~/.claude/agents/file.md`는 전역 사용, `.claude/agents/file.md`는 프로젝트 레벨 사용
    - __프론트매터 필드__
        - name: 고유 식별자
        - description: Claude가 이 서브에이전트에 위임해야 할 때
        - tools: 서브에이전트가 사용할 수 있는 도구. 생략하면 모든 도구 상속
            - Read, Grep, Glob  # 분석만, 수정 불가
            - Read, Grep, Glob, WebFetch, WebSearch  # 정보 수집
            - Read, Write, Edit, Bash, Glob, Grep  # 생성 및 실행
            - Read, Write, Edit, Glob, Grep, WebFetch, WebSearch  # 리서치 포함 문서화
        - disallowedTools: 거부할 도구, 상속되거나 지정된 목록에서 제거됨
        - model: 사용할 모델
        - permissionMode: 권한 모드
        - skills: 서브에이전트의 컨텍스트에 로드할 스킬
        - hooks: 이 서브에이전트로 범위가 지정된 라이프사이클 훅
```markdown
---
name: agent-name
description: When to invoke this agent
tools: Read, Write, Edit, Bash, Glob, Grep
model: sonnet  # 선택사항
permissions: auto  # 선택사항
---

# System Prompt

You are a [role description]...

[Detailed instructions, checklists, patterns]
```

<br/>

#### Subagents 사용 예시

 - `코드 리뷰어 서브에이전트 정의`
```markdown
---
name: code-reviewer
description: Reviews code for bugs, security issues, and best practices. Use when asked to review code, check code quality, or find potential issues.
tools: Read, Grep, Glob
model: sonnet
---

# Code Reviewer

You are a senior software engineer specializing in code review.

## Review Process

1. **Security Check**
   - SQL injection vulnerabilities
   - Authentication/authorization issues
   - Input validation problems
   - Sensitive data exposure

2. **Code Quality**
   - Naming conventions
   - Function complexity (< 50 lines)
   - Code duplication
   - Error handling

3. **Performance**
   - Inefficient algorithms
   - Memory leaks
   - Database query optimization
   - Caching opportunities

## Output Format

Always provide your review in this structured format:

# 🔍 CODE REVIEW REPORT

📊 **Summary:**
- **Verdict**: [NEEDS REVISION | APPROVED WITH SUGGESTIONS]
- **Blockers**: X
- **High Priority Issues**: Y
- **Medium Priority Issues**: Z

## 🚨 Blockers (Must Fix)
[List with file:line, description, actionable fix]

## ⚠️ High Priority Issues
[List with explanation and proposed refactor]

## 💡 Medium Priority Suggestions
[List improvements for clarity, naming, docs]

## ✅ Good Practices Observed
[Acknowledge well-written code]

## Weaknesses to Acknowledge

Be honest about limitations:
- Cannot execute code to verify functionality
- May miss context-specific business logic issues
- Cannot test runtime behavior
```

 - `서브에이전트 호출`
```bash
# 자동 호출: Claude가 description을 보고 자동으로 판단
$ 이 코드에 버그가 있는지 리뷰해줘

# 수동 호출: Subagent 이름 직접 언급
$ code-reviewer subagent로 이 PR을 분석해줘
```
<br/>

### 3-4. Commands

Commands (또는 Slash Commands)는 __반복되는 워크플로우를 위한 프롬프트 템플릿__ 이다. Markdown 파일로 저장해서 슬래시 메뉴를 통해 사용할 수 있습다.

 - 공식 문서: https://code.claude.com/docs/ko/slash-commands
 - MCP 목록: https://smithery.ai/servers

<br/>

#### Commands 추가

`.claude/commands/` 디렉토리의 MarkDown 파일을 생성한다.

<br/>

#### Commands 정의

 - 프론트매터
    - allowed-tools: 명령어가 사용할 수 있는 도구 목록
    - argument-hint: 슬래시 명령어에 필요한 인수. 슬래시 명령어를 자동완성할 떄 사용자에게 표시
    - description: 명령어에 대한 간단한 설명
    - model: 사용할 모델
    - disable-model-invocation: Skill 도구가 이 명령어를 호출하는 것을 방지할지 여부
    - hooks: 이 명령어의 실행 범위로 훅을 정의
```markdown
---
description: Run code review on modified files against Spring Boot project standards
allowed-tools: Read, Grep, Glob
---

## Task

code-reviewer agent를 실행하여 코드리뷰해줘.

@.claude/agents/code-reviewer.md
```
<br/>

### 3-5. MCP

MCP(Model Context Protocol)는 LLM(Claude)이 외부 도구/데이터 소스(GitHub, DB, Sentry, Notion, 사내 API 등)와 표준 방식으로 연결되도록 만든 오픈 표준이다. Claude Code는 MCP 클라이언트로서 여러 MCP 서버에 붙어서 여러 작업을 할 수 있다.

 - 공식 문서: https://code.claude.com/docs/ko/mcp
 - MCP Markeyplace:
    - Smithery: https://smithery.ai/
    - Glama.ai: https://glama.ai/chat
    - MCP.so: https://mcp.so/

#### MCP 서버 사용법

 - `원격 HTTP 서버 추가`
    - SSE (Server-Sent Events) 전송은 더 이상 사용되지 않습니다. 가능한 경우 HTTP 서버를 대신 사용 권장
```bash
# 기본 구문
claude mcp add --transport http <name> <url>

# 실제 예: Notion에 연결
claude mcp add --transport http notion https://mcp.notion.com/mcp

# Bearer 토큰을 사용한 예
claude mcp add --transport http secure-api https://api.example.com/mcp \
  --header "Authorization: Bearer your-token"


# 프로젝트 범위: .mcp.json
# 프로젝트 설정
claude mcp add --transport http paypal --scope project https://mcp.paypal.com/mcp

# 사용자 범위: ~/.claude.json
# 전역 설정
claude mcp add --transport http hubspot --scope user https://mcp.hubspot.com/anthropic
```

 - `.mcp.json`
    - Claude Code는 .mcp.json 파일의 환경 변수 확장을 지원하므로 팀이 구성을 공유하면서 머신 특정 경로 및 API 키와 같은 민감한 값에 대한 유연성을 유지할 수 있다.
```json
{
  "mcpServers": {
    "api-server": {
      "type": "http",
      "url": "${API_BASE_URL:-https://api.example.com}/mcp",
      "headers": {
        "Authorization": "Bearer ${API_KEY}"
      }
    }
  }
}
```
