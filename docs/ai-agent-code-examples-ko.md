# Zed AI Agent 코드 예제 및 사용 패턴 (Code Examples & Usage Patterns)

## 목차
1. [기본 사용법](#기본-사용법)
2. [도구 구현 예제](#도구-구현-예제)
3. [커스텀 프로파일 설정](#커스텀-프로파일-설정)
4. [외부 에이전트 설정](#외부-에이전트-설정)
5. [MCP 서버 통합](#mcp-서버-통합)
6. [고급 사용 패턴](#고급-사용-패턴)

---

## 기본 사용법

### 1. 새 에이전트 스레드 시작

**키보드 단축키:**
```json
{
  "bindings": {
    "cmd-shift-a": "agent::NewThread",
    "cmd-alt-g": ["agent::NewExternalAgentThread", { "agent": "gemini" }],
    "cmd-alt-c": ["agent::NewExternalAgentThread", { "agent": "claude_code" }]
  }
}
```

**사용 예시:**
1. `Cmd+Shift+A` - 네이티브 Zed 에이전트로 새 스레드 시작
2. 또는 상태 표시줄의 ✨ (sparkles) 아이콘 클릭
3. 메시지 입력 후 `Enter`로 전송

### 2. 컨텍스트 추가하기

**멘션 사용:**
```
@파일명 - 파일 내용 포함
@디렉터리/ - 디렉터리 구조 포함
@심볼명 - 특정 함수/클래스 포함
@선택영역 - 현재 선택된 코드
@스레드 - 이전 대화 참조
```

**실제 사용 예:**
```
@src/main.rs의 main 함수를 분석하고, 
@tests/ 디렉터리의 테스트들을 참고하여 
새로운 테스트 케이스를 추가해주세요.
```

### 3. 선택 영역을 컨텍스트로 추가

1. 코드 선택
2. `Cmd+Shift+A` 또는 컨텍스트 메뉴에서 "Add Selection to Thread"
3. 에이전트 패널에서 자동으로 선택 영역이 멘션으로 추가됨

---

## 도구 구현 예제

### 커스텀 도구 만들기

```rust
use crate::AgentTool;
use anyhow::Result;
use gpui::AsyncApp;
use language_model::LanguageModelToolSchemaFormat;
use schemars::{JsonSchema, schema_for};
use serde::{Deserialize, Serialize};

/// 커스텀 도구 입력 스키마
#[derive(Debug, Deserialize, Serialize, JsonSchema)]
pub struct MyToolInput {
    /// 처리할 파일 경로
    pub path: String,
    /// 적용할 변환 타입
    pub transform_type: TransformType,
}

#[derive(Debug, Deserialize, Serialize, JsonSchema)]
pub enum TransformType {
    #[serde(rename = "uppercase")]
    Uppercase,
    #[serde(rename = "lowercase")]
    Lowercase,
    #[serde(rename = "camelcase")]
    CamelCase,
}

/// 커스텀 도구 구현
pub struct MyCustomTool;

impl AgentTool for MyCustomTool {
    fn name() -> &'static str {
        "my_custom_tool"
    }

    fn description() -> &'static str {
        "커스텀 텍스트 변환 도구. 파일의 내용을 다양한 형식으로 변환합니다."
    }

    fn input_schema(format: LanguageModelToolSchemaFormat) -> schemars::Schema {
        match format {
            LanguageModelToolSchemaFormat::JsonSchema => {
                schema_for!(MyToolInput).schema.into()
            }
        }
    }

    fn supports_provider(provider: &language_model::LanguageModelProviderId) -> bool {
        // 모든 프로바이더에서 지원
        true
    }

    async fn run(
        input: serde_json::Value,
        project: &gpui::Entity<project::Project>,
        worktree_id: Option<project::WorktreeId>,
        cx: &AsyncApp,
    ) -> Result<String> {
        let input: MyToolInput = serde_json::from_value(input)?;
        
        // 파일 읽기
        let content = cx.update(|cx| {
            // 파일 내용 읽기 로직
            // ... 
            Ok::<String, anyhow::Error>("file content".to_string())
        })??;

        // 변환 적용
        let transformed = match input.transform_type {
            TransformType::Uppercase => content.to_uppercase(),
            TransformType::Lowercase => content.to_lowercase(),
            TransformType::CamelCase => to_camel_case(&content),
        };

        Ok(format!(
            "변환 완료: {} → {}\n결과:\n{}",
            input.path,
            match input.transform_type {
                TransformType::Uppercase => "대문자",
                TransformType::Lowercase => "소문자",
                TransformType::CamelCase => "카멜케이스",
            },
            transformed
        ))
    }
}

fn to_camel_case(s: &str) -> String {
    // 카멜케이스 변환 로직
    s.split('_')
        .enumerate()
        .map(|(i, word)| {
            if i == 0 {
                word.to_lowercase()
            } else {
                let mut chars = word.chars();
                match chars.next() {
                    None => String::new(),
                    Some(first) => first.to_uppercase().collect::<String>() + &chars.as_str().to_lowercase(),
                }
            }
        })
        .collect()
}
```

### 도구를 시스템에 등록

`crates/agent/src/tools.rs`에 추가:

```rust
mod my_custom_tool;
pub use my_custom_tool::*;

tools! {
    CopyPathTool,
    CreateDirectoryTool,
    // ... 기존 도구들
    MyCustomTool,  // 새 도구 추가
}
```

---

## 커스텀 프로파일 설정

### settings.json 예제

```json
{
  "agent": {
    "version": "2",
    "default_model": {
      "provider": "zed",
      "name": "claude-sonnet-4-20250514"
    },
    "profiles": {
      // 읽기 전용 분석 프로파일
      "read-only-analyzer": {
        "name": "Read-Only Analyzer",
        "description": "코드 분석용 읽기 전용 프로파일",
        "tools": [
          "read_file",
          "grep",
          "find_path",
          "list_directory",
          "diagnostics",
          "web_search",
          "fetch"
        ]
      },
      
      // 파일 편집 전용 프로파일
      "file-editor": {
        "name": "File Editor",
        "description": "파일 편집에 집중한 프로파일",
        "tools": [
          "read_file",
          "edit_file",
          "create_file",
          "grep",
          "diagnostics"
        ]
      },
      
      // 프로젝트 관리 프로파일
      "project-manager": {
        "name": "Project Manager",
        "description": "프로젝트 구조 관리용 프로파일",
        "tools": [
          "list_directory",
          "create_directory",
          "move_path",
          "copy_path",
          "delete_path",
          "find_path"
        ]
      },
      
      // 풀스택 개발 프로파일
      "fullstack-dev": {
        "name": "Full Stack Developer",
        "description": "모든 도구를 사용하는 개발 프로파일",
        "tools": [
          "read_file",
          "edit_file",
          "grep",
          "list_directory",
          "create_directory",
          "move_path",
          "copy_path",
          "delete_path",
          "find_path",
          "diagnostics",
          "terminal",
          "web_search",
          "fetch",
          "now",
          "open"
        ]
      }
    },
    
    // 도구 자동 승인 설정
    "always_allow_tool_actions": false,
    
    // 알림 설정
    "notify_when_agent_waiting": true,
    "play_sound_when_agent_done": true
  }
}
```

### UI를 통한 프로파일 관리

1. **프로파일 생성:**
   - Agent Panel 열기 (`Cmd+Shift+A`)
   - 프로파일 선택기 클릭
   - "Configure" → "Add New Profile"

2. **기존 프로파일 포크:**
   - 프로파일 선택기에서 원하는 프로파일 선택
   - "Fork Profile" 클릭
   - 이름 변경 및 도구 수정

3. **내장 프로파일 오버라이드:**
   - "Write" 프로파일 선택
   - "Configure Tools" 클릭
   - 원하는 도구만 선택
   - 저장하면 settings.json에 자동으로 추가됨

---

## 외부 에이전트 설정

### Gemini CLI 설정

```json
{
  "agent_servers": {
    "gemini": {
      // false로 설정하면 시스템에 설치된 버전 사용
      "ignore_system_version": false,
      
      // 환경 변수 설정
      "env": {
        "GEMINI_API_KEY": "your-api-key-here"
      },
      
      // 커스텀 경로 (선택사항)
      "command": "/custom/path/to/gemini",
      "args": ["--custom-arg"]
    }
  }
}
```

### Claude Code 설정

```json
{
  "agent_servers": {
    "claude": {
      "env": {
        // 커스텀 Claude Code 실행 파일
        "CLAUDE_CODE_EXECUTABLE": "/path/to/alternate-claude-code-executable"
      }
    }
  }
}
```

### Codex 설정

```json
{
  "agent_servers": {
    "codex": {
      "env": {
        // API 키 설정
        "CODEX_API_KEY": "your-codex-api-key",
        // 또는 OpenAI API 키
        "OPENAI_API_KEY": "your-openai-api-key"
      }
    }
  }
}
```

### 완전 커스텀 ACP 에이전트

```json
{
  "agent_servers": {
    "my-custom-agent": {
      "command": "node",
      "args": [
        "/home/user/projects/my-agent/index.js",
        "--acp",
        "--verbose"
      ],
      "env": {
        "AGENT_CONFIG_PATH": "/home/user/.config/my-agent",
        "API_KEY": "your-api-key"
      }
    }
  }
}
```

---

## MCP 서버 통합

### MCP 서버 설정 예제

```json
{
  "context_servers": {
    // 파일 시스템 MCP 서버
    "filesystem": {
      "command": "npx",
      "args": [
        "-y",
        "@modelcontextprotocol/server-filesystem",
        "/path/to/allowed/directory"
      ]
    },
    
    // Git MCP 서버
    "git": {
      "command": "npx",
      "args": [
        "-y",
        "@modelcontextprotocol/server-git"
      ]
    },
    
    // 데이터베이스 MCP 서버
    "postgres": {
      "command": "npx",
      "args": [
        "-y",
        "@modelcontextprotocol/server-postgres"
      ],
      "env": {
        "DATABASE_URL": "postgresql://user:pass@localhost/db"
      }
    },
    
    // 커스텀 MCP 서버
    "my-tools": {
      "command": "python",
      "args": [
        "/path/to/my_mcp_server.py"
      ],
      "env": {
        "API_KEY": "your-api-key"
      }
    }
  }
}
```

### MCP 서버 확장 설치

1. **Zed Extensions를 통한 설치:**
   ```
   Cmd+Shift+P → "zed: extensions"
   → "Agent Servers" 필터
   → 원하는 MCP 서버 설치
   ```

2. **사용 가능한 MCP 서버 확인:**
   - Agent Panel에서 설정 아이콘 클릭
   - "Context Servers" 섹션 확인

---

## 고급 사용 패턴

### 1. 규칙 파일 활용

**프로젝트 루트에 .rules 파일 생성:**

```markdown
# Project Rules

## Code Style
- Use 4 spaces for indentation
- Maximum line length: 100 characters
- Use descriptive variable names

## Architecture
- Follow the repository structure in /docs/architecture.md
- Keep business logic in /src/core/
- Put UI components in /src/ui/

## Testing
- Write unit tests for all new features
- Aim for >80% code coverage
- Use descriptive test names

## Documentation
- Add JSDoc comments for all public APIs
- Update README.md when adding new features
- Keep CHANGELOG.md up to date

## Git Workflow
- Use conventional commits
- Create feature branches from main
- Squash commits before merging
```

**CLAUDE.md 파일 (Claude Code 전용):**

```markdown
# Claude Code Instructions

## Project Context
This is a Rust-based code editor with AI capabilities.

## Key Files
- /crates/agent/ - Core agent logic
- /crates/agent_ui/ - UI components
- /crates/agent_servers/ - External agent integration

## Development Guidelines
1. Follow Rust best practices
2. Use the GPUI framework conventions
3. Write tests for all agent tools
4. Document public APIs

## Common Tasks
- Adding new tools: See /crates/agent/src/tools/
- Modifying UI: See /crates/agent_ui/src/
- Testing: Run `cargo test -p agent`
```

### 2. 체크포인트 활용

```
사용자: @src/main.rs 파일을 리팩토링해주세요.

[에이전트가 변경 수행]

# 변경사항이 마음에 들지 않으면:
1. "Restore Checkpoint" 버튼 클릭
2. 또는 키보드: Cmd+Z (macros)

# 부분적으로 되돌리려면:
1. "Review Changes" 클릭
2. 개별 변경사항 reject
```

### 3. 멀티 스레드 워크플로우

```json
// 키 바인딩 설정
{
  "bindings": {
    // 여러 에이전트를 동시에 사용
    "cmd-1": ["workspace::ActivatePane", { "index": 0 }],
    "cmd-2": ["workspace::ActivatePane", { "index": 1 }],
    "cmd-shift-1": "agent::NewThread",
    "cmd-shift-2": ["agent::NewExternalAgentThread", { "agent": "gemini" }]
  }
}
```

**사용 시나리오:**
1. Zed 네이티브 에이전트로 코드 작성 (Pane 1)
2. Gemini CLI로 코드 리뷰 (Pane 2)
3. 두 결과를 비교하여 최종 결정

### 4. 토큰 최적화 전략

```
# 효율적인 컨텍스트 제공:

나쁜 예:
"@전체_프로젝트/ 를 분석하고 버그를 찾아주세요"
→ 토큰 사용량 과다

좋은 예:
"@src/auth/login.rs 파일의 인증 로직에 버그가 있을 수 있습니다.
@tests/auth_tests.rs 의 failing 테스트를 참고하여 수정해주세요."
→ 필요한 컨텍스트만 제공
```

### 5. 스트림 처리 패턴

```rust
// Thread에서 스트리밍 응답 처리
cx.spawn(async move |thread, mut cx| {
    let mut stream = model.stream_completion(request).await?;
    
    while let Some(event) = stream.next().await {
        match event? {
            LanguageModelCompletionEvent::Text(text) => {
                // UI 업데이트
                thread.update(&mut cx, |thread, cx| {
                    thread.append_text(text, cx);
                })?;
            }
            LanguageModelCompletionEvent::ToolUse(tool_use) => {
                // 도구 실행
                let result = execute_tool(&tool_use).await?;
                thread.update(&mut cx, |thread, cx| {
                    thread.add_tool_result(result, cx);
                })?;
            }
            LanguageModelCompletionEvent::Stop(_) => break,
        }
    }
    
    Ok(())
})
```

### 6. 에러 처리 및 재시도

```rust
// RetryStrategy 활용
pub async fn execute_with_retry<T, F, Fut>(
    mut f: F,
    strategy: RetryStrategy,
) -> Result<T>
where
    F: FnMut() -> Fut,
    Fut: Future<Output = Result<T>>,
{
    match strategy {
        RetryStrategy::ExponentialBackoff { initial_delay, max_attempts } => {
            let mut delay = initial_delay;
            let mut attempts = 0;
            
            loop {
                attempts += 1;
                match f().await {
                    Ok(result) => return Ok(result),
                    Err(e) if attempts >= max_attempts => return Err(e),
                    Err(_) => {
                        smol::Timer::after(delay).await;
                        delay *= 2;
                    }
                }
            }
        }
        RetryStrategy::Fixed { delay, max_attempts } => {
            // Fixed delay retry logic
            // ...
        }
    }
}
```

### 7. 커스텀 템플릿 활용

```rust
// 커스텀 프롬프트 템플릿
use handlebars::Handlebars;

pub struct CustomPromptTemplate {
    pub task: String,
    pub context: Vec<String>,
    pub constraints: Vec<String>,
}

impl Template for CustomPromptTemplate {
    const TEMPLATE_NAME: &'static str = "custom_prompt.hbs";
}

// templates/custom_prompt.hbs
```

```handlebars
You are an expert software engineer.

Task: {{task}}

Context:
{{#each context}}
- {{this}}
{{/each}}

Constraints:
{{#each constraints}}
- {{this}}
{{/each}}

Please provide a detailed solution following these guidelines.
```

### 8. 진단 정보 활용

```
사용자: 빌드 오류를 수정해주세요.

에이전트: [diagnostics 도구 자동 호출]
→ 프로젝트의 모든 오류 및 경고 수집
→ 관련 파일 읽기
→ 오류 수정
→ diagnostics 재실행하여 검증
```

### 9. 웹 리서치 통합

```
사용자: Rust의 async/await에 대한 최신 베스트 프랙티스를 
적용하여 @src/async_handler.rs 를 리팩토링해주세요.

에이전트 워크플로우:
1. [web_search] "Rust async await best practices 2024"
2. [fetch] 관련 문서 URL 가져오기
3. [read_file] src/async_handler.rs 읽기
4. 최신 패턴 적용
5. [edit_file] 변경사항 적용
```

### 10. 터미널 명령 실행

```
사용자: 프로젝트의 모든 테스트를 실행하고 
실패한 테스트가 있으면 수정해주세요.

에이전트 워크플로우:
1. [terminal] "cargo test"
2. 실패한 테스트 분석
3. [read_file] 테스트 파일 읽기
4. [read_file] 구현 파일 읽기
5. [edit_file] 버그 수정
6. [terminal] "cargo test" 재실행
7. [diagnostics] 검증
```

---

## 실전 사용 사례

### 사례 1: 새 기능 추가

```
사용자: 사용자 인증 기능을 추가해주세요.
- JWT 토큰 기반 인증
- 로그인/로그아웃 엔드포인트
- 미들웨어로 인증 검증
- 테스트 포함

[에이전트 프로파일: Write]
[컨텍스트: @src/api/, @docs/api-spec.md]

에이전트 작업:
1. [create_directory] src/auth
2. [create_file] src/auth/jwt.rs
3. [create_file] src/auth/middleware.rs
4. [edit_file] src/api/routes.rs (라우트 추가)
5. [create_file] tests/auth_tests.rs
6. [terminal] cargo test
7. [diagnostics] 오류 확인 및 수정
```

### 사례 2: 코드베이스 이해

```
사용자: 이 프로젝트의 에이전트 시스템 아키텍처를 설명해주세요.

[에이전트 프로파일: Ask]

에이전트 작업:
1. [list_directory] crates/
2. [read_file] crates/agent/Cargo.toml
3. [read_file] crates/agent/src/agent.rs (일부)
4. [grep] "pub struct" in crates/agent/src/
5. [find_path] "**/README.md"
6. 분석 결과 제공
```

### 사례 3: 리팩토링

```
사용자: @src/legacy_code.rs를 현대적인 Rust 패턴으로 
리팩토링해주세요. 기능은 유지하되 가독성과 
유지보수성을 개선해주세요.

[에이전트 프로파일: Write]
[체크포인트 자동 생성]

에이전트 작업:
1. [read_file] src/legacy_code.rs
2. [diagnostics] 현재 경고/오류 확인
3. [web_search] "Rust refactoring best practices"
4. [edit_file] 리팩토링 수행
   - 불필요한 클론 제거
   - 에러 처리 개선
   - 타입 안정성 향상
5. [terminal] cargo clippy
6. [terminal] cargo test
7. [diagnostics] 검증
```

### 사례 4: 문서 생성

```
사용자: @src/agent/src/tools/ 디렉터리의 
모든 도구에 대한 문서를 생성해주세요.

[에이전트 프로파일: Ask]

에이전트 작업:
1. [list_directory] src/agent/src/tools/
2. [read_file] 각 도구 파일 읽기
3. 각 도구의 목적, 입력, 출력 분석
4. [create_file] docs/tools-reference.md
   - 모든 도구 문서화
   - 사용 예제 포함
   - API 스키마 포함
```

---

## 디버깅 팁

### 1. ACP 로그 확인

```
Cmd+Shift+P → "dev: open acp logs"
```

메시지 교환 내용 확인:
- Request/Response 페어
- 도구 호출 및 결과
- 에러 메시지

### 2. 스레드를 Markdown으로 내보내기

```
Cmd+Shift+P → "agent: open thread as markdown"
```

전체 대화 내용을 Markdown 형식으로 볼 수 있음:
- 디버깅에 유용
- GitHub 이슈 첨부 가능

### 3. 토큰 사용량 모니터링

Agent Panel 하단의 토큰 사용량 확인:
- 입력 토큰
- 출력 토큰
- 캐시 토큰 (지원 시)

### 4. 도구 실행 로그

각 도구 실행 시:
- 입력 파라미터 확인
- 실행 시간 측정
- 결과 검증

---

## 마무리

이 문서는 Zed AI Agent의 실전 사용법과 코드 예제를 제공합니다. 
더 많은 정보는 다음 문서를 참고하세요:

- [AI Agent 분석 문서 (한국어)](./ai-agent-analysis-ko.md)
- [AI Agent Analysis (English)](./ai-agent-analysis-en.md)
- [Architecture Diagram](./ai-agent-architecture-diagram.md)
- [Zed 공식 문서](https://zed.dev/docs/ai)

---

**문서 버전**: 1.0
**작성일**: 2025년 11월
**라이선스**: GPL-3.0-or-later
