# Gemini CLI - Architecture & Flow Analysis

## Table of Contents
1. [High-Level Architecture](#high-level-architecture)
2. [Message & Invocation Flow](#message--invocation-flow)
3. [Key Components](#key-components)
4. [Design Decisions](#design-decisions)
5. [Package Structure](#package-structure)
6. [Detailed Flow Diagrams](#detailed-flow-diagrams)

---

## High-Level Architecture

Gemini CLI is a monorepo-based TypeScript application that provides an interactive terminal interface for communicating with Google's Gemini AI models.

```mermaid
graph TB
    subgraph "User Interface Layer"
        CLI[CLI Entry Point<br/>packages/cli/index.ts]
        TUI[Terminal UI<br/>React + Ink]
        NonInt[Non-Interactive Mode]
    end

    subgraph "Core Layer"
        Config[Config Manager]
        GeminiChat[GeminiChat<br/>Conversation Handler]
        ContentGen[ContentGenerator<br/>API Abstraction]
        Scheduler[Scheduler<br/>Tool Orchestrator]
        ToolReg[Tool Registry]
        AgentReg[Agent Registry]
        PolicyEng[Policy Engine]
    end

    subgraph "External Services"
        GeminiAPI[Google Gemini API]
        VertexAI[Vertex AI]
        MCP[MCP Servers]
        IDE[IDE Integration]
    end

    CLI --> TUI
    CLI --> NonInt
    TUI --> GeminiChat
    NonInt --> GeminiChat
    GeminiChat --> ContentGen
    ContentGen --> GeminiAPI
    ContentGen --> VertexAI
    GeminiChat --> Scheduler
    Scheduler --> ToolReg
    Scheduler --> PolicyEng
    ToolReg --> MCP
    TUI --> IDE

    style CLI fill:#e1f5ff
    style GeminiChat fill:#fff4e1
    style Scheduler fill:#ffe1f5
    style GeminiAPI fill:#e1ffe1
```

---

## Message & Invocation Flow

### User Prompt → LLM Response Flow

```mermaid
sequenceDiagram
    participant User
    participant CLI as CLI Entry
    participant PromptProc as Prompt Processors
    participant GeminiChat
    participant ContentGen as Content Generator
    participant LLM as Gemini API
    participant Scheduler
    participant Tools

    User->>CLI: Input prompt
    CLI->>PromptProc: Process input
    Note over PromptProc: Handle @file references<br/>Shell command injection<br/>Argument processing
    PromptProc->>GeminiChat: Formatted prompt
    GeminiChat->>ContentGen: generateContentStream()
    ContentGen->>LLM: HTTP/gRPC Request

    loop Streaming Response
        LLM-->>ContentGen: Stream chunk
        ContentGen-->>GeminiChat: GenerateContentResponse
        GeminiChat-->>CLI: Display chunk
        CLI-->>User: Render in TUI
    end

    alt Contains Tool Calls
        GeminiChat->>Scheduler: Execute tool calls
        Scheduler->>Tools: Invoke tools
        Tools-->>Scheduler: Tool results
        Scheduler-->>GeminiChat: Tool responses
        GeminiChat->>ContentGen: Continue with tool results
        ContentGen->>LLM: Next turn with results
    end

    LLM-->>ContentGen: Final response
    ContentGen-->>GeminiChat: Complete
    GeminiChat-->>CLI: Final output
    CLI-->>User: Display result
```

---

## Key Components

### 1. Entry Point & Initialization
**File:** `packages/cli/index.ts:9` → `packages/cli/src/gemini.tsx:318`

The main entry point handles:
- Global error handling
- Process cleanup
- Delegates to `main()` function

```typescript
// Simplified flow
main() {
  1. Load settings & configuration
  2. Parse CLI arguments
  3. Setup authentication
  4. Handle sandbox (if enabled)
  5. Initialize app (tools, agents, policies)
  6. Start UI (interactive) OR run non-interactive
}
```

### 2. GeminiChat - Conversation Manager
**File:** `packages/core/src/core/geminiChat.ts`

Manages the conversation lifecycle:
- Maintains conversation history
- Handles streaming responses
- Implements retry logic for failures
- Validates content from LLM
- Manages turn compression

**Key Methods:**
- `sendMessage()` - Send user message to LLM
- `sendMessageStream()` - Stream LLM responses
- `extractCuratedHistory()` - Clean invalid turns from history

### 3. ContentGenerator - API Abstraction
**File:** `packages/core/src/core/contentGenerator.ts`

Abstracts authentication methods:
- **OAuth (Login with Google)** - `AuthType.LOGIN_WITH_GOOGLE`
- **API Key** - `AuthType.USE_GEMINI`
- **Vertex AI** - `AuthType.USE_VERTEX_AI`
- **Compute Default Credentials** - `AuthType.COMPUTE_ADC`

```mermaid
graph LR
    CG[ContentGenerator]

    CG -->|OAuth| GCA[Google Code Assist]
    CG -->|API Key| GeminiAPI[Gemini API]
    CG -->|ADC| VertexAI[Vertex AI]

    GCA --> Logging[LoggingContentGenerator]
    GeminiAPI --> Logging
    VertexAI --> Logging
```

### 4. Scheduler - Tool Execution Orchestrator
**File:** `packages/core/src/scheduler/scheduler.ts:84`

Event-driven orchestrator for tool execution:

```mermaid
stateDiagram-v2
    [*] --> Queued: Tool call requested
    Queued --> PolicyCheck: Process queue
    PolicyCheck --> Denied: Policy denies
    PolicyCheck --> NeedsConfirmation: Requires approval
    PolicyCheck --> Executing: Auto-approved
    NeedsConfirmation --> WaitingForUser: Request confirmation
    WaitingForUser --> Denied: User denies
    WaitingForUser --> Executing: User approves
    Executing --> Completed: Success
    Executing --> Errored: Failure
    Completed --> [*]
    Errored --> [*]
    Denied --> [*]
```

**States:**
- `Queued` - Tool call received
- `Validating` - Policy engine checking
- `Executing` - Tool running
- `Completed` - Success
- `Errored` - Failed

### 5. Tool Registry
**File:** `packages/core/src/tools/tool-registry.ts`

Manages all available tools:
- Built-in tools (Read, Write, Edit, Bash, Grep, Glob, etc.)
- MCP (Model Context Protocol) tools
- Custom extension tools
- Agent tools (subagents)

**Built-in Tools:**
- `Read` - Read files (`packages/core/src/tools/read-file.ts`)
- `Write` - Write files (`packages/core/src/tools/write-file.ts`)
- `Edit` - Edit files (`packages/core/src/tools/edit.ts`)
- `Bash` - Execute shell commands (`packages/core/src/tools/shell.ts`)
- `Grep` - Search content (`packages/core/src/tools/grep.ts`)
- `Glob` - Find files by pattern (`packages/core/src/tools/glob.ts`)
- `WebFetch` - Fetch web content (`packages/core/src/tools/web-fetch.ts`)
- `WebSearch` - Search the web (`packages/core/src/tools/web-search.ts`)

### 6. Agent Executor
**File:** `packages/core/src/agents/local-executor.ts:91`

Executes agent loops (subagents):
- Creates isolated tool registry per agent
- Manages agent-specific conversation
- Enforces turn limits & timeouts
- Requires `complete_task` tool call to finish

```mermaid
graph TB
    Start([Agent Invoked])
    Init[Initialize Agent Context]
    SendPrompt[Send Agent Prompt]
    Stream[Stream LLM Response]
    HasTools{Has Tool Calls?}
    ExecuteTools[Execute Tools via Scheduler]
    Complete{complete_task called?}
    MaxTurns{Max turns exceeded?}
    Return([Return Result])
    Error([Error/Timeout])

    Start --> Init
    Init --> SendPrompt
    SendPrompt --> Stream
    Stream --> HasTools
    HasTools -->|Yes| ExecuteTools
    HasTools -->|No| Complete
    ExecuteTools --> Complete
    Complete -->|Yes| Return
    Complete -->|No| MaxTurns
    MaxTurns -->|No| SendPrompt
    MaxTurns -->|Yes| Error

    style Return fill:#90EE90
    style Error fill:#FFB6C6
```

### 7. Policy Engine
**File:** `packages/core/src/policy/`

Controls tool access and behavior:
- Approval modes: `auto`, `prompt`, `reject`
- Tool-specific policies
- Pattern matching for file operations
- Integrates with MessageBus for decisions

---

## Design Decisions

### 1. **Monorepo Architecture**
- **Packages:**
  - `core` - Business logic, tools, agents
  - `cli` - User interface, TUI components
  - `sdk` - Extension development SDK
  - `a2a-server` - Agent-to-agent server
  - `vscode-ide-companion` - VSCode extension

**Rationale:** Enables code sharing while maintaining separation of concerns.

### 2. **Event-Driven Architecture (MessageBus)**
**File:** `packages/core/src/confirmation-bus/message-bus.ts`

Components communicate via events:
- Tool confirmation requests
- Policy updates
- User feedback
- Console logs

**Benefits:**
- Loose coupling
- Easy to extend
- Supports async workflows

### 3. **React + Ink for TUI**
**File:** `packages/cli/src/ui/App.tsx`

Uses React components for terminal UI:
- Declarative UI updates
- Component reusability
- Familiar React patterns

**Components:**
- `AppContainer` - Root component
- `ChatHistory` - Message display
- `InputPrompt` - User input
- `ToolExecutionDisplay` - Tool call visualization

### 4. **Streaming by Default**
All LLM interactions use streaming:
- Better UX (immediate feedback)
- Supports long-running responses
- Enables real-time tool execution

### 5. **Sandboxing Support**
**File:** `packages/cli/src/utils/sandbox.js`

Optional Docker/Podman sandboxing:
- Isolates tool execution
- Enhanced security
- Clean environment per session

### 6. **Prompt Processing Pipeline**
**Files:** `packages/cli/src/services/prompt-processors/`

Multi-stage processing:
1. **Argument Processor** - CLI arguments
2. **Shell Processor** - Command injection
3. **@File Processor** - File content injection
4. **Injection Parser** - Special syntax

### 7. **Compression & Context Management**
**File:** `packages/core/src/services/chatCompressionService.ts`

Manages token limits:
- Compresses old turns when needed
- Preserves recent context
- Maintains conversation coherence

### 8. **Extension System**
**File:** `packages/cli/src/config/extension-manager.ts`

Supports extensions via:
- Custom tools
- Custom agents
- Themes
- Hooks (SessionStart, SessionEnd, etc.)
- MCP servers

---

## Package Structure

```
gemini-cli/
├── packages/
│   ├── core/                    # Core business logic
│   │   ├── src/
│   │   │   ├── agents/          # Agent execution (local-executor, registry)
│   │   │   ├── core/            # Core APIs (geminiChat, contentGenerator)
│   │   │   ├── scheduler/       # Tool orchestration
│   │   │   ├── tools/           # Tool implementations
│   │   │   ├── policy/          # Policy engine
│   │   │   ├── hooks/           # Hook system
│   │   │   ├── mcp/             # MCP client
│   │   │   ├── services/        # Shared services
│   │   │   ├── config/          # Configuration management
│   │   │   └── utils/           # Utilities
│   │
│   ├── cli/                     # CLI interface
│   │   ├── src/
│   │   │   ├── ui/              # React TUI components
│   │   │   │   ├── components/  # Reusable components
│   │   │   │   ├── contexts/    # React contexts
│   │   │   │   └── hooks/       # Custom hooks
│   │   │   ├── config/          # CLI-specific config
│   │   │   ├── services/        # Prompt processors, command loaders
│   │   │   ├── utils/           # CLI utilities
│   │   │   ├── gemini.tsx       # Main entry point
│   │   │   └── nonInteractiveCli.ts  # Non-interactive mode
│   │
│   ├── sdk/                     # Extension SDK
│   ├── a2a-server/              # Agent-to-agent communication
│   ├── vscode-ide-companion/    # VSCode extension
│   └── test-utils/              # Testing utilities
```

---

## Detailed Flow Diagrams

### Complete Request Flow (Interactive Mode)

```mermaid
sequenceDiagram
    participant User
    participant InputPrompt as Input Prompt (React)
    participant AppContainer
    participant PromptProc as Prompt Processors
    participant Hooks as Hook System
    participant GeminiChat
    participant ContentGen
    participant API as Gemini API
    participant Scheduler
    participant ToolRegistry
    participant Policy as Policy Engine
    participant Tool as Tool Implementation
    participant MessageBus

    User->>InputPrompt: Type prompt & press Enter
    InputPrompt->>AppContainer: onSubmit(prompt)
    AppContainer->>PromptProc: Process prompt

    Note over PromptProc: 1. Argument injection<br/>2. Shell command expansion<br/>3. @file content injection

    PromptProc->>Hooks: Fire UserPrompt hook
    Hooks-->>PromptProc: Hook response (optional context)

    PromptProc->>GeminiChat: sendMessageStream(processedPrompt)
    GeminiChat->>ContentGen: generateContentStream(request)
    ContentGen->>API: POST /generateContent (streaming)

    loop Stream Response Chunks
        API-->>ContentGen: Chunk
        ContentGen-->>GeminiChat: GenerateContentResponse
        GeminiChat-->>AppContainer: Stream event
        AppContainer-->>User: Display chunk in TUI
    end

    alt Response contains tool calls
        GeminiChat->>Scheduler: scheduleTools(toolCalls)

        loop For each tool call
            Scheduler->>Policy: checkPolicy(toolName, args)

            alt Policy requires confirmation
                Policy->>MessageBus: Emit confirmation request
                MessageBus-->>AppContainer: Display confirmation dialog
                AppContainer-->>User: Show tool confirmation UI
                User->>AppContainer: Approve/Reject
                AppContainer->>MessageBus: Confirmation response
                MessageBus-->>Scheduler: Continue/Cancel
            end

            alt Policy approves
                Scheduler->>ToolRegistry: getTool(toolName)
                ToolRegistry-->>Scheduler: Tool instance
                Scheduler->>Tool: execute(args)
                Tool-->>Scheduler: ToolResult
                Scheduler->>MessageBus: Emit tool status
                MessageBus-->>AppContainer: Update UI
            end
        end

        Scheduler-->>GeminiChat: All tool results
        GeminiChat->>ContentGen: Continue conversation with results
        ContentGen->>API: Next turn (with function responses)

        Note over API: LLM processes tool results<br/>and generates next response

        API-->>ContentGen: Response
        ContentGen-->>GeminiChat: Final response
    end

    GeminiChat-->>AppContainer: Conversation complete
    AppContainer-->>User: Display final response
```

### Non-Interactive Mode Flow

```mermaid
sequenceDiagram
    participant User as User (stdin/--prompt)
    participant Main as main()
    participant NonInt as runNonInteractive()
    participant GeminiChat
    participant Scheduler
    participant stdout

    User->>Main: Execute: gemini --prompt "..."
    Main->>Main: loadSettings()
    Main->>Main: parseArguments()
    Main->>Main: initialize config
    Main->>NonInt: runNonInteractive({config, input})

    NonInt->>GeminiChat: sendMessageStream(input)

    loop Streaming
        GeminiChat-->>NonInt: Chunk
        NonInt->>stdout: Write chunk
        stdout-->>User: Display
    end

    alt Has tool calls
        GeminiChat->>Scheduler: Execute tools
        Scheduler-->>GeminiChat: Results
        Note over GeminiChat,Scheduler: Tools auto-execute<br/>(no user confirmation)
        GeminiChat->>GeminiChat: Continue with results
    end

    GeminiChat-->>NonInt: Complete
    NonInt->>stdout: Final output
    NonInt->>Main: Exit
    Main-->>User: Process exits
```

### Tool Execution Flow (Detailed)

```mermaid
graph TB
    Start([Tool Call from LLM])
    Queue[Add to Scheduler Queue]
    Process{Processing Loop}
    GetNext[Get Next Queued Tool]
    PolicyCheck[Policy Engine Check]

    subgraph "Policy Decision"
        Auto[Auto-approve]
        Prompt[Require Confirmation]
        Reject[Deny]
    end

    WaitUser[Display Confirmation UI]
    UserDecision{User Approves?}

    Modify{Tool Modifiable?}
    ModifyUI[Show Modification UI]
    UserModify{User Modifies?}

    Execute[Execute Tool]
    Success{Success?}
    Result[Return Result]
    Error[Return Error]
    NextTool{More Tools?}
    Complete([All Complete])

    Start --> Queue
    Queue --> Process
    Process --> GetNext
    GetNext --> PolicyCheck

    PolicyCheck --> Auto
    PolicyCheck --> Prompt
    PolicyCheck --> Reject

    Auto --> Modify
    Prompt --> WaitUser
    Reject --> Error

    WaitUser --> UserDecision
    UserDecision -->|No| Error
    UserDecision -->|Yes| Modify

    Modify -->|Yes| ModifyUI
    Modify -->|No| Execute

    ModifyUI --> UserModify
    UserModify -->|Modified| Execute
    UserModify -->|Cancelled| Error

    Execute --> Success
    Success -->|Yes| Result
    Success -->|No| Error

    Result --> NextTool
    Error --> NextTool
    NextTool -->|Yes| GetNext
    NextTool -->|No| Complete

    style Auto fill:#90EE90
    style Reject fill:#FFB6C6
    style Result fill:#90EE90
    style Error fill:#FFB6C6
```

### Agent Execution Flow

```mermaid
graph TB
    Invoke([Agent Tool Called])
    Create[Create LocalAgentExecutor]
    Registry[Create Isolated Tool Registry]
    LoadTools[Load Agent's Tools]
    InitChat[Initialize Agent Chat]
    SendPrompt[Send Agent System + User Prompt]

    subgraph "Agent Loop"
        Stream[Stream LLM Response]
        Parse{Parse Response}
        HasFC{Has Function Calls?}
        CheckComplete{complete_task called?}
        ScheduleTools[Schedule Tool Execution]
        WaitTools[Wait for Results]
        AddToHistory[Add to Agent History]
        CheckLimits{Within Limits?}
    end

    ExtractResult[Extract Final Result]
    Return([Return to Parent])
    Timeout([Timeout Error])

    Invoke --> Create
    Create --> Registry
    Registry --> LoadTools
    LoadTools --> InitChat
    InitChat --> SendPrompt

    SendPrompt --> Stream
    Stream --> Parse
    Parse --> HasFC

    HasFC -->|No| CheckComplete
    HasFC -->|Yes| ScheduleTools
    ScheduleTools --> WaitTools
    WaitTools --> AddToHistory

    CheckComplete -->|Yes| ExtractResult
    CheckComplete -->|No| CheckLimits

    AddToHistory --> CheckLimits
    CheckLimits -->|Yes| SendPrompt
    CheckLimits -->|No| Timeout

    ExtractResult --> Return

    style Return fill:#90EE90
    style Timeout fill:#FFB6C6
```

---

## Key Code Navigation Guide

### To understand user input processing:
1. Start at `packages/cli/src/gemini.tsx:318` (main function)
2. For interactive: `packages/cli/src/ui/AppContainer.tsx` (React UI)
3. For non-interactive: `packages/cli/src/nonInteractiveCli.ts:58`
4. Prompt processing: `packages/cli/src/services/prompt-processors/`

### To understand LLM communication:
1. `packages/core/src/core/geminiChat.ts` - Main chat handler
2. `packages/core/src/core/contentGenerator.ts:133` - API abstraction
3. Authentication: `packages/core/src/core/contentGenerator.ts:51-57` (AuthType enum)

### To understand tool execution:
1. `packages/core/src/scheduler/scheduler.ts:84` - Scheduler class
2. `packages/core/src/tools/tool-registry.ts` - Tool management
3. Individual tools: `packages/core/src/tools/*.ts`
4. Policy engine: `packages/core/src/policy/`

### To understand agent execution:
1. `packages/core/src/agents/local-executor.ts:91` - Agent executor
2. `packages/core/src/agents/registry.ts` - Agent registry
3. Agent definitions: `packages/core/src/agents/*-agent.ts`

### To understand the UI:
1. `packages/cli/src/ui/App.tsx` - Main app component
2. `packages/cli/src/ui/components/` - Reusable components
3. `packages/cli/src/ui/hooks/` - Custom React hooks
4. State management: `packages/cli/src/ui/contexts/`

---

## Design Patterns Used

1. **Registry Pattern** - Tool Registry, Agent Registry
2. **Strategy Pattern** - Different auth types via ContentGenerator
3. **Observer Pattern** - MessageBus for event communication
4. **Command Pattern** - Tool execution via Scheduler
5. **Builder Pattern** - Config construction
6. **Factory Pattern** - Tool/Agent creation
7. **State Machine** - Tool execution states in Scheduler
8. **Decorator Pattern** - LoggingContentGenerator wraps real generator

---

## Extension Points

### 1. Custom Tools
Create new tools by implementing the tool interface:
```typescript
// packages/core/src/tools/tools.ts
interface AnyDeclarativeTool {
  name: string;
  description: string;
  build(config: Config): FunctionDeclaration;
  execute(args: unknown, context: ToolContext): Promise<ToolResult>;
}
```

### 2. Custom Agents
Define agents in agent registry:
```typescript
// packages/core/src/agents/registry.ts
interface LocalAgentDefinition {
  name: string;
  systemInstruction: string;
  toolConfig?: ToolConfig;
  outputSchema?: z.ZodTypeAny;
}
```

### 3. Hooks
Implement lifecycle hooks:
- `SessionStart` - Run at session start
- `SessionEnd` - Run at session end
- `UserPrompt` - Intercept user prompts
- `ToolCall` - Intercept tool calls

### 4. MCP Servers
Connect Model Context Protocol servers for external tools.

### 5. Themes
Custom color schemes via theme extensions.

---

## Summary

The Gemini CLI architecture is:
- **Modular** - Clear separation via packages
- **Extensible** - Hook system, extensions, MCP
- **Event-driven** - MessageBus for loose coupling
- **Secure** - Policy engine, sandboxing, approval flows
- **Interactive** - Real-time streaming, TUI with React
- **Flexible** - Multiple auth methods, interactive/non-interactive modes

The core flow is: **User Input → Prompt Processing → LLM API → Stream Response → Tool Execution (if needed) → Loop until complete → Display Result**

This architecture enables a powerful, extensible AI coding assistant that can safely execute tools while maintaining user control through policies and confirmations.
