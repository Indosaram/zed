# Zed AI Agent Feature Analysis

## Overview

This document provides a comprehensive technical analysis of Zed editor's AI agent functionality. Zed's agent system leverages large language models (LLMs) to interact with codebases, perform autonomous edits, and execute various tasks through a powerful set of tools.

## Architecture

### Core Crates

Zed's agent functionality is organized into several independent crates, each with clear responsibilities:

#### 1. `agent` Crate
**Location**: `crates/agent/`
**Role**: Core agent logic and business logic

**Key Components**:
- `agent.rs` (1,636 lines) - Main agent entry point and orchestration
- `thread.rs` (2,640 lines) - Conversation thread management and message processing
- `edit_agent.rs` (1,491 lines) - Code editing agent
- `tools/` - 14 built-in tool implementations
- `db.rs` - Database persistence
- `history_store.rs` - Conversation history management
- `native_agent_server.rs` - Native agent server implementation

**Tool System** (17 tools total):
```
read_file_tool.rs       (970 lines)  - Read file contents
edit_file_tool.rs       (1,762 lines) - Edit files
grep_tool.rs            (1,190 lines) - Search code
list_directory_tool.rs  (662 lines)  - List directory contents
find_path_tool.rs       (253 lines)  - Find file paths
terminal_tool.rs        (213 lines)  - Execute terminal commands
diagnostics_tool.rs     (165 lines)  - Get diagnostics
fetch_tool.rs           (164 lines)  - Fetch URLs
open_tool.rs            (170 lines)  - Open files/URLs
delete_path_tool.rs     (140 lines)  - Delete paths
web_search_tool.rs      (132 lines)  - Search the web
move_path_tool.rs       (124 lines)  - Move paths
copy_path_tool.rs       (113 lines)  - Copy paths
create_directory_tool.rs (90 lines)  - Create directories
now_tool.rs             (64 lines)   - Get current time
thinking_tool.rs        (52 lines)   - Thinking process
context_server_registry.rs (253 lines) - MCP server integration
```

#### 2. `agent_ui` Crate
**Location**: `crates/agent_ui/`
**Role**: User interface and interactions

**Key Components**:
- `agent_panel.rs` - Main agent panel UI
- `agent_model_selector.rs` - Model selection UI
- `agent_configuration.rs` - Configuration UI
- `agent_diff.rs` - Change comparison UI
- `inline_assistant.rs` - Inline assistant
- `terminal_inline_assistant.rs` - Terminal inline assistant
- `context_picker.rs` - Context picker
- `slash_command.rs` - Slash command system

#### 3. `agent_servers` Crate
**Location**: `crates/agent_servers/`
**Role**: External agent server integration

**Supported Agents**:
- Gemini CLI (Google)
- Claude Code (Anthropic)
- Codex (OpenAI)
- Custom ACP-compatible agents

#### 4. `agent_settings` Crate
**Location**: `crates/agent_settings/`
**Role**: Settings and profile management

**Features**:
- Agent profiles (Write, Ask, Minimal)
- Tool permission settings
- Model selection settings
- Custom rules

#### 5. `acp_thread` Crate
**Location**: `crates/acp_thread/`
**Role**: Agent Client Protocol (ACP) implementation

**Features**:
- ACP protocol message handling
- External agent communication
- Stream processing

## Agent Client Protocol (ACP)

### Overview
ACP is a protocol that enables Zed to communicate with external terminal-based agents. This allows external agents like Gemini CLI, Claude Code, and Codex to be used within the Zed UI.

### Protocol Features
- **Standard Interface**: All ACP-compatible agents use the same protocol
- **Bidirectional Communication**: Real-time message exchange between Zed ↔ external agents
- **Tool Calling**: Agents can invoke Zed's tools
- **Streaming**: Real-time response streaming support

### Supported External Agents

#### Gemini CLI
- Google's official Gemini CLI tool
- OAuth and API key authentication support
- Vertex AI integration available
- Automatic installation and updates

#### Claude Code
- Anthropic's Claude Code agent
- Use Claude Pro/Max subscription or API key
- Automatic CLAUDE.md file recognition
- Subagent support
- Custom slash commands

#### Codex CLI
- OpenAI's Codex agent
- ChatGPT subscription or API key
- Multiple authentication methods

## Tools System

### Tool Categories

#### Read & Search Tools
1. **read_file** - Read file contents
2. **grep** - Search code using regex
3. **find_path** - Find files using glob patterns
4. **list_directory** - List directory contents
5. **diagnostics** - Get errors and warnings
6. **fetch** - Fetch content from URLs
7. **web_search** - Search the web
8. **now** - Get current date/time
9. **open** - Open files/URLs
10. **thinking** - Agent's thinking process

#### Edit Tools
1. **edit_file** - Edit files (text replacement)
2. **create_file** - Create new files
3. **delete_path** - Delete files/directories
4. **move_path** - Move files/directories
5. **copy_path** - Copy files/directories
6. **create_directory** - Create directories
7. **terminal** - Execute shell commands

