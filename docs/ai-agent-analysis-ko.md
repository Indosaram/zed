# Zed AI Agent 기능 분석 (AI Agent Feature Analysis)

## 개요 (Overview)

이 문서는 Zed 에디터의 AI 에이전트 기능에 대한 포괄적인 기술 분석을 제공합니다. Zed의 에이전트 시스템은 대규모 언어 모델(LLM)을 활용하여 코드베이스와 상호작용하고, 자율적으로 편집을 수행하며, 다양한 도구를 통해 작업을 수행할 수 있는 강력한 기능을 제공합니다.

## 아키텍처 (Architecture)

### 핵심 크레이트 (Core Crates)

Zed의 에이전트 기능은 여러 개의 독립적인 크레이트로 구성되어 있으며, 각각 명확한 책임을 가지고 있습니다:

#### 1. `agent` 크레이트
**위치**: `crates/agent/`
**역할**: 핵심 에이전트 로직 및 비즈니스 로직

**주요 컴포넌트**:
- `agent.rs` (1,636 줄) - 메인 에이전트 엔트리포인트 및 조율
- `thread.rs` (2,640 줄) - 대화 스레드 관리 및 메시지 처리
- `edit_agent.rs` (1,491 줄) - 코드 편집 에이전트
- `tools/` - 14개의 내장 도구 구현
- `db.rs` - 데이터베이스 영속성
- `history_store.rs` - 대화 히스토리 관리
- `native_agent_server.rs` - 네이티브 에이전트 서버 구현

**도구 시스템** (총 17개 도구):
```
read_file_tool.rs       (970 줄)  - 파일 읽기
edit_file_tool.rs       (1,762 줄) - 파일 편집
grep_tool.rs            (1,190 줄) - 코드 검색
list_directory_tool.rs  (662 줄)  - 디렉터리 나열
find_path_tool.rs       (253 줄)  - 파일 경로 찾기
terminal_tool.rs        (213 줄)  - 터미널 명령 실행
diagnostics_tool.rs     (165 줄)  - 진단 정보
fetch_tool.rs           (164 줄)  - URL 가져오기
open_tool.rs            (170 줄)  - 파일/URL 열기
delete_path_tool.rs     (140 줄)  - 경로 삭제
web_search_tool.rs      (132 줄)  - 웹 검색
move_path_tool.rs       (124 줄)  - 경로 이동
copy_path_tool.rs       (113 줄)  - 경로 복사
create_directory_tool.rs (90 줄)  - 디렉터리 생성
now_tool.rs             (64 줄)   - 현재 시간
thinking_tool.rs        (52 줄)   - 사고 프로세스
context_server_registry.rs (253 줄) - MCP 서버 통합
```

#### 2. `agent_ui` 크레이트
**위치**: `crates/agent_ui/`
**역할**: 사용자 인터페이스 및 상호작용

**주요 컴포넌트**:
- `agent_panel.rs` - 메인 에이전트 패널 UI
- `agent_model_selector.rs` - 모델 선택 UI
- `agent_configuration.rs` - 설정 UI
- `agent_diff.rs` - 변경사항 비교 UI
- `inline_assistant.rs` - 인라인 어시스턴트
- `terminal_inline_assistant.rs` - 터미널 인라인 어시스턴트
- `context_picker.rs` - 컨텍스트 선택기
- `slash_command.rs` - 슬래시 커맨드 시스템

#### 3. `agent_servers` 크레이트
**위치**: `crates/agent_servers/`
**역할**: 외부 에이전트 서버 통합

**지원 에이전트**:
- Gemini CLI (Google)
- Claude Code (Anthropic)
- Codex (OpenAI)
- Custom ACP 호환 에이전트

#### 4. `agent_settings` 크레이트
**위치**: `crates/agent_settings/`
**역할**: 설정 및 프로파일 관리

**기능**:
- 에이전트 프로파일 (Write, Ask, Minimal)
- 도구 권한 설정
- 모델 선택 설정
- 사용자 정의 규칙

#### 5. `acp_thread` 크레이트
**위치**: `crates/acp_thread/`
**역할**: Agent Client Protocol (ACP) 구현

