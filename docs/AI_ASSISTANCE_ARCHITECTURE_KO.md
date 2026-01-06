# Zed AI Assistance 아키텍처 분석

이 문서는 Zed 에디터의 AI 지원 기능이 어떻게 동작하는지에 대한 기술적 분석입니다.

## 목차

1. [개요](#개요)
2. [핵심 구성 요소](#핵심-구성-요소)
3. [언어 모델 시스템](#언어-모델-시스템)
4. [에이전트 시스템](#에이전트-시스템)
5. [도구(Tools) 시스템](#도구tools-시스템)
6. [인라인 어시스턴트](#인라인-어시스턴트)
7. [데이터 흐름](#데이터-흐름)
8. [MCP (Model Context Protocol)](#mcp-model-context-protocol)

---

## 개요

Zed의 AI 지원 기능은 여러 크레이트(Rust 패키지)로 구성된 모듈식 아키텍처를 가지고 있습니다. 주요 기능은 다음과 같습니다:

- **Agent Panel**: LLM과의 대화형 인터페이스로, 코드 생성 및 편집을 수행
- **Inline Assistant**: 에디터 내에서 직접 코드를 수정하는 기능
- **Edit Prediction**: AI 기반 코드 자동완성 예측 기능
- **Text Threads**: 텍스트 기반의 원시적인 대화 인터페이스

---

## 핵심 구성 요소

### 크레이트 구조

```
crates/
├── agent/                    # 핵심 에이전트 로직
│   ├── thread.rs             # 대화 스레드 관리
│   ├── agent.rs              # 에이전트 코어 로직
│   └── tools/                # 내장 도구들
├── agent_ui/                 # 에이전트 UI 컴포넌트
│   ├── agent_panel.rs        # Agent Panel UI
│   ├── inline_assistant.rs   # 인라인 어시스턴트
│   └── buffer_codegen.rs     # 버퍼 코드 생성
├── language_model/           # 언어 모델 추상화 레이어
│   ├── language_model.rs     # LanguageModel 트레이트 정의
│   ├── registry.rs           # 모델 레지스트리
│   └── request.rs            # 요청 구조체
├── language_models/          # 각 제공자별 구현
│   └── provider/
│       ├── anthropic.rs      # Claude 모델
│       ├── open_ai.rs        # OpenAI 모델
│       ├── google.rs         # Google AI 모델
│       ├── ollama.rs         # 로컬 Ollama 모델
│       └── cloud.rs          # Zed Cloud 제공자
└── acp_thread/               # Agent Client Protocol
    ├── acp_thread.rs         # ACP 스레드 구현
    └── connection.rs         # 연결 관리
```

---

## 언어 모델 시스템

### LanguageModel 트레이트

`language_model` 크레이트에서 정의된 `LanguageModel` 트레이트는 모든 LLM 제공자가 구현해야 하는 인터페이스입니다:

```rust
pub trait LanguageModel: Send + Sync {
    // 모델 식별
    fn id(&self) -> LanguageModelId;
    fn name(&self) -> LanguageModelName;
    fn provider_id(&self) -> LanguageModelProviderId;
    fn provider_name(&self) -> LanguageModelProviderName;
    
    // 모델 기능
    fn supports_images(&self) -> bool;
    fn supports_tools(&self) -> bool;
    fn max_token_count(&self) -> u64;
    
    // 핵심 메서드
    fn count_tokens(&self, request: LanguageModelRequest, cx: &App) -> BoxFuture<'static, Result<u64>>;
    fn stream_completion(&self, request: LanguageModelRequest, cx: &AsyncApp) -> BoxFuture<'static, Result<BoxStream<...>>>;
}
```

### 모델 제공자 (Providers)

Zed는 다양한 LLM 제공자를 지원합니다:

| 제공자 | 설명 | 크레이트 |
|--------|------|----------|
| **Anthropic** | Claude 모델 (Sonnet, Opus 등) | `provider/anthropic.rs` |
| **OpenAI** | GPT-4, GPT-4o 등 | `provider/open_ai.rs` |
| **Google AI** | Gemini 시리즈 | `provider/google.rs` |
| **Ollama** | 로컬 모델 호스팅 | `provider/ollama.rs` |
| **Zed Cloud** | Zed의 호스팅 서비스 | `provider/cloud.rs` |
| **OpenRouter** | 여러 모델 게이트웨이 | `provider/open_router.rs` |
| **LM Studio** | 로컬 모델 서버 | `provider/lmstudio.rs` |
| **DeepSeek** | DeepSeek 모델 | `provider/deepseek.rs` |
| **Mistral** | Mistral AI 모델 | `provider/mistral.rs` |

### LanguageModelRegistry

모델들은 `LanguageModelRegistry`를 통해 관리됩니다:

```rust
// 전역 레지스트리 접근
let registry = LanguageModelRegistry::global(cx);

// 기본 모델 가져오기
let default_model = registry.default_model();

// 특정 모델 선택
let model = registry.select_model(&selected, cx);
```

---

## 에이전트 시스템

### NativeAgent

`NativeAgent`는 Zed의 내장 에이전트 구현입니다:

```rust
pub struct NativeAgent {
    sessions: HashMap<acp::SessionId, Session>,     // 세션 관리
    history: Entity<HistoryStore>,                   // 대화 기록
    project_context: Entity<ProjectContext>,         // 프로젝트 컨텍스트
    context_server_registry: Entity<ContextServerRegistry>, // MCP 서버
    templates: Arc<Templates>,                       // 프롬프트 템플릿
    models: LanguageModels,                          // 사용 가능한 모델
    project: Entity<Project>,                        // 프로젝트 참조
    prompt_store: Option<Entity<PromptStore>>,       // 프롬프트 저장소
    fs: Arc<dyn Fs>,                                 // 파일 시스템
}
```

### Thread (대화 스레드)

`Thread`는 에이전트와의 대화를 관리합니다:

```rust
pub struct Thread {
    id: acp::SessionId,                      // 스레드 ID
    messages: Vec<Message>,                  // 메시지 목록
    model: Option<Arc<dyn LanguageModel>>,   // 현재 모델
    tools: BTreeMap<SharedString, Arc<dyn AnyAgentTool>>, // 사용 가능한 도구
    running_turn: Option<RunningTurn>,       // 진행 중인 턴
    pending_message: Option<AgentMessage>,   // 대기 중인 메시지
    project: Entity<Project>,                // 프로젝트 참조
    action_log: Entity<ActionLog>,           // 액션 로그
}
```

### 메시지 흐름

1. **사용자 메시지 전송** (`Thread::send`)
   ```rust
   pub fn send(&mut self, id: UserMessageId, content: impl IntoIterator<Item = T>, cx: &mut Context<Self>) 
       -> Result<mpsc::UnboundedReceiver<Result<ThreadEvent>>>
   ```

2. **모델 응답 처리** (`Thread::run_turn_internal`)
   - 완료 요청 생성
   - 스트리밍 응답 처리
   - 도구 호출 처리

3. **이벤트 스트림** (`ThreadEvent`)
   ```rust
   pub enum ThreadEvent {
       UserMessage(UserMessage),
       AgentText(String),
       AgentThinking(String),
       ToolCall(acp::ToolCall),
       ToolCallUpdate(ToolCallUpdate),
       ToolCallAuthorization(ToolCallAuthorization),
       Retry(RetryStatus),
       Stop(acp::StopReason),
   }
   ```

---

## 도구(Tools) 시스템

### AgentTool 트레이트

모든 도구는 `AgentTool` 트레이트를 구현합니다:

```rust
pub trait AgentTool {
    type Input: for<'de> Deserialize<'de> + Serialize + JsonSchema;
    type Output: for<'de> Deserialize<'de> + Serialize + Into<LanguageModelToolResultContent>;

    fn name() -> &'static str;
    fn description() -> SharedString;
    fn kind() -> acp::ToolKind;
    fn initial_title(&self, input: Result<Self::Input, serde_json::Value>, cx: &mut App) -> SharedString;
    fn run(self: Arc<Self>, input: Self::Input, event_stream: ToolCallEventStream, cx: &mut App) -> Task<Result<Self::Output>>;
}
```

### 내장 도구 목록

#### 읽기/검색 도구
| 도구 | 설명 | 파일 |
|------|------|------|
| `diagnostics` | 파일/프로젝트 에러 및 경고 조회 | `diagnostics_tool.rs` |
| `fetch` | URL 콘텐츠를 Markdown으로 가져오기 | `fetch_tool.rs` |
| `find_path` | glob 패턴으로 파일 찾기 | `find_path_tool.rs` |
| `grep` | 정규식으로 코드 검색 | `grep_tool.rs` |
| `list_directory` | 디렉토리 내용 나열 | `list_directory_tool.rs` |
| `now` | 현재 날짜/시간 반환 | `now_tool.rs` |
| `open` | 기본 앱으로 파일/URL 열기 | `open_tool.rs` |
| `read_file` | 파일 내용 읽기 | `read_file_tool.rs` |
| `thinking` | 문제 해결을 위한 사고 과정 | `thinking_tool.rs` |
| `web_search` | 웹 검색 수행 | `web_search_tool.rs` |

#### 편집 도구
| 도구 | 설명 | 파일 |
|------|------|------|
| `copy_path` | 파일/디렉토리 복사 | `copy_path_tool.rs` |
| `create_directory` | 새 디렉토리 생성 | `create_directory_tool.rs` |
| `delete_path` | 파일/디렉토리 삭제 | `delete_path_tool.rs` |
| `edit_file` | 파일 내용 수정 | `edit_file_tool.rs` |
| `move_path` | 파일/디렉토리 이동 | `move_path_tool.rs` |
| `save_file` | 파일 저장 | `save_file_tool.rs` |
| `terminal` | 쉘 명령 실행 | `terminal_tool.rs` |

### 도구 등록

```rust
pub fn add_default_tools(&mut self, environment: Rc<dyn ThreadEnvironment>, cx: &mut Context<Self>) {
    self.add_tool(CopyPathTool::new(self.project.clone()));
    self.add_tool(CreateDirectoryTool::new(self.project.clone()));
    self.add_tool(DeletePathTool::new(self.project.clone(), self.action_log.clone()));
    self.add_tool(DiagnosticsTool::new(self.project.clone()));
    self.add_tool(EditFileTool::new(self.project.clone(), cx.weak_entity(), language_registry, Templates::new()));
    self.add_tool(FetchTool::new(self.project.read(cx).client().http_client()));
    self.add_tool(FindPathTool::new(self.project.clone()));
    self.add_tool(GrepTool::new(self.project.clone()));
    self.add_tool(ListDirectoryTool::new(self.project.clone()));
    self.add_tool(MovePathTool::new(self.project.clone()));
    self.add_tool(NowTool);
    self.add_tool(OpenTool::new(self.project.clone()));
    self.add_tool(ReadFileTool::new(cx.weak_entity(), self.project.clone(), self.action_log.clone()));
    self.add_tool(SaveFileTool::new(self.project.clone()));
    self.add_tool(RestoreFileFromDiskTool::new(self.project.clone()));
    self.add_tool(TerminalTool::new(self.project.clone(), environment));
    self.add_tool(ThinkingTool);
    self.add_tool(WebSearchTool);
}
```

---

## 인라인 어시스턴트

### InlineAssistant

`InlineAssistant`는 에디터 내에서 직접 코드를 수정하는 기능을 제공합니다:

```rust
pub struct InlineAssistant {
    next_assist_id: InlineAssistId,
    assists: HashMap<InlineAssistId, InlineAssist>,
    assists_by_editor: HashMap<WeakEntity<Editor>, EditorInlineAssists>,
    assist_groups: HashMap<InlineAssistGroupId, InlineAssistGroup>,
    prompt_history: VecDeque<String>,
    prompt_builder: Arc<PromptBuilder>,
    fs: Arc<dyn Fs>,
}
```

### 동작 방식

1. 사용자가 텍스트 선택 후 `assistant::InlineAssist` 실행
2. 프롬프트 에디터가 표시됨
3. 사용자가 프롬프트 입력 후 Enter
4. 선택된 코드와 프롬프트가 LLM으로 전송
5. 응답이 스트리밍되면서 선택 영역이 수정됨
6. 사용자가 변경 사항을 수락하거나 취소

---

## 데이터 흐름

### 완료 요청 흐름

```
┌─────────────┐
│   사용자    │
│  프롬프트   │
└──────┬──────┘
       │
       ▼
┌─────────────┐
│   Thread    │
│  (send())   │
└──────┬──────┘
       │
       ▼
┌─────────────────────────────┐
│   build_completion_request  │
│  • 시스템 프롬프트 생성     │
│  • 메시지 이력 포함         │
│  • 도구 정의 추가           │
└──────────────┬──────────────┘
               │
               ▼
┌─────────────────────────────┐
│   LanguageModel             │
│   stream_completion()       │
└──────────────┬──────────────┘
               │
               ▼
┌─────────────────────────────┐
│   스트리밍 이벤트 처리      │
│  • Text                     │
│  • Thinking                 │
│  • ToolUse                  │
│  • UsageUpdate              │
│  • Stop                     │
└──────────────┬──────────────┘
               │
               ▼
┌─────────────────────────────┐
│   도구 실행 (해당 시)       │
│  • 입력 파싱                │
│  • 도구 실행                │
│  • 결과 수집                │
└──────────────┬──────────────┘
               │
               ▼
┌─────────────────────────────┐
│   결과 반환 및 표시         │
│  • ThreadEvent 방출         │
│  • UI 업데이트              │
└─────────────────────────────┘
```

### 시스템 프롬프트 구성

시스템 프롬프트는 `SystemPromptTemplate`를 통해 생성됩니다:

```rust
pub struct SystemPromptTemplate {
    pub project: ProjectContext,     // 프로젝트 정보
    pub available_tools: Vec<SharedString>, // 사용 가능한 도구
    pub model_name: Option<String>,  // 모델 이름
}
```

프롬프트에 포함되는 요소:
- 프로젝트 구조 정보
- Rules 파일 내용 (`.rules`, `.cursorrules` 등)
- 사용 가능한 도구 목록
- 사용자 정의 규칙

---

## MCP (Model Context Protocol)

### MCP 서버 통합

Zed는 Model Context Protocol을 통해 외부 도구 및 컨텍스트 서버를 통합합니다:

```rust
pub struct ContextServerRegistry {
    server_store: Entity<ContextServerStore>,
    servers: HashMap<ContextServerId, ContextServerState>,
}

struct ContextServerState {
    tools: HashMap<SharedString, Arc<dyn AnyAgentTool>>,
    prompts: Vec<ContextServerPrompt>,
}
```

### MCP 서버 설정

`settings.json`에서 MCP 서버를 구성할 수 있습니다:

```json
{
  "context_servers": {
    "my-server": {
      "command": "node",
      "args": ["./server.js"],
      "env": {}
    },
    "remote-server": {
      "url": "https://api.example.com/mcp",
      "headers": { "Authorization": "Bearer <token>" }
    }
  }
}
```

### 확장 기반 MCP 서버

Zed 확장을 통해 MCP 서버를 배포할 수도 있습니다:
- GitHub MCP Server
- Puppeteer MCP Server
- Brave Search MCP Server
- 등

---

## 프로필 시스템

### AgentProfileSettings

프로필을 통해 도구 세트를 관리할 수 있습니다:

```rust
pub struct AgentProfileSettings {
    pub name: SharedString,
    pub tools: HashMap<SharedString, bool>,           // 내장 도구 활성화
    pub enable_all_context_servers: bool,             // MCP 서버 전체 활성화
    pub context_servers: HashMap<String, ContextServerProfile>, // MCP 서버별 설정
    pub default_model: Option<LanguageModelSelection>, // 기본 모델
}
```

### 내장 프로필

1. **Write**: 모든 도구 활성화, 코드 작성용
2. **Ask**: 읽기 전용 도구만 활성화, 질문용
3. **Minimal**: 도구 없음, 일반 대화용

---

## 에러 처리 및 재시도

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

### 재시도 가능한 에러

- `TOO_MANY_REQUESTS`: Rate limit 초과
- `SERVICE_UNAVAILABLE`: 서버 과부하
- `INTERNAL_SERVER_ERROR`: 서버 오류 (최대 3회)
- 네트워크 오류: 일시적 연결 문제

### 재시도 불가능한 에러

- `PAYLOAD_TOO_LARGE`: 프롬프트가 너무 큽니다
- `UNAUTHORIZED`: 인증 실패
- `FORBIDDEN`: 권한 없음
- API 키 없음

---

## 주요 설정 옵션

### agent 설정

```json
{
  "agent": {
    "default_model": {
      "provider": "anthropic",
      "model": "claude-sonnet-4-20250514"
    },
    "default_profile": "Write",
    "always_allow_tool_actions": false,
    "inline_alternatives": [],
    "notify_when_agent_waiting": true,
    "play_sound_when_agent_done": false
  }
}
```

---

## 결론

Zed의 AI 지원 시스템은 다음과 같은 핵심 원칙을 따릅니다:

1. **모듈화**: 각 기능이 독립적인 크레이트로 분리
2. **추상화**: `LanguageModel` 트레이트를 통한 다양한 제공자 지원
3. **확장성**: MCP를 통한 외부 도구 통합
4. **유연성**: 프로필을 통한 도구 세트 관리
5. **안정성**: 재시도 전략과 에러 처리

이 아키텍처는 새로운 LLM 제공자 추가, 커스텀 도구 개발, MCP 서버 통합을 쉽게 할 수 있도록 설계되었습니다.