### Tool Approval System
- `agent.always_allow_tool_actions` setting
- Default: false (approval required)
- Set to true for automatic approval

### Model Compatibility
Not all tools work with all models. Zed checks:
- Model's tool calling support
- Model provider support for specific tools
- Zed's hosted models support all tools

## Thread & Message Management

### Thread Structure
```rust
pub struct Thread {
    // List of conversation messages
    messages: Vec<Message>,
    // Language model in use
    model: Arc<dyn LanguageModel>,
    // Project reference
    project: Entity<Project>,
    // Tool registry
    tools: Vec<Box<dyn AgentTool>>,
    // Running task
    current_task: Option<Task<()>>,
}
```

### Message Types
```rust
pub enum Message {
    User(UserMessage),      // User messages
    Agent(AgentMessage),    // Agent responses
    Resume,                 // Resume marker
}
```

### UserMessage Structure
```rust
pub struct UserMessage {
    id: UserMessageId,
    content: Vec<UserMessageContent>,
}

pub enum UserMessageContent {
    Text(SharedString),                // Text
    Image(LanguageModelImage),         // Images
    Mention { uri, content },          // Mentions (@file, @symbol, etc.)
}
```

### AgentMessage Structure
```rust
pub struct AgentMessage {
    chunks: Vec<AgentMessageChunk>,    // Response chunks
    tool_uses: Vec<ToolUse>,           // Tools used
    status: MessageStatus,             // Message status
}
```

## Context System

### Mention System
Users can add various contexts using the `@` symbol:

1. **@file** - Include file contents
2. **@directory** - Include directory structure
3. **@symbol** - Specific code symbols
4. **@selection** - Currently selected text
5. **@thread** - Previous conversation threads
6. **@rules** - Rules files (.rules, CLAUDE.md, etc.)
7. **@fetch** - Web URL contents

### Rules Files
Zed automatically recognizes these rules files:
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
    // Project snapshots
    worktree_snapshots: Vec<WorktreeSnapshot>,
    // Rules context
    rules_context: RulesFileContext,
    // User rules
    user_rules: UserRulesContext,
}
```

## Edit Agent

### EditAgent Structure
```rust
pub struct EditAgent {
    model: Arc<dyn LanguageModel>,
    project: Entity<Project>,
    action_log: Entity<ActionLog>,
    templates: Arc<Templates>,
    edit_format: EditFormat,
}
```

### Edit Formats
- **XML Format** - Specify edit locations using XML tags
- **Diff Fenced Format** - Use diff blocks for editing

### Edit Parser
```rust
pub struct EditParser {
    // Streaming diff processing
    streaming_diff: StreamingDiff,
    // Fuzzy matching
    fuzzy_matcher: StreamingFuzzyMatcher,
    // Parser metrics
    metrics: EditParserMetrics,
}
```

### Edit Process
1. **Resolve Edit Range** - Find where to edit
2. **Fuzzy Matching** - Find similar code patterns
3. **Streaming Diff** - Apply changes in real-time
4. **Validation** - Check for syntax errors

## Profile System

### Built-in Profiles

#### Write Profile
- **Purpose**: Allow write operations to codebase
- **Tools**: All read + all edit tools
- **Use Cases**: Code generation, refactoring, file manipulation

#### Ask Profile
- **Purpose**: Read-only operations
- **Tools**: Read and search tools only
- **Use Cases**: Codebase exploration, answering questions

#### Minimal Profile
- **Purpose**: General conversation
- **Tools**: No tools
- **Use Cases**: General LLM conversation

### Custom Profiles
Users can create custom profiles through:
- Profile creation/editing via UI
- Manual configuration in `settings.json`
- Forking existing profiles
- Overriding built-in profiles

### Profile Configuration Example
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

### Overview
MCP is a standardized way to add external tools to the agent.

### Integration Method
1. **Install MCP Server** - Install as Zed extension
2. **Auto Registration** - Register with ContextServerRegistry
3. **Tool Exposure** - Agent can use MCP tools

### Support Status
- **Zed Native Agent**: Full support
- **Claude Code**: Supported
- **Codex**: Supported
- **Gemini CLI**: Partial support (see documentation)

## Database & Persistence

### History Storage
```rust
pub struct HistoryStore {
    db: Arc<AgentDb>,
    sessions: HashMap<SessionId, ThreadSnapshot>,
}
```

### Database Schema
- **threads** - Conversation threads
- **messages** - Message contents
- **tool_uses** - Tool usage records
- **snapshots** - Project snapshots

### Checkpoint System
- Automatic checkpoint creation before each edit
- Can restore to previous state anytime
- Access via "Restore Checkpoint" button in UI

## Language Model Integration

### Supported Providers
- **Zed Pro** - Zed's hosted models
- **Anthropic** - Claude models
- **OpenAI** - GPT models
- **Google AI** - Gemini models
- **Ollama** - Local models
- **OpenRouter** - Various model gateway
- **LM Studio** - Local LLM server

### Model Selection
```rust
pub struct LanguageModels {
    models: HashMap<ModelId, Arc<dyn LanguageModel>>,
    model_list: AgentModelList,
}
```

### Model Authentication
- API key-based
- OAuth-based (Gemini)
- Subscription-based (Zed Pro)

## Token Usage Management

### Usage Tracking
```rust
pub struct TokenUsage {
    input_tokens: u64,
    output_tokens: u64,
    cache_creation_tokens: Option<u64>,
    cache_read_tokens: Option<u64>,
}
```

### UI Display
- Display token usage for current thread
- Warning when approaching context window
- Suggest thread summarization and new thread creation

## Error Handling & Retry

### Retry Strategy
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

### Error Types
- Network errors - Automatic retry
- Authentication errors - Require user action
- Rate limits - Retry after backoff
- Model errors - Display to user

## Collaboration Features

### Agent Following
- Follow what the agent reads and edits in real-time
- Activate via crosshair icon or `Cmd/Ctrl + Enter`
- Leverages Zed's native collaboration features

### Notifications
```json
{
  "agent": {
    "notify_when_agent_waiting": true,
    "play_sound_when_agent_done": true
  }
}
```

## Feedback & Improvement

### Rating System
- Thumbs up/down buttons
- Detailed feedback text
- Sent to Zed servers (opt-in)

### Privacy
- No data collection without rating
- Only applies to Zed Pro models
- External agents have separate privacy policies

## Performance Optimizations

### Streaming
- Real-time response streaming
- Chunk-based processing
- Optimized UI updates

### Caching
- Project context caching
- Model list caching
- Token usage caching

### Asynchronous Processing
```rust
cx.spawn(async move |this, cx| {
    // Background work
})
```

## Testing

### Test Structure
- `crates/agent/src/tests/` - Integration tests
- Unit tests for each tool
- EditAgent evaluation tests

### Test Tools
```rust
#[cfg(test)]
mod tests {
    use super::*;
    