**기능**:
- ACP 프로토콜 메시지 처리
- 외부 에이전트와의 통신
- 스트림 처리

## Agent Client Protocol (ACP)

### 개요
ACP는 Zed가 외부 터미널 기반 에이전트와 통신하기 위한 프로토콜입니다. 이를 통해 Gemini CLI, Claude Code, Codex 등과 같은 외부 에이전트를 Zed UI 내에서 사용할 수 있습니다.

### 프로토콜 특징
- **표준 인터페이스**: 모든 ACP 호환 에이전트는 동일한 프로토콜을 사용
- **양방향 통신**: Zed ↔ 외부 에이전트 간 실시간 메시지 교환
- **도구 호출**: 에이전트가 Zed의 도구를 호출 가능
- **스트리밍**: 실시간 응답 스트리밍 지원

### 지원되는 외부 에이전트

#### Gemini CLI
- Google의 공식 Gemini CLI 도구
- OAuth 및 API 키 인증 지원
- Vertex AI 통합 가능
- 자동 설치 및 업데이트

#### Claude Code
- Anthropic의 Claude Code 에이전트
- Claude Pro/Max 구독 또는 API 키 사용
- CLAUDE.md 파일 자동 인식
- 서브에이전트 지원
- 커스텀 슬래시 커맨드

#### Codex CLI
- OpenAI의 Codex 에이전트
- ChatGPT 구독 또는 API 키 사용
- 다양한 인증 방법

## 도구 시스템 (Tools System)

### 도구 카테고리

#### 읽기 및 검색 도구 (Read & Search Tools)
1. **read_file** - 파일 내용 읽기
2. **grep** - 정규식을 사용한 코드 검색
3. **find_path** - glob 패턴으로 파일 찾기
4. **list_directory** - 디렉터리 내용 나열
5. **diagnostics** - 오류 및 경고 가져오기
6. **fetch** - URL에서 컨텐츠 가져오기
7. **web_search** - 웹 검색
8. **now** - 현재 날짜/시간
9. **open** - 파일/URL 열기
10. **thinking** - 에이전트의 사고 과정

#### 편집 도구 (Edit Tools)
1. **edit_file** - 파일 편집 (텍스트 대체)
2. **create_file** - 새 파일 생성
3. **delete_path** - 파일/디렉터리 삭제
4. **move_path** - 파일/디렉터리 이동
5. **copy_path** - 파일/디렉터리 복사
6. **create_directory** - 디렉터리 생성
7. **terminal** - 셸 명령 실행

### 도구 승인 시스템
- `agent.always_allow_tool_actions` 설정
- 기본값: false (승인 필요)
- true로 설정 시 자동 승인

### 모델 호환성
모든 도구가 모든 모델에서 작동하는 것은 아닙니다. Zed는 다음을 확인합니다:
- 모델의 도구 호출 지원 여부
- 특정 도구에 대한 모델 프로바이더 지원
- Zed의 호스팅 모델은 모든 도구를 지원

## 스레드 및 메시지 관리 (Thread & Message Management)

### Thread 구조
```rust
pub struct Thread {
    // 대화 메시지 목록
    messages: Vec<Message>,
    // 사용 중인 언어 모델
    model: Arc<dyn LanguageModel>,
    // 프로젝트 참조
    project: Entity<Project>,
    // 도구 레지스트리
    tools: Vec<Box<dyn AgentTool>>,
    // 실행 중인 태스크
    current_task: Option<Task<()>>,
}
```

### Message 타입
```rust
pub enum Message {
    User(UserMessage),      // 사용자 메시지
    Agent(AgentMessage),    // 에이전트 응답
    Resume,                 // 재개 마커
}
```

### UserMessage 구조
```rust
pub struct UserMessage {
    id: UserMessageId,
    content: Vec<UserMessageContent>,
}

pub enum UserMessageContent {
    Text(SharedString),                // 텍스트
    Image(LanguageModelImage),         // 이미지
    Mention { uri, content },          // 멘션 (@파일, @심볼 등)
}
```

