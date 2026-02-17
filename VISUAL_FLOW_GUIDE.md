# Gemini CLI - Visual Flow Guide

This document provides additional detailed visualizations to help navigate the codebase.

## Table of Contents
1. [Package Dependency Graph](#package-dependency-graph)
2. [Class Hierarchy](#class-hierarchy)
3. [Data Flow Diagrams](#data-flow-diagrams)
4. [State Machines](#state-machines)
5. [Component Interaction](#component-interaction)

---

## Package Dependency Graph

```mermaid
graph TB
    subgraph "Packages"
        CLI[cli<br/>User Interface]
        CORE[core<br/>Business Logic]
        SDK[sdk<br/>Extension API]
        A2A[a2a-server<br/>Agent Server]
        VSCODE[vscode-ide-companion<br/>IDE Extension]
        TEST[test-utils<br/>Testing]
    end

    subgraph "External Dependencies"
        GENAI[@google/genai<br/>API Client]
        MCP_SDK[@modelcontextprotocol/sdk<br/>MCP Protocol]
        INK[ink<br/>React Terminal UI]
        REACT[react]
    end

    CLI --> CORE
    SDK --> CORE
    A2A --> CORE
    VSCODE --> CORE
    TEST --> CORE

    CLI --> INK
    CLI --> REACT
    CORE --> GENAI
    CORE --> MCP_SDK

    style CORE fill:#fff4e1
    style CLI fill:#e1f5ff
    style SDK fill:#ffe1f5
```

---

## Class Hierarchy

### ContentGenerator Hierarchy

```mermaid
classDiagram
    class ContentGenerator {
        <<interface>>
        +generateContent()
        +generateContentStream()
        +countTokens()
        +embedContent()
    }

    class LoggingContentGenerator {
        -wrapped: ContentGenerator
        +generateContent()
        +generateContentStream()
    }

    class RecordingContentGenerator {
        -wrapped: ContentGenerator
        +generateContent()
        +generateContentStream()
    }

    class GoogleGenAIGenerator {
        -client: GoogleGenAI
        +generateContent()
        +generateContentStream()
    }

    class CodeAssistGenerator {
        -codeAssistClient
        +generateContent()
        +generateContentStream()
    }

    class FakeContentGenerator {
        -responses: []
        +generateContent()
        +generateContentStream()
    }

    ContentGenerator <|.. GoogleGenAIGenerator
    ContentGenerator <|.. CodeAssistGenerator
    ContentGenerator <|.. FakeContentGenerator
    ContentGenerator <|.. LoggingContentGenerator
    ContentGenerator <|.. RecordingContentGenerator

    LoggingContentGenerator o-- ContentGenerator
    RecordingContentGenerator o-- ContentGenerator
```

### Tool Hierarchy

```mermaid
classDiagram
    class AnyDeclarativeTool {
        <<interface>>
        +name: string
        +description: string
        +build()
        +execute()
    }

    class ModifiableTool {
        <<interface>>
        +canModifyBeforeExecution: true
        +showModificationUI()
    }

    class McpTool {
        +serverName: string
        +build()
        +execute()
    }

    class ReadFileTool {
        +build()
        +execute()
    }

    class WriteFileTool {
        +build()
        +execute()
        +showModificationUI()
    }

    class BashTool {
        +build()
        +execute()
        +showModificationUI()
    }

    class EditTool {
        +build()
        +execute()
    }

    class GrepTool {
        +build()
        +execute()
    }

    class GlobTool {
        +build()
        +execute()
    }

    AnyDeclarativeTool <|.. ReadFileTool
    AnyDeclarativeTool <|.. WriteFileTool
    AnyDeclarativeTool <|.. BashTool
    AnyDeclarativeTool <|.. EditTool
    AnyDeclarativeTool <|.. GrepTool
    AnyDeclarativeTool <|.. GlobTool
    AnyDeclarativeTool <|.. McpTool

    ModifiableTool <|.. WriteFileTool
    ModifiableTool <|.. BashTool
```

---

## Data Flow Diagrams

### Message History Management

```mermaid
graph TB
    UserInput[User Input]
    CreateContent[Create User Content]
    AddToHistory[Add to History]
    Compress{Token limit exceeded?}
    CompressionService[Chat Compression Service]
    SendToAPI[Send to API]
    ReceiveResponse[Receive Model Response]
    ValidateResponse{Valid response?}
    AddModelResponse[Add Model to History]
    ToolCalls{Has tool calls?}
    ExecuteTools[Execute Tools]
    CreateFunctionResponse[Create Function Response]
    AddFunctionResponse[Add to History]
    CuratedHistory[Extract Curated History]

    UserInput --> CreateContent
    CreateContent --> AddToHistory
    AddToHistory --> Compress

    Compress -->|Yes| CompressionService
    Compress -->|No| SendToAPI
    CompressionService --> SendToAPI

    SendToAPI --> ReceiveResponse
    ReceiveResponse --> ValidateResponse

    ValidateResponse -->|Invalid| SendToAPI
    ValidateResponse -->|Valid| AddModelResponse

    AddModelResponse --> ToolCalls

    ToolCalls -->|Yes| ExecuteTools
    ToolCalls -->|No| CuratedHistory

    ExecuteTools --> CreateFunctionResponse
    CreateFunctionResponse --> AddFunctionResponse
    AddFunctionResponse --> Compress

    style CompressionService fill:#ffe1e1
    style ValidateResponse fill:#fff4e1
```

### Configuration Loading Flow

```mermaid
graph TB
    Start([Application Start])
    LoadEnv[Load Environment Variables]
    LoadSettingsFiles[Load Settings Files]

    subgraph "Settings Hierarchy"
        Default[Default Settings]
        User[User Settings<br/>~/.config/gemini/settings.json]
        Workspace[Workspace Settings<br/>.gemini/settings.json]
        Project[Project Settings<br/>project-specific]
    end

    MergeSettings[Merge Settings<br/>Project > Workspace > User > Default]
    ValidateSettings[Validate Settings Schema]
    ParseArgs[Parse CLI Arguments]
    OverrideWithArgs[Override with CLI Args]
    LoadHooks[Load Hooks]
    LoadExtensions[Load Extensions]
    LoadMCP[Load MCP Servers]
    InitPolicy[Initialize Policy Engine]
    CreateConfig[Create Config Object]
    Ready([Config Ready])

    Start --> LoadEnv
    LoadEnv --> LoadSettingsFiles

    LoadSettingsFiles --> Default
    LoadSettingsFiles --> User
    LoadSettingsFiles --> Workspace
    LoadSettingsFiles --> Project

    Default --> MergeSettings
    User --> MergeSettings
    Workspace --> MergeSettings
    Project --> MergeSettings

    MergeSettings --> ValidateSettings
    ValidateSettings --> ParseArgs
    ParseArgs --> OverrideWithArgs
    OverrideWithArgs --> LoadHooks
    LoadHooks --> LoadExtensions
    LoadExtensions --> LoadMCP
    LoadMCP --> InitPolicy
    InitPolicy --> CreateConfig
    CreateConfig --> Ready

    style MergeSettings fill:#e1f5ff
    style ValidateSettings fill:#fff4e1
```

### Tool Call Resolution Flow

```mermaid
graph TB
    LLMResponse[LLM Response with Tool Calls]
    ParseCalls[Parse Function Calls]
    CreateRequests[Create ToolCallRequestInfo objects]
    QueueRequests[Queue in Scheduler]
    ProcessQueue{Process Queue}
    GetNext[Get Next Request]
    FindTool[Find Tool in Registry]
    ToolExists{Tool exists?}
    CheckAuth{Tool authorized?}
    PolicyCheck[Policy Engine Check]
    PolicyResult{Policy decision?}

    subgraph "Approval Flow"
        AutoApprove[Auto-approve]
        RequireConfirm[Require Confirmation]
        Deny[Deny]
    end

    SendConfirmReq[Send Confirmation Request]
    MessageBusEmit[MessageBus.emit]
    UIShowDialog[UI: Show Confirmation Dialog]
    WaitUser[Wait for User Response]
    UserResponse{User approves?}

    CheckModifiable{Tool modifiable?}
    ShowModUI[Show Modification UI]
    UserModifies{User modifies?}

    ExecuteTool[Execute Tool]
    HandleError[Handle Error]
    ReturnResult[Return ToolCallResponseInfo]
    MoreTools{More tools?}
    AllComplete[All Tools Complete]
    CreateModelContent[Create Model Content with Results]

    LLMResponse --> ParseCalls
    ParseCalls --> CreateRequests
    CreateRequests --> QueueRequests
    QueueRequests --> ProcessQueue

    ProcessQueue --> GetNext
    GetNext --> FindTool
    FindTool --> ToolExists

    ToolExists -->|No| HandleError
    ToolExists -->|Yes| CheckAuth

    CheckAuth -->|No| HandleError
    CheckAuth -->|Yes| PolicyCheck

    PolicyCheck --> PolicyResult

    PolicyResult --> AutoApprove
    PolicyResult --> RequireConfirm
    PolicyResult --> Deny

    Deny --> HandleError

    AutoApprove --> CheckModifiable
    RequireConfirm --> SendConfirmReq

    SendConfirmReq --> MessageBusEmit
    MessageBusEmit --> UIShowDialog
    UIShowDialog --> WaitUser
    WaitUser --> UserResponse

    UserResponse -->|No| HandleError
    UserResponse -->|Yes| CheckModifiable

    CheckModifiable -->|No| ExecuteTool
    CheckModifiable -->|Yes| ShowModUI

    ShowModUI --> UserModifies
    UserModifies -->|Cancelled| HandleError
    UserModifies -->|Modified| ExecuteTool

    ExecuteTool --> ReturnResult
    HandleError --> ReturnResult

    ReturnResult --> MoreTools
    MoreTools -->|Yes| GetNext
    MoreTools -->|No| AllComplete

    AllComplete --> CreateModelContent

    style AutoApprove fill:#90EE90
    style Deny fill:#FFB6C6
    style HandleError fill:#FFB6C6
```

---

## State Machines

### Application State Machine

```mermaid
stateDiagram-v2
    [*] --> Initializing: Start

    Initializing --> LoadingAuth: Load config
    LoadingAuth --> AuthFailed: Auth error
    LoadingAuth --> Ready: Auth success

    AuthFailed --> [*]: Exit
    AuthFailed --> LoadingAuth: Retry

    Ready --> Idle: Initialized
    Idle --> ProcessingInput: User submits prompt

    ProcessingInput --> StreamingResponse: LLM request sent
    StreamingResponse --> StreamingResponse: Receive chunks

    StreamingResponse --> ExecutingTools: Tool calls received
    ExecutingTools --> WaitingForConfirmation: Needs approval
    WaitingForConfirmation --> ExecutingTools: Approved
    WaitingForConfirmation --> StreamingResponse: Rejected
    ExecutingTools --> StreamingResponse: Tools complete

    StreamingResponse --> Idle: Response complete
    StreamingResponse --> Error: Stream error

    Error --> Idle: Error handled
    Error --> [*]: Fatal error

    Idle --> [*]: Exit command

    note right of WaitingForConfirmation
        User sees confirmation dialog
        Can approve/reject/modify
    end note
```

### Tool Execution State Machine

```mermaid
stateDiagram-v2
    [*] --> Queued: Tool call requested

    Queued --> Validating: Start processing

    Validating --> Denied: Policy denies
    Validating --> PendingConfirmation: Needs user approval
    Validating --> PendingModification: Tool is modifiable
    Validating --> Executing: Auto-approved

    PendingConfirmation --> Denied: User denies
    PendingConfirmation --> PendingModification: User approves (if modifiable)
    PendingConfirmation --> Executing: User approves (not modifiable)

    PendingModification --> Denied: User cancels
    PendingModification --> Executing: User confirms

    Executing --> Completed: Success
    Executing --> Errored: Failure

    Denied --> [*]
    Completed --> [*]
    Errored --> [*]

    note right of PendingConfirmation
        MessageBus coordination
        UI displays dialog
    end note

    note right of PendingModification
        User can edit tool args
        e.g., modify bash command
    end note
```

### Agent Execution State Machine

```mermaid
stateDiagram-v2
    [*] --> Created: Agent tool called

    Created --> Initialized: Setup complete
    Initialized --> SendingPrompt: Agent loop starts

    SendingPrompt --> StreamingAgentResponse: Waiting for LLM
    StreamingAgentResponse --> ProcessingAgentResponse: Stream complete

    ProcessingAgentResponse --> CheckingCompletion: Parse response

    CheckingCompletion --> AgentComplete: complete_task called
    CheckingCompletion --> ExecutingAgentTools: Tool calls found
    CheckingCompletion --> SendingPrompt: No tools, continue

    ExecutingAgentTools --> CheckingLimits: Tools complete

    CheckingLimits --> TurnLimitExceeded: Max turns reached
    CheckingLimits --> TimeLimitExceeded: Timeout
    CheckingLimits --> SendingPrompt: Within limits

    AgentComplete --> ReturningResult: Extract output
    TurnLimitExceeded --> ReturningError: Max turns error
    TimeLimitExceeded --> ReturningError: Timeout error

    ReturningResult --> [*]
    ReturningError --> [*]

    note right of CheckingLimits
        Default: 25 turns
        Default: 30 minutes
    end note
```

---

## Component Interaction

### Interactive Session Component Flow

```mermaid
graph TB
    subgraph "React Component Tree"
        AppContainer[AppContainer]
        App[App]
        ChatHistory[ChatHistory]
        InputPrompt[InputPrompt]
        ToolDisplay[ToolExecutionDisplay]
        ConfirmDialog[ConfirmationDialog]
        StatusBar[StatusBar]
    end

    subgraph "React Contexts"
        AppContext[AppContext]
        UIStateContext[UIStateContext]
        ConfigContext[ConfigContext]
        StreamingContext[StreamingContext]
        VimModeContext[VimModeContext]
    end

    subgraph "Core Logic"
        GeminiChat[GeminiChat]
        Scheduler[Scheduler]
        MessageBus[MessageBus]
        Config[Config]
    end

    AppContainer --> App
    App --> ChatHistory
    App --> InputPrompt
    App --> ToolDisplay
    App --> ConfirmDialog
    App --> StatusBar

    AppContainer -.-> AppContext
    App -.-> UIStateContext
    App -.-> ConfigContext
    App -.-> StreamingContext
    InputPrompt -.-> VimModeContext

    AppContext --> GeminiChat
    AppContext --> Scheduler
    AppContext --> MessageBus
    ConfigContext --> Config

    InputPrompt -->|User input| AppContainer
    AppContainer -->|Submit| GeminiChat
    GeminiChat -->|Stream events| StreamingContext
    StreamingContext -->|Updates| ChatHistory

    Scheduler -->|Confirmation needed| MessageBus
    MessageBus -->|Event| ConfirmDialog
    ConfirmDialog -->|Response| MessageBus
    MessageBus -->|Result| Scheduler

    style AppContainer fill:#e1f5ff
    style GeminiChat fill:#fff4e1
    style MessageBus fill:#ffe1f5
```

### Non-Interactive Session Flow

```mermaid
sequenceDiagram
    participant CLI as CLI Process
    participant Config
    participant GeminiChat
    participant ContentGen
    participant API
    participant Scheduler
    participant Tools
    participant stdout

    CLI->>Config: Initialize
    Config-->>CLI: Ready
    CLI->>GeminiChat: sendMessageStream(prompt)
    GeminiChat->>ContentGen: generateContentStream()
    ContentGen->>API: POST request

    loop Streaming
        API-->>ContentGen: Chunk
        ContentGen-->>GeminiChat: Response chunk
        GeminiChat-->>CLI: Stream event
        CLI->>stdout: Write to stdout
    end

    alt Has tool calls
        GeminiChat->>Scheduler: scheduleTools()

        Note over Scheduler,Tools: All tools auto-execute<br/>No user confirmations

        loop Each tool
            Scheduler->>Tools: execute()
            Tools-->>Scheduler: Result
        end

        Scheduler-->>GeminiChat: All results
        GeminiChat->>ContentGen: Next turn
        ContentGen->>API: Continue conversation
    end

    API-->>ContentGen: Final response
    ContentGen-->>GeminiChat: Complete
    GeminiChat-->>CLI: Done
    CLI->>CLI: Exit process
```

### MCP Integration Flow

```mermaid
graph TB
    Start([MCP Server Configured])
    LoadConfig[Load MCP Config<br/>from settings.json]
    StartServer[Start MCP Server Process]
    InitTransport[Initialize Transport<br/>stdio/sse]
    Connect[MCP Client Connect]
    ListTools[List Available Tools]
    RegisterTools[Register in Tool Registry]

    subgraph "Runtime"
        ToolCall[Tool Called by LLM]
        FindMCP[Find MCP Tool]
        SendRequest[Send MCP Request]
        WaitResponse[Wait for MCP Response]
        ParseResult[Parse Result]
        ReturnToLLM[Return to LLM]
    end

    Ready([MCP Tools Ready])

    Start --> LoadConfig
    LoadConfig --> StartServer
    StartServer --> InitTransport
    InitTransport --> Connect
    Connect --> ListTools
    ListTools --> RegisterTools
    RegisterTools --> Ready

    Ready --> ToolCall
    ToolCall --> FindMCP
    FindMCP --> SendRequest
    SendRequest --> WaitResponse
    WaitResponse --> ParseResult
    ParseResult --> ReturnToLLM

    style RegisterTools fill:#e1f5ff
    style SendRequest fill:#fff4e1
```

### Hook System Flow

```mermaid
sequenceDiagram
    participant App
    participant HookRegistry
    participant MessageBus
    participant HookImpl as Hook Implementation

    App->>HookRegistry: Register hook
    HookRegistry-->>App: Registered

    Note over App: Session starts

    App->>MessageBus: Emit SessionStart event
    MessageBus->>HookRegistry: Route to hooks
    HookRegistry->>HookImpl: execute(SessionStartEvent)

    alt Hook returns context
        HookImpl-->>HookRegistry: { additionalContext: "..." }
        HookRegistry-->>MessageBus: Hook result
        MessageBus-->>App: Add context to prompt
    end

    Note over App: Session continues

    App->>MessageBus: Emit ToolCall event
    MessageBus->>HookRegistry: Route to hooks
    HookRegistry->>HookImpl: execute(ToolCallEvent)

    alt Hook blocks tool
        HookImpl-->>HookRegistry: { blocked: true, reason: "..." }
        HookRegistry-->>MessageBus: Block result
        MessageBus-->>App: Cancel tool execution
    end

    Note over App: Session ends

    App->>MessageBus: Emit SessionEnd event
    MessageBus->>HookRegistry: Route to hooks
    HookRegistry->>HookImpl: execute(SessionEndEvent)
    HookImpl-->>HookRegistry: Cleanup complete
```

---

## Request/Response Data Structures

### GeminiChat Message Format

```mermaid
classDiagram
    class Content {
        +role: "user" | "model"
        +parts: Part[]
    }

    class Part {
        <<union>>
    }

    class TextPart {
        +text: string
    }

    class ThoughtPart {
        +thought: string
    }

    class FunctionCallPart {
        +functionCall: FunctionCall
    }

    class FunctionResponsePart {
        +functionResponse: FunctionResponse
    }

    class InlineDataPart {
        +inlineData: { mimeType, data }
    }

    class FileDataPart {
        +fileData: { mimeType, fileUri }
    }

    Content *-- Part
    Part <|-- TextPart
    Part <|-- ThoughtPart
    Part <|-- FunctionCallPart
    Part <|-- FunctionResponsePart
    Part <|-- InlineDataPart
    Part <|-- FileDataPart

    class FunctionCall {
        +name: string
        +args: object
    }

    class FunctionResponse {
        +name: string
        +response: object
    }

    FunctionCallPart *-- FunctionCall
    FunctionResponsePart *-- FunctionResponse
```

### Tool Call Lifecycle Objects

```mermaid
classDiagram
    class ToolCallRequestInfo {
        +callId: string
        +name: string
        +args: unknown
        +argsString?: string
    }

    class ToolCallResponseInfo {
        +callId: string
        +responseParts: Part[]
        +resultDisplay: string
        +error?: Error
        +errorType?: ToolErrorType
    }

    class ToolCall {
        <<union type>>
    }

    class QueuedToolCall {
        +status: "queued"
        +request: ToolCallRequestInfo
    }

    class ValidatingToolCall {
        +status: "validating"
        +request: ToolCallRequestInfo
    }

    class ExecutingToolCall {
        +status: "executing"
        +request: ToolCallRequestInfo
    }

    class CompletedToolCall {
        +status: "completed"
        +request: ToolCallRequestInfo
        +response: ToolCallResponseInfo
    }

    class ErroredToolCall {
        +status: "errored"
        +request: ToolCallRequestInfo
        +error: Error
    }

    ToolCall <|-- QueuedToolCall
    ToolCall <|-- ValidatingToolCall
    ToolCall <|-- ExecutingToolCall
    ToolCall <|-- CompletedToolCall
    ToolCall <|-- ErroredToolCall

    QueuedToolCall *-- ToolCallRequestInfo
    ValidatingToolCall *-- ToolCallRequestInfo
    ExecutingToolCall *-- ToolCallRequestInfo
    CompletedToolCall *-- ToolCallRequestInfo
    CompletedToolCall *-- ToolCallResponseInfo
    ErroredToolCall *-- ToolCallRequestInfo
```

---

## Streaming Architecture

### Stream Processing Pipeline

```mermaid
graph LR
    subgraph "API Layer"
        API[Gemini API]
    end

    subgraph "Content Generator"
        Stream[generateContentStream]
        Retry[Retry Logic]
    end

    subgraph "GeminiChat"
        Validate[Validate Response]
        Accumulate[Accumulate Chunks]
        Parse[Parse Complete Response]
    end

    subgraph "UI Layer"
        Buffer[Stream Buffer]
        Render[React Render]
        Display[Terminal Display]
    end

    API -->|HTTP/2 Stream| Stream
    Stream --> Retry
    Retry -->|Chunk| Validate
    Validate --> Accumulate
    Accumulate --> Parse

    Parse -->|Text chunks| Buffer
    Parse -->|Tool calls| Scheduler[Scheduler]

    Buffer --> Render
    Render --> Display

    style API fill:#e1ffe1
    style Validate fill:#fff4e1
    style Buffer fill:#e1f5ff
```

### Error Handling & Retry Flow

```mermaid
graph TB
    Request[API Request]
    Execute[Execute Request]
    CheckError{Error?}

    subgraph "Error Classification"
        Retryable{Retryable?}
        QuotaError[Quota Error]
        NetworkError[Network Error]
        InvalidContent[Invalid Content]
        FatalError[Fatal Error]
    end

    Retryable --> QuotaError
    Retryable --> NetworkError
    Retryable --> InvalidContent
    Retryable --> FatalError

    Backoff[Apply Backoff Delay]
    CheckAttempts{Max attempts?}
    FallbackModel[Try Fallback Model]
    ReturnError[Return Error]
    Success[Return Response]

    Request --> Execute
    Execute --> CheckError

    CheckError -->|Yes| Retryable
    CheckError -->|No| Success

    QuotaError --> FallbackModel
    NetworkError --> Backoff
    InvalidContent --> Backoff
    FatalError --> ReturnError

    FallbackModel --> Success
    FallbackModel --> ReturnError

    Backoff --> CheckAttempts
    CheckAttempts -->|No| Execute
    CheckAttempts -->|Yes| ReturnError

    style QuotaError fill:#FFB6C6
    style FatalError fill:#FFB6C6
    style Success fill:#90EE90
```

---

## Summary

This visual guide provides detailed diagrams for:
- **Package dependencies** and how they relate
- **Class hierarchies** for major components
- **Data flow** through the application
- **State machines** governing execution
- **Component interactions** in both interactive and non-interactive modes
- **Data structures** used for messages and tool calls
- **Streaming architecture** for real-time responses
- **Error handling** and retry mechanisms

Use these diagrams in conjunction with the Architecture Analysis document to navigate and understand the codebase structure and flow.