    #[gpui::test]
    async fn test_agent_functionality(cx: &mut App) {
        // Test code
    }
}
```

## Key APIs & Usage Examples

### Create New Thread
```rust
// Create thread with native agent
agent::NewThread

// Create thread with external agent
agent::NewExternalAgentThread { agent: "gemini" }
```

### Send Message
```rust
thread.send_message(user_message, cx)
```

### Register Tool
```rust
impl AgentTool for MyCustomTool {
    fn name() -> &'static str {
        "my_custom_tool"
    }
    
    fn description() -> &'static str {
        "My custom tool description"
    }
    
    fn input_schema(format: LanguageModelToolSchemaFormat) -> Schema {
        // Schema definition
    }
    
    async fn run(input: serde_json::Value, cx: &AsyncApp) -> Result<String> {
        // Tool logic
    }
}
```

## Configuration Examples

### Basic Configuration
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

### External Agent Configuration
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

## Development Guide

### Adding New Tools
1. Create new file in `crates/agent/src/tools/`
2. Implement `AgentTool` trait
3. Add to `tools!` macro in `tools.rs`

### Adding New Agent Provider
1. Implement in `crates/agent_servers/src/`
2. Comply with ACP protocol
3. Define configuration schema

### Build & Test
```bash
# Run clippy
./script/clippy -p agent

# Run tests
cargo test -p agent

# Run specific test
cargo test -p agent --test test_name
```

## Future Improvements

### Planned Features
- [ ] More external agent support
- [ ] Enhanced checkpoint system
- [ ] Better diff visualization
- [ ] Enhanced multimodal support
- [ ] More MCP server integrations

### Limitations
- Some features limited for external agents (message editing, history restoration, etc.)
- Tool support varies by model
- Token limitations

## Conclusion

Zed's AI agent system is a comprehensive and extensible architecture with the following characteristics:

1. **Modular Design** - Each component is clearly separated
2. **Extensibility** - Easy to add tools, profiles, and external agents
3. **User-Friendly** - Intuitive UI and configuration
4. **Powerful Features** - 17 built-in tools + MCP support
5. **Privacy-Focused** - Opt-in data collection
6. **Provider-Neutral** - Support for various LLM providers

This system enables efficient execution of various tasks including code editing, refactoring, codebase exploration, and general inquiries.

## References

- [Zed AI Documentation](https://zed.dev/docs/ai)
- [Agent Client Protocol](https://agentclientprotocol.com)
- [Zed GitHub Repository](https://github.com/zed-industries/zed)
- [Model Context Protocol](https://modelcontextprotocol.io)

---

**Document Version**: 1.0
**Created**: November 2025
**License**: GPL-3.0-or-later