### AgentMessage 구조
```rust
pub struct AgentMessage {
    chunks: Vec<AgentMessageChunk>,    // 응답 청크
    tool_uses: Vec<ToolUse>,           // 사용된 도구
    status: MessageStatus,             // 메시지 상태
}
```

## 컨텍스트 시스템 (Context System)

### 멘션 시스템
사용자는 `@` 기호를 사용하여 다양한 컨텍스트를 추가할 수 있습니다:

1. **@파일** - 파일 내용 포함
2. **@디렉터리** - 디렉터리 구조 포함
3. **@심볼** - 특정 코드 심볼
4. **@선택영역** - 현재 선택된 텍스트
5. **@스레드** - 이전 대화 스레드
6. **@규칙** - 규칙 파일 (.rules, CLAUDE.md 등)
7. **@페치** - 웹 URL 컨텐츠

### 규칙 파일 (Rules Files)
Zed는 다음 규칙 파일을 자동으로 인식합니다:
```rust
const RULES_FILE_NAMES: [&str; 9] = [
    ".rules",
    ".cursorrules",
    ".windsurfrules",
    ".clinerules",
    ".github/copilot-instructions.md",
    "CLAUDE.md",
    "AGENT.md",
    "AGENTS.md",
    "GEMINI.md",
];
```

### ProjectContext
```rust
pub struct ProjectContext {
    // 프로젝트 스냅샷
    worktree_snapshots: Vec<WorktreeSnapshot>,
    // 규칙 컨텍스트
    rules_context: RulesFileContext,
    // 사용자 규칙
    user_rules: UserRulesContext,
}
```

## 편집 에이전트 (Edit Agent)

### EditAgent 구조
```rust
pub struct EditAgent {
    model: Arc<dyn LanguageModel>,
    project: Entity<Project>,
    action_log: Entity<ActionLog>,
    templates: Arc<Templates>,
    edit_format: EditFormat,
}
```

### 편집 형식
- **XML 형식** - XML 태그로 편집 위치 지정
- **Diff Fenced 형식** - diff 블록을 사용한 편집

### 편집 파서
```rust
pub struct EditParser {
    // 스트리밍 diff 처리
    streaming_diff: StreamingDiff,
    // 퍼지 매칭
    fuzzy_matcher: StreamingFuzzyMatcher,
    // 파서 메트릭
    metrics: EditParserMetrics,
}
```

### 편집 프로세스
1. **편집 범위 해결** - 편집할 위치 찾기
2. **퍼지 매칭** - 유사한 코드 패턴 찾기
3. **스트리밍 diff** - 실시간 변경사항 적용
4. **검증** - 구문 오류 확인

## 프로파일 시스템 (Profile System)

### 내장 프로파일

#### Write 프로파일
- **목적**: 코드베이스에 쓰기 작업 허용
- **도구**: 모든 읽기 + 모든 편집 도구
- **사용 사례**: 코드 생성, 리팩토링, 파일 조작

#### Ask 프로파일
- **목적**: 읽기 전용 작업
- **도구**: 읽기 및 검색 도구만
- **사용 사례**: 코드베이스 탐색, 질문 답변

#### Minimal 프로파일
- **목적**: 일반 대화
- **도구**: 도구 없음
- **사용 사례**: 일반적인 LLM 대화

### 커스텀 프로파일
사용자는 다음을 통해 커스텀 프로파일을 생성할 수 있습니다:
- UI를 통한 프로파일 생성/편집
- `settings.json`에서 수동 설정
- 기존 프로파일 포크
- 내장 프로파일 오버라이드

### 프로파일 설정 예시
```json
{
  "agent": {
    "profiles": {
      "my-custom-profile": {
        "name": "My Custom Profile",
        "tools": [
          "read_file",
          "grep",
          "edit_file"
        ]
      }
    }
  }
}
```

## Model Context Protocol (MCP)

### 개요
MCP는 외부 도구를 에이전트에 추가하는 표준화된 방법입니다.

### 통합 방식
1. **MCP 서버 설치** - Zed 익스텐션으로 설치
2. **자동 등록** - ContextServerRegistry에 등록
3. **도구 노출** - 에이전트가 MCP 도구 사용 가능

