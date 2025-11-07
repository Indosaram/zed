# AI Agent 완전 가이드 (Complete AI Agent Guide)

> 이 문서는 AI 에이전트 시스템의 아키텍처, 구현, 사용 패턴에 대한 포괄적인 가이드입니다.
> 다른 코드베이스에서 AI 에이전트를 구현하거나 이해하는 데 참고할 수 있습니다.

## 📚 목차 (Table of Contents)

### Part I: 아키텍처 및 설계
1. [개요](#개요)
2. [시스템 아키텍처](#시스템-아키텍처)
3. [핵심 컴포넌트](#핵심-컴포넌트)
4. [도구 시스템](#도구-시스템)
5. [프로토콜 및 통신](#프로토콜-및-통신)

### Part II: 구현 상세
6. [스레드 및 메시지 관리](#스레드-및-메시지-관리)
7. [컨텍스트 시스템](#컨텍스트-시스템)
8. [편집 에이전트](#편집-에이전트)
9. [프로파일 시스템](#프로파일-시스템)
10. [데이터 영속성](#데이터-영속성)

### Part III: 실전 가이드
11. [기본 사용법](#기본-사용법)
12. [커스텀 도구 구현](#커스텀-도구-구현)
13. [프로파일 설정](#프로파일-설정)
14. [외부 에이전트 통합](#외부-에이전트-통합)
15. [고급 패턴](#고급-패턴)

---

# Part I: 아키텍처 및 설계

## 개요

AI 에이전트 시스템은 대규모 언어 모델(LLM)을 활용하여 코드베이스와 상호작용하고, 자율적으로 편집을 수행하며, 다양한 도구를 통해 작업을 수행하는 시스템입니다.

### 핵심 설계 원칙

1. **모듈식 아키텍처**: 각 컴포넌트가 독립적이고 교체 가능
2. **확장 가능성**: 새로운 도구와 에이전트 쉽게 추가
3. **타입 안전성**: 강력한 타입 시스템으로 런타임 오류 최소화
4. **비동기 처리**: 효율적인 리소스 사용

### 주요 기능

- ✅ 17개 내장 도구 (읽기, 편집, 검색, 실행)
- ✅ 다중 LLM 프로바이더 지원
- ✅ 외부 에이전트 통합 (ACP)
- ✅ 확장 가능한 도구 시스템 (MCP)
- ✅ 체크포인트 기반 안전한 편집
- ✅ 실시간 스트리밍 응답

## 시스템 아키텍처

### 전체 레이어 구조

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                               UI Layer                                        │
│  ┌──────────────────────────────────────────────────────────────────────┐  │
│  │  Agent Panel │ Model Selector │ Context Picker │ Diff Viewer        │  │
│  │  Inline Assistant │ Terminal Assistant │ Slash Commands             │  │
│  └──────────────────────────────────────────────────────────────────────┘  │
└───────────────────────────────────────┬─────────────────────────────────────┘
                                        │
                                        ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│                          Core Agent Logic Layer                              │
│  ┌──────────────────────────────────────────────────────────────────────┐  │
│  │                        NativeAgent                                    │  │
│  │  ┌───────────────┐  ┌────────────────┐  ┌───────────────┐          │  │
│  │  │ Thread        │  │ LanguageModels │  │ ProjectContext│          │  │
│  │  │ Management    │  │                 │  │               │          │  │
│  │  └───────────────┘  └────────────────┘  └───────────────┘          │  │
│  │  ┌───────────────┐  ┌────────────────┐  ┌───────────────┐          │  │
│  │  │ History Store │  │ Templates      │  │ Context Server│          │  │
│  │  └───────────────┘  └────────────────┘  └───────────────┘          │  │
│  └──────────────────────────────────────────────────────────────────────┘  │
│                                                                               │
│  ┌──────────────────────────────────────────────────────────────────────┐  │
│  │                         EditAgent                                     │  │
│  │  ┌────────────┐  ┌──────────────┐  ┌──────────────────┐            │  │
│  │  │ Edit Parser│  │ Fuzzy Matcher│  │ Streaming Diff   │            │  │
│  │  └────────────┘  └──────────────┘  └──────────────────┘            │  │
│  └──────────────────────────────────────────────────────────────────────┘  │
│                                                                               │
│  ┌──────────────────────────────────────────────────────────────────────┐  │
│  │                    Tools System (17 tools)                            │  │
│  │  Read/Search Tools     │  Edit Tools         │  Utility Tools        │  │
│  │  • read_file           │  • edit_file        │  • thinking           │  │
│  │  • grep                │  • create_file      │  • now                │  │
│  │  • find_path           │  • delete_path      │  • open               │  │
│  │  • list_directory      │  • move_path        │  • web_search         │  │
│  │  • diagnostics         │  • copy_path        │  • fetch              │  │
│  │                        │  • create_directory │                       │  │
│  │                        │  • terminal         │                       │  │
│  └──────────────────────────────────────────────────────────────────────┘  │
└───────────────────────────────────┬───────────────────────────────────────┬─┘
                                    │                                       │
                                    ▼                                       │
┌─────────────────────────────────────────────────────────┐                │
│            Agent Client Protocol (ACP) Layer             │                │
│  ┌──────────────────────────────────────────────────┐  │                │
│  │              ACP Thread                           │  │                │
│  │  Protocol Message Handling │ Stream Processing  │  │                │
│  └──────────────────────────────────────────────────┘  │                │
│  ┌──────────────────────────────────────────────────┐  │                │
│  │         External Agent Servers                    │  │                │
│  │  Gemini CLI │ Claude Code │ Codex │ Custom       │  │                │
│  └──────────────────────────────────────────────────┘  │                │
└─────────────────────────────────────────────────────────┘                │
                                                                            │
                                                                            ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│                      Language Model Integration Layer                        │
│  ┌──────────────────────────────────────────────────────────────────────┐  │
│  │                    Model Providers                                    │  │
│  │  Zed Pro │ Anthropic │ OpenAI │ Google AI │ Ollama │ OpenRouter     │  │
│  └──────────────────────────────────────────────────────────────────────┘  │
└───────────────────────────────────────┬─────────────────────────────────────┘
                                        │
                                        ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│                       Settings & Configuration Layer                         │
│  ┌──────────────────────────────────────────────────────────────────────┐  │
│  │  Profiles │ Tool Config │ Model Selection │ Rules Files               │  │
│  └──────────────────────────────────────────────────────────────────────┘  │
└───────────────────────────────────────┬─────────────────────────────────────┘
                                        │
                                        ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│                       Data Persistence Layer (SQLite)                        │
│  ┌──────────────────────────────────────────────────────────────────────┐  │
│  │  threads │ messages │ tool_uses │ snapshots │ checkpoints            │  │
│  └──────────────────────────────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────────────────────────┘
```

### 데이터 흐름

```
User Input → Agent Panel → Thread → Language Model → Tools → Code Changes
     ↓           ↓           ↓            ↓           ↓           ↓
  Mention   Context     Messages    Streaming    Tool Call   Apply Edits
  System    Picker                  Response                      ↓
     ↓                                  ↓                    Checkpoint
  Project                          Parse Tool                     ↓
  Context                          Results                   Save to DB
     ↓                                  ↓                          ↓
  Rules                            Update UI              Show in Diff View
  Files                                 ↓                          ↓
                                   Agent Response         User Accept/Reject
```

### 메시지 처리 흐름

```
┌──────────────┐
│ User Message │
└──────┬───────┘
       ▼
┌──────────────────────────┐
│ Collect Context          │
│ - Mentions               │
│ - Project snapshot       │
│ - Rules files            │
└──────┬───────────────────┘
       ▼
┌──────────────────────────┐
│ Build LLM Request        │
│ - System prompt          │
│ - User message           │
│ - Available tools        │
└──────┬───────────────────┘
       ▼
┌──────────────────────────┐
│ Stream LLM Response      │
└──────┬───────────────────┘
       ▼
┌──────────────────────────┐
│ Parse Response           │
│ - Text chunks            │
│ - Tool calls             │
└──────┬───────────────────┘
       ▼
┌──────────────────────────┐
│ Execute Tools            │
└──────┬───────────────────┘
       ▼
┌──────────────────────────┐
│ Apply Edits              │
└──────┬───────────────────┘
       ▼
┌──────────────────────────┐
│ Store in History         │
│ Create Checkpoint        │
└──────┬───────────────────┘
       ▼
┌──────────────────────────┐
│ Update UI                │
└──────────────────────────┘
```

## 핵심 컴포넌트

### 1. Agent Core Module

**구조**:
```rust
pub struct NativeAgent {
    sessions: HashMap<SessionId, Session>,
    history: Entity<HistoryStore>,
    project_context: Entity<ProjectContext>,
    context_server_registry: Entity<ContextServerRegistry>,
    templates: Arc<Templates>,
    models: LanguageModels,
    project: Entity<Project>,
    fs: Arc<dyn Fs>,
}
```

**역할**:
- 세션 관리 및 조율
- 언어 모델 통합
- 프로젝트 컨텍스트 유지
- 템플릿 관리

### 2. Thread Management

**구조**:
```rust
pub struct Thread {
    messages: Vec<Message>,
    model: Arc<dyn LanguageModel>,
    project: Entity<Project>,
    tools: Vec<Box<dyn AgentTool>>,
    current_task: Option<Task<()>>,
}

pub enum Message {
    User(UserMessage),
    Agent(AgentMessage),
    Resume,
}
```

**역할**:
- 대화 흐름 제어
- 메시지 처리
- 도구 실행 관리
- 상태 추적

### 3. EditAgent

**구조**:
```rust
pub struct EditAgent {
    model: Arc<dyn LanguageModel>,
    project: Entity<Project>,
    action_log: Entity<ActionLog>,
    templates: Arc<Templates>,
    edit_format: EditFormat,
}

pub struct EditParser {
    streaming_diff: StreamingDiff,
    fuzzy_matcher: StreamingFuzzyMatcher,
    metrics: EditParserMetrics,
}
```

**역할**:
- 코드 편집 전문화
- 스트리밍 diff 처리
- 퍼지 매칭
- 편집 검증

### 4. Language Models

**구조**:
```rust
pub struct LanguageModels {
    models: HashMap<ModelId, Arc<dyn LanguageModel>>,
    model_list: AgentModelList,
    refresh_models_rx: watch::Receiver<()>,
}
```

**지원 프로바이더**:
- Zed Pro (호스팅 모델)
- Anthropic (Claude)
- OpenAI (GPT)
- Google AI (Gemini)
- Ollama (로컬)
- LM Studio (로컬)
- OpenRouter (게이트웨이)

## 도구 시스템

### 도구 아키텍처

```rust
pub trait AgentTool: Send + Sync {
    fn name() -> &'static str;
    fn description() -> &'static str;
    fn input_schema(format: LanguageModelToolSchemaFormat) -> Schema;
    fn supports_provider(provider: &LanguageModelProviderId) -> bool;
    async fn run(
        input: serde_json::Value,
        project: &Entity<Project>,
        worktree_id: Option<WorktreeId>,
        cx: &AsyncApp,
    ) -> Result<String>;
}
```

### 내장 도구 (17개)

#### 읽기 및 검색 도구

1. **read_file** (970 줄)
   - 파일 내용 읽기
   - 범위 지정 가능
   - 구문 강조 지원

2. **grep** (1,190 줄)
   - 정규식 기반 검색
   - 다중 파일 지원
   - 컨텍스트 라인 포함

3. **find_path** (253 줄)
   - Glob 패턴 매칭
   - 빠른 파일 찾기
   - 알파벳순 정렬

4. **list_directory** (662 줄)
   - 디렉터리 구조 탐색
   - 재귀적 나열
   - 파일 메타데이터

5. **diagnostics** (165 줄)
   - 오류 및 경고 수집
   - LSP 통합
   - 프로젝트 전체 또는 파일별

6. **fetch** (164 줄)
   - URL 컨텐츠 가져오기
   - Markdown 변환
   - 문서 참조용

7. **web_search** (132 줄)
   - 웹 검색 실행
   - 스니펫 및 링크
   - 실시간 정보 접근

8. **now** (64 줄)
   - 현재 날짜/시간
   - 시간 기반 작업

9. **open** (170 줄)
   - 파일/URL 열기
   - 시스템 기본 앱

10. **thinking** (52 줄)
    - 에이전트 사고 프로세스
    - 계획 수립
    - 중간 추론

#### 편집 도구

1. **edit_file** (1,762 줄)
   - 파일 편집
   - 텍스트 대체
   - 스트리밍 적용

2. **create_file** (edit_file에 포함)
   - 새 파일 생성
   - 전체 내용 작성

3. **delete_path** (140 줄)
   - 파일/디렉터리 삭제
   - 재귀적 삭제
   - 확인 메시지

4. **move_path** (124 줄)
   - 파일/디렉터리 이동
   - 이름 변경
   - 경로 업데이트

5. **copy_path** (113 줄)
   - 파일/디렉터리 복사
   - 재귀적 복사
   - 효율적 처리

6. **create_directory** (90 줄)
   - 디렉터리 생성
   - 부모 디렉터리 자동 생성
   - mkdir -p 동작

7. **terminal** (213 줄)
   - 셸 명령 실행
   - 출력 캡처
   - 새 프로세스

### 도구 확장 (MCP)

**Model Context Protocol** 통합:
- 외부 도구 추가
- 표준화된 인터페이스
- 익스텐션으로 배포

## 프로토콜 및 통신

### Agent Client Protocol (ACP)

**개요**:
- 외부 에이전트와의 표준 통신 프로토콜
- 양방향 메시지 교환
- 스트리밍 지원
- 도구 호출 인터페이스

**프로토콜 특징**:
```rust
pub struct AcpThread {
    // 프로토콜 메시지 처리
    message_handler: MessageHandler,
    // 스트림 처리
    stream_processor: StreamProcessor,
    // 도구 레지스트리
    tool_registry: ToolRegistry,
}
```

**지원 외부 에이전트**:

1. **Gemini CLI** (Google)
   - OAuth 인증
   - API 키 지원
   - Vertex AI 통합

2. **Claude Code** (Anthropic)
   - Claude Pro/Max 구독
   - API 키
   - CLAUDE.md 지원
   - 서브에이전트

3. **Codex** (OpenAI)
   - ChatGPT 구독
   - API 키
   - 다양한 인증 방법

4. **Custom Agents**
   - ACP 프로토콜 준수
   - 사용자 정의 구현

---

# Part II: 구현 상세

## 스레드 및 메시지 관리

### Message 타입

```rust
pub enum Message {
    User(UserMessage),
    Agent(AgentMessage),
    Resume,
}

pub struct UserMessage {
    id: UserMessageId,
    content: Vec<UserMessageContent>,
}

pub enum UserMessageContent {
    Text(SharedString),
    Image(LanguageModelImage),
    Mention { uri, content },
}

pub struct AgentMessage {
    chunks: Vec<AgentMessageChunk>,
    tool_uses: Vec<ToolUse>,
    status: MessageStatus,
}
```

### 메시지 처리

**사용자 메시지 처리**:
1. 입력 수집
2. 컨텍스트 추가 (멘션)
3. 시스템 프롬프트 구성
4. LLM 요청 생성

**에이전트 응답 처리**:
1. 스트리밍 수신
2. 청크 파싱
3. 도구 호출 실행
4. UI 업데이트

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

**오류 처리**:
- 네트워크 오류 → 자동 재시도
- 인증 오류 → 사용자 액션 필요
- 레이트 리밋 → 백오프 후 재시도
- 모델 오류 → 사용자에게 표시

## 컨텍스트 시스템

### 멘션 시스템

**지원 멘션 타입**:
```rust
pub enum MentionUri {
    File { abs_path },
    Directory { abs_path },
    Symbol { abs_path, line_range },
    Selection { abs_path, line_range },
    Thread { thread_id },
    TextThread { thread_id },
    Rule { path },
    Fetch { url },
    PastedImage,
}
```

**컨텍스트 구성**:
```xml
<files>
<!-- 파일 내용 -->
</files>

<directories>
<!-- 디렉터리 구조 -->
</directories>

<symbols>
<!-- 코드 심볼 -->
</symbols>

<selection>
<!-- 선택된 코드 -->
</selection>

<threads>
<!-- 이전 대화 -->
</threads>

<rules>
<!-- 규칙 파일 -->
</rules>

<fetch>
<!-- 웹 컨텐츠 -->
</fetch>
```

### 규칙 파일

**자동 인식 파일**:
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

**규칙 파일 구조**:
```markdown
# Project Rules

## Code Style
- Indentation: 4 spaces
- Max line length: 100
- Naming: descriptive

## Architecture
- Follow MVC pattern
- Keep business logic separate
- Use dependency injection

## Testing
- Unit tests for all features
- >80% coverage
- Descriptive test names
```

### ProjectContext

```rust
pub struct ProjectContext {
    worktree_snapshots: Vec<WorktreeSnapshot>,
    rules_context: RulesFileContext,
    user_rules: UserRulesContext,
}
```

**자동 수집 정보**:
- 프로젝트 구조
- 파일 트리
- 규칙 파일
- 사용자 설정

## 편집 에이전트

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

**1. XML 형식**:
```xml
<edit>
  <path>src/main.rs</path>
  <old>
fn old_function() {
    // old code
}
  </old>
  <new>
fn new_function() {
    // new code
}
  </new>
</edit>
```

**2. Diff Fenced 형식**:
````markdown
```diff src/main.rs
- old line
+ new line
```
````

### 편집 파서

```rust
pub struct EditParser {
    streaming_diff: StreamingDiff,
    fuzzy_matcher: StreamingFuzzyMatcher,
    metrics: EditParserMetrics,
}
```

**편집 프로세스**:
1. **범위 해결**: 편집할 위치 찾기
2. **퍼지 매칭**: 유사한 패턴 찾기
3. **스트리밍 적용**: 실시간 변경
4. **검증**: 구문 오류 확인

### 편집 이벤트

```rust
pub enum EditAgentOutputEvent {
    ResolvingEditRange(Range<Anchor>),
    UnresolvedEditRange,
    AmbiguousEditRange(Vec<Range<usize>>),
    Edited(Range<Anchor>),
}
```

## 프로파일 시스템

### 내장 프로파일

**1. Write Profile**:
```json
{
  "name": "Write",
  "tools": [
    "read_file", "edit_file", "create_file",
    "delete_path", "move_path", "copy_path",
    "create_directory", "grep", "find_path",
    "list_directory", "diagnostics", "terminal",
    "web_search", "fetch", "now", "open", "thinking"
  ]
}
```

**2. Ask Profile**:
```json
{
  "name": "Ask",
  "tools": [
    "read_file", "grep", "find_path",
    "list_directory", "diagnostics",
    "web_search", "fetch", "now", "open", "thinking"
  ]
}
```

**3. Minimal Profile**:
```json
{
  "name": "Minimal",
  "tools": []
}
```

### 커스텀 프로파일

**설정 구조**:
```rust
pub struct AgentProfile {
    pub name: String,
    pub description: Option<String>,
    pub tools: Vec<String>,
}
```

**프로파일 관리**:
- UI를 통한 생성/편집
- JSON 설정 파일
- 내장 프로파일 오버라이드

## 데이터 영속성

### 데이터베이스 스키마

**SQLite 구조**:

```sql
-- 스레드 테이블
CREATE TABLE threads (
    session_id TEXT PRIMARY KEY,
    timestamp INTEGER,
    model TEXT,
    profile TEXT
);

-- 메시지 테이블
CREATE TABLE messages (
    id INTEGER PRIMARY KEY,
    thread_id TEXT,
    role TEXT,
    content TEXT,
    timestamp INTEGER,
    FOREIGN KEY (thread_id) REFERENCES threads(session_id)
);

-- 도구 사용 테이블
CREATE TABLE tool_uses (
    id INTEGER PRIMARY KEY,
    message_id INTEGER,
    tool_name TEXT,
    input TEXT,
    output TEXT,
    FOREIGN KEY (message_id) REFERENCES messages(id)
);

-- 체크포인트 테이블
CREATE TABLE checkpoints (
    id INTEGER PRIMARY KEY,
    message_id INTEGER,
    state TEXT,
    timestamp INTEGER,
    FOREIGN KEY (message_id) REFERENCES messages(id)
);

-- 토큰 사용량 테이블
CREATE TABLE token_usage (
    id INTEGER PRIMARY KEY,
    message_id INTEGER,
    input_tokens INTEGER,
    output_tokens INTEGER,
    cache_tokens INTEGER,
    FOREIGN KEY (message_id) REFERENCES messages(id)
);
```

### HistoryStore

```rust
pub struct HistoryStore {
    db: Arc<AgentDb>,
    sessions: HashMap<SessionId, ThreadSnapshot>,
}
```

**기능**:
- 대화 히스토리 저장
- 체크포인트 관리
- 빠른 검색
- 효율적 로딩

### 체크포인트 시스템

**작동 방식**:
1. 편집 전 자동 스냅샷
2. 프로젝트 상태 저장
3. 롤백 기능 제공
4. UI에서 복원 가능

---

# Part III: 실전 가이드

## 기본 사용법

### 1. 에이전트 시작하기

**초기화 코드**:
```rust
// 에이전트 생성
let agent = NativeAgent::new(
    project,
    history_store,
    templates,
    prompt_store,
    fs,
    &mut cx
).await?;

// 새 스레드 시작
let thread = agent.new_thread(model, profile, &mut cx);
```

### 2. 메시지 전송

**사용자 메시지 구성**:
```rust
let message = UserMessage {
    id: UserMessageId::new(),
    content: vec![
        UserMessageContent::Text("분석해주세요".into()),
        UserMessageContent::Mention {
            uri: MentionUri::File {
                abs_path: PathBuf::from("/path/to/file.rs")
            },
            content: file_content,
        },
    ],
};

thread.send_message(message, cx);
```

### 3. 컨텍스트 추가

**멘션 사용**:
- `@파일명` - 파일 내용 포함
- `@디렉터리/` - 디렉터리 구조
- `@심볼명` - 특정 함수/클래스
- `@선택영역` - 현재 선택
- `@스레드` - 이전 대화

**예제**:
```
@src/main.rs의 main 함수를 분석하고,
@tests/ 디렉터리의 테스트를 참고하여
새 테스트를 추가해주세요.
```

### 4. 응답 처리

**스트리밍 수신**:
```rust
let mut stream = model.stream_completion(request).await?;

while let Some(event) = stream.next().await {
    match event? {
        LanguageModelCompletionEvent::Text(text) => {
            // UI 업데이트
            update_ui(text);
        }
        LanguageModelCompletionEvent::ToolUse(tool_use) => {
            // 도구 실행
            let result = execute_tool(&tool_use).await?;
            add_tool_result(result);
        }
        LanguageModelCompletionEvent::Stop(_) => break,
    }
}
```

## 커스텀 도구 구현

### 도구 구조

```rust
use schemars::{JsonSchema, schema_for};
use serde::{Deserialize, Serialize};

#[derive(Debug, Deserialize, Serialize, JsonSchema)]
pub struct MyToolInput {
    /// 처리할 파일 경로
    pub path: String,
    /// 변환 타입
    pub transform_type: TransformType,
}

#[derive(Debug, Deserialize, Serialize, JsonSchema)]
pub enum TransformType {
    #[serde(rename = "uppercase")]
    Uppercase,
    #[serde(rename = "lowercase")]
    Lowercase,
}
```

### 도구 구현

```rust
pub struct MyCustomTool;

impl AgentTool for MyCustomTool {
    fn name() -> &'static str {
        "my_custom_tool"
    }

    fn description() -> &'static str {
        "텍스트 변환 도구. 파일 내용을 다양한 형식으로 변환합니다."
    }

    fn input_schema(format: LanguageModelToolSchemaFormat) -> Schema {
        match format {
            LanguageModelToolSchemaFormat::JsonSchema => {
                schema_for!(MyToolInput).schema.into()
            }
        }
    }

    fn supports_provider(provider: &LanguageModelProviderId) -> bool {
        true // 모든 프로바이더 지원
    }

    async fn run(
        input: serde_json::Value,
        project: &Entity<Project>,
        worktree_id: Option<WorktreeId>,
        cx: &AsyncApp,
    ) -> Result<String> {
        let input: MyToolInput = serde_json::from_value(input)?;
        
        // 파일 읽기
        let content = read_file(&input.path)?;
        
        // 변환 적용
        let transformed = match input.transform_type {
            TransformType::Uppercase => content.to_uppercase(),
            TransformType::Lowercase => content.to_lowercase(),
        };
        
        Ok(format!("변환 완료:\n{}", transformed))
    }
}
```

### 도구 등록

```rust
// tools.rs에 추가
mod my_custom_tool;
pub use my_custom_tool::*;

tools! {
    // ... 기존 도구들
    MyCustomTool,
}
```

## 프로파일 설정

### 설정 파일 예제

```json
{
  "agent": {
    "version": "2",
    "default_model": {
      "provider": "zed",
      "name": "claude-sonnet-4-20250514"
    },
    "profiles": {
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
      "project-manager": {
        "name": "Project Manager",
        "description": "프로젝트 구조 관리용",
        "tools": [
          "list_directory",
          "create_directory",
          "move_path",
          "copy_path",
          "delete_path",
          "find_path"
        ]
      },
      "fullstack-dev": {
        "name": "Full Stack Developer",
        "description": "모든 도구 사용",
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
    "always_allow_tool_actions": false,
    "notify_when_agent_waiting": true,
    "play_sound_when_agent_done": true
  }
}
```

### 프로파일 로딩

```rust
pub fn load_profiles(config_path: &Path) -> Result<HashMap<String, AgentProfile>> {
    let config = fs::read_to_string(config_path)?;
    let settings: Settings = serde_json::from_str(&config)?;
    Ok(settings.agent.profiles)
}
```

## 외부 에이전트 통합

### ACP 에이전트 설정

```json
{
  "agent_servers": {
    "gemini": {
      "command": "gemini",
      "args": ["--acp"],
      "env": {
        "GEMINI_API_KEY": "your-key"
      }
    },
    "claude": {
      "command": "claude-code",
      "args": ["--acp"],
      "env": {
        "CLAUDE_CODE_EXECUTABLE": "/path/to/claude"
      }
    },
    "custom": {
      "command": "node",
      "args": ["/path/to/agent/index.js", "--acp"],
      "env": {
        "API_KEY": "your-key"
      }
    }
  }
}
```

### ACP 프로토콜 구현

```rust
// 커스텀 ACP 에이전트 예제
pub struct CustomAcpAgent {
    stdin: ChildStdin,
    stdout: BufReader<ChildStdout>,
}

impl CustomAcpAgent {
    pub async fn handle_message(&mut self, msg: AcpMessage) -> Result<()> {
        match msg {
            AcpMessage::Request { id, method, params } => {
                let result = self.process_request(method, params).await?;
                self.send_response(id, result).await?;
            }
            AcpMessage::Notification { method, params } => {
                self.process_notification(method, params).await?;
            }
            _ => {}
        }
        Ok(())
    }
}
```

### MCP 서버 통합

```json
{
  "context_servers": {
    "filesystem": {
      "command": "npx",
      "args": [
        "-y",
        "@modelcontextprotocol/server-filesystem",
        "/allowed/path"
      ]
    },
    "git": {
      "command": "npx",
      "args": ["-y", "@modelcontextprotocol/server-git"]
    },
    "postgres": {
      "command": "npx",
      "args": ["-y", "@modelcontextprotocol/server-postgres"],
      "env": {
        "DATABASE_URL": "postgresql://localhost/db"
      }
    },
    "custom": {
      "command": "python",
      "args": ["/path/to/mcp_server.py"],
      "env": {
        "API_KEY": "your-key"
      }
    }
  }
}
```

## 고급 패턴

### 1. 규칙 파일 활용

**.rules 파일**:
```markdown
# Project Rules

## Code Style
- 4 spaces indentation
- Max 100 characters per line
- Descriptive variable names

## Architecture
- Follow /docs/architecture.md
- Business logic in /src/core/
- UI components in /src/ui/

## Testing
- Unit tests for all features
- >80% coverage
- Descriptive test names

## Documentation
- JSDoc for public APIs
- Update README for new features
- Keep CHANGELOG current

## Git Workflow
- Conventional commits
- Feature branches from main
- Squash before merge
```

### 2. 체크포인트 활용

```rust
// 체크포인트 생성
pub fn create_checkpoint(
    thread: &Thread,
    cx: &mut App
) -> Result<CheckpointId> {
    let state = capture_project_state(cx)?;
    let checkpoint_id = thread.save_checkpoint(state, cx)?;
    Ok(checkpoint_id)
}

// 체크포인트 복원
pub fn restore_checkpoint(
    thread: &mut Thread,
    checkpoint_id: CheckpointId,
    cx: &mut App
) -> Result<()> {
    let state = thread.load_checkpoint(checkpoint_id, cx)?;
    apply_project_state(state, cx)?;
    Ok(())
}
```

### 3. 토큰 최적화

**효율적인 컨텍스트**:
```rust
// 나쁜 예: 전체 프로젝트
add_context("@entire_project/");

// 좋은 예: 필요한 파일만
add_context("@src/auth/login.rs");
add_context("@tests/auth_tests.rs");
```

**컨텍스트 우선순위**:
1. 직접 관련 파일
2. 테스트 파일
3. 문서/규칙
4. 주변 컨텍스트

### 4. 스트리밍 처리

```rust
pub async fn handle_streaming_response(
    mut stream: impl Stream<Item = Result<Event>>,
    thread: &mut Thread,
    cx: &mut App
) -> Result<()> {
    while let Some(event) = stream.next().await {
        match event? {
            Event::Text(text) => {
                thread.append_text(text, cx);
            }
            Event::ToolUse(tool_use) => {
                let result = execute_tool_async(&tool_use, cx).await?;
                thread.add_tool_result(result, cx);
            }
            Event::Stop(reason) => {
                thread.finalize(reason, cx);
                break;
            }
        }
    }
    Ok(())
}
```

### 5. 에러 처리 및 재시도

```rust
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
                        sleep(delay).await;
                        delay *= 2;
                    }
                }
            }
        }
        RetryStrategy::Fixed { delay, max_attempts } => {
            for attempt in 0..max_attempts {
                match f().await {
                    Ok(result) => return Ok(result),
                    Err(e) if attempt == max_attempts - 1 => return Err(e),
                    Err(_) => sleep(delay).await,
                }
            }
            unreachable!()
        }
    }
}
```

### 6. 커스텀 템플릿

```rust
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

Provide a detailed solution.
```

### 7. 병렬 처리

```rust
pub async fn process_multiple_files(
    files: Vec<PathBuf>,
    tool: &dyn AgentTool,
    cx: &AsyncApp,
) -> Result<Vec<String>> {
    let tasks: FuturesUnordered<_> = files
        .into_iter()
        .map(|file| {
            let input = serde_json::json!({ "path": file });
            tool.run(input, project, None, cx)
        })
        .collect();
    
    tasks.try_collect().await
}
```

### 8. 진단 정보 활용

```rust
pub async fn fix_diagnostics(
    thread: &mut Thread,
    cx: &mut App
) -> Result<()> {
    // 1. 진단 정보 수집
    let diagnostics = collect_diagnostics(cx)?;
    
    // 2. 에이전트에 전달
    let message = format!(
        "다음 오류들을 수정해주세요:\n{}",
        format_diagnostics(&diagnostics)
    );
    
    // 3. 수정 수행
    thread.send_message(message, cx).await?;
    
    // 4. 재검증
    let new_diagnostics = collect_diagnostics(cx)?;
    
    if new_diagnostics.is_empty() {
        println!("모든 오류 수정 완료!");
    }
    
    Ok(())
}
```

### 9. 웹 리서치 통합

```rust
pub async fn research_and_apply(
    query: &str,
    file_path: &Path,
    thread: &mut Thread,
    cx: &mut App
) -> Result<()> {
    // 1. 웹 검색
    let search_results = web_search(query).await?;
    
    // 2. 관련 문서 가져오기
    let mut docs = Vec::new();
    for result in search_results.iter().take(3) {
        if let Ok(content) = fetch_url(&result.url).await {
            docs.push(content);
        }
    }
    
    // 3. 파일 읽기
    let file_content = read_file(file_path)?;
    
    // 4. 컨텍스트와 함께 요청
    let message = format!(
        "최신 베스트 프랙티스를 적용하여 리팩토링:\n\n\
         참고 문서:\n{}\n\n\
         파일:\n{}",
        docs.join("\n---\n"),
        file_content
    );
    
    thread.send_message(message, cx).await?;
    Ok(())
}
```

### 10. 실전 워크플로우

#### 새 기능 추가

```rust
pub async fn add_feature_workflow(
    description: &str,
    thread: &mut Thread,
    cx: &mut App
) -> Result<()> {
    // 1. 현재 구조 파악
    let structure = list_directory("src/", cx)?;
    
    // 2. 관련 파일 찾기
    let related_files = find_related_files(description, cx)?;
    
    // 3. 컨텍스트 구성
    let mut context = vec![
        format!("프로젝트 구조:\n{}", structure),
    ];
    for file in related_files {
        context.push(format!("@{}", file.display()));
    }
    
    // 4. 요청 전송
    let message = format!(
        "{}\n\n참고:\n{}",
        description,
        context.join("\n")
    );
    thread.send_message(message, cx).await?;
    
    // 5. 테스트 실행
    run_tests(cx).await?;
    
    // 6. 진단 확인
    let diagnostics = collect_diagnostics(cx)?;
    if !diagnostics.is_empty() {
        fix_diagnostics(thread, cx).await?;
    }
    
    Ok(())
}
```

#### 코드 리뷰

```rust
pub async fn review_changes(
    files: Vec<PathBuf>,
    thread: &mut Thread,
    cx: &mut App
) -> Result<String> {
    let mut review = String::new();
    
    for file in files {
        let diff = get_file_diff(&file, cx)?;
        
        let message = format!(
            "다음 변경사항을 리뷰해주세요:\n{}",
            diff
        );
        
        let response = thread.send_and_wait(message, cx).await?;
        review.push_str(&format!("\n## {}\n{}\n", file.display(), response));
    }
    
    Ok(review)
}
```

---

## 참고 자료

### 공식 문서
- Agent Client Protocol: https://agentclientprotocol.com
- Model Context Protocol: https://modelcontextprotocol.io

### 관련 프로젝트
- Zed Editor: https://github.com/zed-industries/zed
- Gemini CLI: https://github.com/google-gemini/gemini-cli
- Claude Code: https://www.anthropic.com/claude-code

### 추가 리소스
- Rust Async Book: https://rust-lang.github.io/async-book/
- SQLite Documentation: https://www.sqlite.org/docs.html
- JSON Schema: https://json-schema.org/

---

## 라이선스

이 문서는 GPL-3.0-or-later 라이선스 하에 배포됩니다.

**문서 버전**: 1.0
**최종 업데이트**: 2025년 11월
**작성자**: AI Agent Documentation Team

---

**완료! 이 문서는 다른 코드베이스에서 AI 에이전트 시스템을 이해하고 구현하는 데 활용할 수 있습니다.**