### 지원 현황
- **Zed 네이티브 에이전트**: 완전 지원
- **Claude Code**: 지원
- **Codex**: 지원
- **Gemini CLI**: 부분 지원 (문서 참조)

## 데이터베이스 및 영속성 (Database & Persistence)

### 히스토리 저장
```rust
pub struct HistoryStore {
    db: Arc<AgentDb>,
    sessions: HashMap<SessionId, ThreadSnapshot>,
}
```

### 데이터베이스 스키마
- **threads** - 대화 스레드
- **messages** - 메시지 내용
- **tool_uses** - 도구 사용 기록
- **snapshots** - 프로젝트 스냅샷

### 체크포인트 시스템
- 각 편집 전 자동 체크포인트 생성
- 언제든지 이전 상태로 복원 가능
- UI에서 "Restore Checkpoint" 버튼으로 접근

## 언어 모델 통합 (Language Model Integration)

### 지원 프로바이더
- **Zed Pro** - Zed의 호스팅 모델
- **Anthropic** - Claude 모델
- **OpenAI** - GPT 모델
- **Google AI** - Gemini 모델
- **Ollama** - 로컬 모델
- **OpenRouter** - 다양한 모델 게이트웨이
- **LM Studio** - 로컬 LLM 서버

### 모델 선택
```rust
pub struct LanguageModels {
    models: HashMap<ModelId, Arc<dyn LanguageModel>>,
    model_list: AgentModelList,
}
```

### 모델 인증
- API 키 기반
- OAuth 기반 (Gemini)
- 구독 기반 (Zed Pro)

## 토큰 사용량 관리 (Token Usage Management)

### 사용량 추적
```rust
pub struct TokenUsage {
    input_tokens: u64,
    output_tokens: u64,
    cache_creation_tokens: Option<u64>,
    cache_read_tokens: Option<u64>,
}
```

### UI 표시
- 현재 스레드의 토큰 사용량 표시
- 컨텍스트 윈도우 근접 시 경고
- 스레드 요약 및 새 스레드 시작 제안

## 오류 처리 및 재시도 (Error Handling & Retry)

### 재시도 전략
```rust
enum RetryStrategy {
    ExponentialBackoff {
        initial_delay: Duration,
        max_attempts: u8,
    },
    Fixed {
        delay: Duration,
        max_attempts: u8,
    },
}
```

### 오류 유형
- 네트워크 오류 - 자동 재시도
- 인증 오류 - 사용자 액션 필요
- 레이트 리밋 - 백오프 후 재시도
- 모델 오류 - 사용자에게 표시

## 협업 기능 (Collaboration Features)

### 에이전트 팔로우
- 에이전트가 읽고 편집하는 것을 실시간으로 따라가기
- Crosshair 아이콘 또는 `Cmd/Ctrl + Enter`로 활성화
- Zed의 네이티브 협업 기능 활용

### 알림
```json
{
  "agent": {
    "notify_when_agent_waiting": true,
    "play_sound_when_agent_done": true
  }
}
```

## 피드백 및 개선 (Feedback & Improvement)

### 평가 시스템
- Thumbs up/down 버튼
- 상세 피드백 텍스트
- Zed 서버로 전송 (옵트인)

### 프라이버시
- 평가를 하지 않으면 데이터가 수집되지 않음
- Zed Pro 모델만 해당
- 외부 에이전트는 별도 개인정보 보호 정책

## 성능 최적화 (Performance Optimizations)

### 스트리밍
- 실시간 응답 스트리밍
- 청크 단위 처리
- UI 업데이트 최적화

### 캐싱
- 프로젝트 컨텍스트 캐싱
- 모델 리스트 캐싱
- 토큰 사용량 캐싱

### 비동기 처리
```rust
cx.spawn(async move |this, cx| {
    // 백그라운드 작업
})
```

## 테스트 (Testing)

### 테스트 구조
- `crates/agent/src/tests/` - 통합 테스트
- 각 도구별 단위 테스트
- EditAgent 평가 테스트

### 테스트 도구
```rust
#[cfg(test)]
mod tests {
    use super::*;
    
    #[gpui::test]
    async fn test_agent_functionality(cx: &mut App) {
        // 테스트 코드
    }
}
```

## 주요 API 및 사용 예시 (Key APIs & Usage Examples)

### 새 스레드 생성
```rust
// 네이티브 에이전트로 스레드 생성
agent::NewThread

// 외부 에이전트로 스레드 생성
agent::NewExternalAgentThread { agent: "gemini" }
```

### 메시지 전송
```rust
thread.send_message(user_message, cx)
```

### 도구 등록
```rust
impl AgentTool for MyCustomTool {
    fn name() -> &'static str {
        "my_custom_tool"
    }
    
    fn description() -> &'static str {
        "My custom tool description"
    }
    
    fn input_schema(format: LanguageModelToolSchemaFormat) -> Schema {
        // 스키마 정의
    }
    
    async fn run(input: serde_json::Value, cx: &AsyncApp) -> Result<String> {
        // 도구 로직
    }
}
```

## 설정 예시 (Configuration Examples)

### 기본 설정
```json
{
  "agent": {
    "version": "2",
    "default_model": {
      "provider": "zed",
      "name": "claude-sonnet-4-20250514"
    },
    "always_allow_tool_actions": false,
    "notify_when_agent_waiting": true,
    "play_sound_when_agent_done": true
  }
}
```

### 외부 에이전트 설정
```json
{
  "agent_servers": {
    "gemini": {
      "ignore_system_version": false
    },
    "claude": {
      "env": {
        "CLAUDE_CODE_EXECUTABLE": "/path/to/claude-code"
      }
    },
    "my_custom_agent": {
      "command": "node",
      "args": ["~/projects/agent/index.js", "--acp"],
      "env": {}
    }
  }
}
```

## 개발 가이드 (Development Guide)

### 새 도구 추가
1. `crates/agent/src/tools/`에 새 파일 생성
2. `AgentTool` 트레잇 구현
3. `tools.rs`의 `tools!` 매크로에 추가

### 새 에이전트 프로바이더 추가
1. `crates/agent_servers/src/`에 구현
2. ACP 프로토콜 준수
3. 설정 스키마 정의

### 빌드 및 테스트
```bash
# Clippy 실행
./script/clippy -p agent

# 테스트 실행
cargo test -p agent

# 특정 테스트 실행
cargo test -p agent --test test_name
```

## 향후 개선 방향 (Future Improvements)

### 계획된 기능
- [ ] 더 많은 외부 에이전트 지원
- [ ] 개선된 체크포인트 시스템
- [ ] 더 나은 diff 시각화
- [ ] 멀티모달 지원 강화
- [ ] 더 많은 MCP 서버 통합

### 제한사항
- 외부 에이전트의 일부 기능 제한 (메시지 편집, 히스토리 복원 등)
- 모델별 도구 지원 차이
- 토큰 제한

## 결론 (Conclusion)

Zed의 AI 에이전트 시스템은 다음과 같은 특징을 가진 포괄적이고 확장 가능한 아키텍처입니다:

1. **모듈식 설계** - 각 컴포넌트가 명확히 분리됨
2. **확장성** - 도구, 프로파일, 외부 에이전트 쉽게 추가 가능
3. **사용자 친화적** - 직관적인 UI와 설정
4. **강력한 기능** - 17개 내장 도구 + MCP 지원
5. **프라이버시 중심** - 옵트인 데이터 수집
6. **프로바이더 중립적** - 다양한 LLM 프로바이더 지원

이 시스템은 코드 편집, 리팩토링, 코드베이스 탐색, 일반 질문 등 다양한 작업을 효율적으로 수행할 수 있게 해줍니다.

## 참고 자료 (References)

- [Zed AI 문서](https://zed.dev/docs/ai)
- [Agent Client Protocol](https://agentclientprotocol.com)
- [Zed GitHub 저장소](https://github.com/zed-industries/zed)
- [Model Context Protocol](https://modelcontextprotocol.io)

---

**문서 버전**: 1.0
**작성일**: 2025년 11월
**라이선스**: GPL-3.0-or-later
