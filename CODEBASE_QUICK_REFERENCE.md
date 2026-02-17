# Gemini CLI - Codebase Quick Reference

A quick reference guide for navigating the gemini-cli codebase. Use this to quickly find the file you need.

## 📁 Directory Structure Quick Map

```
packages/
├── cli/                      # User interface & TUI
│   ├── src/
│   │   ├── gemini.tsx       # 🔴 MAIN ENTRY POINT
│   │   ├── nonInteractiveCli.ts  # Non-interactive mode
│   │   ├── ui/              # React TUI components
│   │   ├── config/          # CLI configuration
│   │   └── services/        # Prompt processing
│
├── core/                     # Business logic & core APIs
│   ├── src/
│   │   ├── core/            # 🔴 CORE APIs (Chat, ContentGen)
│   │   ├── agents/          # Agent execution
│   │   ├── scheduler/       # Tool orchestration
│   │   ├── tools/           # Tool implementations
│   │   ├── policy/          # Access control
│   │   ├── hooks/           # Lifecycle hooks
│   │   ├── mcp/             # MCP client
│   │   └── services/        # Shared services
│
├── sdk/                      # Extension SDK
├── a2a-server/              # Agent-to-agent server
├── vscode-ide-companion/    # VSCode extension
└── test-utils/              # Testing utilities
```

---

## 🔴 Critical Entry Points

### Start Here

| File | Purpose | Line |
|------|---------|------|
| `packages/cli/index.ts` | Main CLI entry point | 9 |
| `packages/cli/src/gemini.tsx` | Main application logic | 318 |
| `packages/core/src/core/geminiChat.ts` | LLM conversation handler | - |
| `packages/core/src/scheduler/scheduler.ts` | Tool execution orchestrator | 84 |

### Understanding the Flow

```
User Input
    ↓
packages/cli/index.ts (Entry)
    ↓
packages/cli/src/gemini.tsx (Main)
    ↓
    ├─→ Interactive: packages/cli/src/ui/AppContainer.tsx
    │       ↓
    │   packages/cli/src/ui/App.tsx
    │
    └─→ Non-Interactive: packages/cli/src/nonInteractiveCli.ts
    ↓
packages/core/src/core/geminiChat.ts (LLM Communication)
    ↓
packages/core/src/core/contentGenerator.ts (API Client)
    ↓
Google Gemini API
```

---

## 📚 Core Components Reference

### LLM Communication

| File | What It Does | Key Classes/Functions |
|------|--------------|----------------------|
| `packages/core/src/core/geminiChat.ts` | Manages conversation with LLM | `GeminiChat` class |
| `packages/core/src/core/contentGenerator.ts` | Abstracts API authentication | `ContentGenerator` interface<br/>`createContentGenerator()` |
| `packages/core/src/core/geminiRequest.ts` | Request type definitions | `GeminiCodeRequest` type |

### Tool System

| File | What It Does | Key Classes/Functions |
|------|--------------|----------------------|
| `packages/core/src/scheduler/scheduler.ts` | Orchestrates tool execution | `Scheduler` class (line 84) |
| `packages/core/src/tools/tool-registry.ts` | Manages available tools | `ToolRegistry` class |
| `packages/core/src/tools/tools.ts` | Tool interface & utilities | `AnyDeclarativeTool` interface |

### Built-in Tools

| File | Tool Name | What It Does |
|------|-----------|--------------|
| `packages/core/src/tools/read-file.ts` | Read | Read file contents |
| `packages/core/src/tools/write-file.ts` | Write | Write file contents |
| `packages/core/src/tools/edit.ts` | Edit | Edit file with string replacement |
| `packages/core/src/tools/shell.ts` | Bash | Execute shell commands |
| `packages/core/src/tools/grep.ts` | Grep | Search file contents (ripgrep) |
| `packages/core/src/tools/glob.ts` | Glob | Find files by pattern |
| `packages/core/src/tools/web-fetch.ts` | WebFetch | Fetch web page content |
| `packages/core/src/tools/web-search.ts` | WebSearch | Search the web |
| `packages/core/src/tools/ls.ts` | Ls | List directory contents |

### Agent System

| File | What It Does | Key Classes/Functions |
|------|--------------|----------------------|
| `packages/core/src/agents/local-executor.ts` | Executes agent loops | `LocalAgentExecutor` class (line 91) |
| `packages/core/src/agents/registry.ts` | Agent definitions registry | `AgentRegistry` class |
| `packages/core/src/agents/agent-scheduler.ts` | Agent tool invocation | `scheduleAgentTools()` |

### Policy & Security

| File | What It Does | Key Classes/Functions |
|------|--------------|----------------------|
| `packages/core/src/policy/policyEngine.ts` | Access control engine | `PolicyEngine` class |
| `packages/core/src/scheduler/policy.ts` | Policy checking in scheduler | `checkPolicy()` |

### Configuration

| File | What It Does | Key Classes/Functions |
|------|--------------|----------------------|
| `packages/core/src/config/config.ts` | Main config interface | `Config` class |
| `packages/cli/src/config/config.ts` | CLI config parsing | `parseArguments()`<br/>`loadCliConfig()` |
| `packages/cli/src/config/settings.ts` | Settings management | `loadSettings()` |

---

## 🎨 UI Components Reference

### Main UI Components

| File | Component | Purpose |
|------|-----------|---------|
| `packages/cli/src/ui/AppContainer.tsx` | `AppContainer` | Root component with contexts |
| `packages/cli/src/ui/App.tsx` | `App` | Main app layout |
| `packages/cli/src/ui/components/ChatHistory.tsx` | `ChatHistory` | Message history display |
| `packages/cli/src/ui/components/InputPrompt.tsx` | `InputPrompt` | User input field |
| `packages/cli/src/ui/components/ToolExecutionDisplay.tsx` | `ToolExecutionDisplay` | Tool call visualization |
| `packages/cli/src/ui/components/ConfirmationDialog.tsx` | `ConfirmationDialog` | Tool confirmation UI |
| `packages/cli/src/ui/components/StatusBar.tsx` | `StatusBar` | Bottom status bar |

### React Contexts

| File | Context | Purpose |
|------|---------|---------|
| `packages/cli/src/ui/contexts/AppContext.tsx` | `AppContext` | Global app state |
| `packages/cli/src/ui/contexts/UIStateContext.tsx` | `UIStateContext` | UI state management |
| `packages/cli/src/ui/contexts/ConfigContext.tsx` | `ConfigContext` | Config access |
| `packages/cli/src/ui/contexts/StreamingContext.tsx` | `StreamingContext` | LLM stream state |
| `packages/cli/src/ui/contexts/VimModeContext.tsx` | `VimModeContext` | Vim mode state |

---

## 🔧 Services & Utilities

### Prompt Processing

| File | Purpose |
|------|---------|
| `packages/cli/src/services/prompt-processors/argumentProcessor.ts` | Process CLI arguments in prompt |
| `packages/cli/src/services/prompt-processors/atFileProcessor.ts` | Process @file references |
| `packages/cli/src/services/prompt-processors/shellProcessor.ts` | Process shell command injection |
| `packages/cli/src/services/prompt-processors/injectionParser.ts` | Parse special syntax |

### Message Bus & Events

| File | Purpose |
|------|---------|
| `packages/core/src/confirmation-bus/message-bus.ts` | Event communication system |
| `packages/core/src/utils/events.ts` | Core event definitions |
| `packages/cli/src/utils/events.ts` | App event definitions |

### MCP Integration

| File | Purpose |
|------|---------|
| `packages/core/src/mcp/mcp-client.ts` | MCP client implementation |
| `packages/core/src/tools/mcp-client-manager.ts` | Manages MCP connections |
| `packages/core/src/tools/mcp-tool.ts` | MCP tool wrapper |

### Hooks System

| File | Purpose |
|------|---------|
| `packages/core/src/hooks/hook-registry.ts` | Hook registry & execution |
| `packages/core/src/hooks/types.ts` | Hook type definitions |

---

## 🧪 Testing

| Directory | Purpose |
|-----------|---------|
| `packages/core/src/**/*.test.ts` | Unit tests for core |
| `packages/cli/src/**/*.test.tsx` | Unit tests for CLI |
| `integration-tests/` | Integration tests |
| `evals/` | LLM evaluation tests |

---

## 🔍 Common Tasks - Where to Look

### Task: Understand how user input is processed

1. Start: `packages/cli/src/ui/components/InputPrompt.tsx` (User types input)
2. Processing: `packages/cli/src/services/prompt-processors/` (Transform input)
3. Submission: `packages/cli/src/ui/AppContainer.tsx` (Handle submit)
4. Execution: `packages/core/src/core/geminiChat.ts` (Send to LLM)

### Task: Understand how LLM responses are displayed

1. Request: `packages/core/src/core/geminiChat.ts` → `sendMessageStream()`
2. Stream: `packages/core/src/core/contentGenerator.ts` → `generateContentStream()`
3. Receive: `packages/cli/src/ui/AppContainer.tsx` → Stream event handling
4. Display: `packages/cli/src/ui/components/ChatHistory.tsx` → Render message

### Task: Add a new tool

1. Create tool: `packages/core/src/tools/my-tool.ts`
2. Implement interface: `AnyDeclarativeTool` (from `packages/core/src/tools/tools.ts`)
3. Register: Add to `packages/core/src/tools/definitions/index.ts`
4. Test: Create `packages/core/src/tools/my-tool.test.ts`

### Task: Modify tool execution flow

1. Scheduling: `packages/core/src/scheduler/scheduler.ts`
2. Policy check: `packages/core/src/scheduler/policy.ts`
3. Confirmation: `packages/core/src/scheduler/confirmation.ts`
4. Execution: `packages/core/src/scheduler/tool-executor.ts`

### Task: Add a new agent

1. Define: `packages/core/src/agents/my-agent.ts`
2. Register: `packages/core/src/agents/registry.ts` → `getAllLocalAgentDefinitions()`
3. Executor: Uses `packages/core/src/agents/local-executor.ts` (no changes needed)

### Task: Understand authentication flow

1. Entry: `packages/cli/src/gemini.tsx` → Line 442-483 (Auth refresh)
2. Auth methods: `packages/cli/src/config/auth.ts`
3. Content generator: `packages/core/src/core/contentGenerator.ts` → `createContentGenerator()`
4. OAuth: `packages/core/src/core/oauth.ts`

### Task: Modify settings schema

1. Schema: `packages/core/src/config/settingsSchema.ts`
2. Types: `packages/cli/src/config/settings.ts`
3. Loading: `packages/cli/src/config/settings.ts` → `loadSettings()`
4. Validation: `packages/cli/src/config/settings-validation.ts`

### Task: Add a new hook

1. Type: `packages/core/src/hooks/types.ts` → Add to `HookType` enum
2. Implementation: Create hook in extension or built-in
3. Registry: `packages/core/src/hooks/hook-registry.ts` → Register hook
4. Trigger: Fire hook via `MessageBus` or directly

---

## 🌟 Key Patterns & Conventions

### File Naming

- **Implementation**: `my-feature.ts`
- **Tests**: `my-feature.test.ts`
- **Types**: `types.ts` (in feature directory)
- **Constants**: `constants.ts` (in feature directory)

### Code Organization

```
feature/
├── index.ts           # Public exports
├── types.ts           # Type definitions
├── constants.ts       # Constants
├── feature-main.ts    # Main implementation
├── feature-utils.ts   # Utilities
└── feature-main.test.ts  # Tests
```

### Import Conventions

- Use `.js` extensions in imports (TypeScript requirement for ESM)
- Relative imports for local files
- Package imports for cross-package dependencies

```typescript
// Good
import { Config } from './config.js';
import { GeminiChat } from '@google/gemini-cli-core';

// Bad
import { Config } from './config';  // Missing .js
```

### Async/Await

- Prefer `async/await` over `.then()` chains
- Always handle errors with try/catch or error handlers

### Event Emission

- Use `MessageBus` for cross-component communication
- Use `coreEvents` for core system events
- Use `appEvents` for app-level events

---

## 🚀 Debugging Tips

### Enable Debug Mode

```bash
gemini --debug
# or
DEBUG=1 gemini
```

**What it shows:**
- Debug logs to console
- Performance profiling
- Detailed error stacks

### View Debug Logs

Look for `debugLogger` usage:
```typescript
import { debugLogger } from '@google/gemini-cli-core';
debugLogger.debug('My debug message');
```

### Trace Tool Execution

1. Enable debug mode
2. Look for logs in `packages/core/src/scheduler/scheduler.ts`
3. Check `packages/core/src/telemetry/loggers.ts` for telemetry

### Trace LLM Requests

1. Check `packages/core/src/core/geminiChat.ts` → `sendMessageStream()`
2. Enable request logging in `packages/core/src/core/loggingContentGenerator.ts`

### UI Debugging

```bash
# Enable React DevTools
gemini --devtools
```

Access at: `http://localhost:8097` (when running)

---

## 📊 Key Metrics & Limits

| Metric | Default | Configurable |
|--------|---------|--------------|
| Max agent turns | 25 | Yes (per agent) |
| Agent timeout | 30 minutes | Yes (per agent) |
| Token limit | Model-dependent | No |
| Max retry attempts | 2-3 (varies) | Some |
| Compression threshold | Token-based | Indirect |

---

## 🔗 Related Documentation

- Main README: `/README.md`
- Architecture Analysis: `/ARCHITECTURE_ANALYSIS.md`
- Visual Flow Guide: `/VISUAL_FLOW_GUIDE.md`
- SDK Design: `/packages/sdk/SDK_DESIGN.md`
- Contributing: `/CONTRIBUTING.md`

---

## 💡 Quick Answers

**Q: Where is the main loop?**
A: `packages/core/src/core/geminiChat.ts` for LLM, `packages/core/src/agents/local-executor.ts` for agents

**Q: How do I add a keyboard shortcut?**
A: `packages/cli/src/ui/hooks/useKeyboardShortcuts.ts`

**Q: Where are settings stored?**
A: `~/.config/gemini/settings.json` (user), `.gemini/settings.json` (workspace)

**Q: How do I add a slash command?**
A: `packages/cli/src/ui/commands/` (create new file) + register in command loader

**Q: Where is error handling?**
A: `packages/cli/src/utils/errors.ts` (CLI), `packages/core/src/utils/errorReporting.ts` (core)

**Q: How do I access the config?**
A: Via `Config` interface from `@google/gemini-cli-core` or `ConfigContext` in React

**Q: Where is telemetry?**
A: `packages/core/src/telemetry/`

**Q: How do I disable telemetry?**
A: Set `telemetry.enabled: false` in settings

---

## 🎯 Best Practices

1. **Always read the file first** before editing (use Read tool)
2. **Use TypeScript types** - they're your documentation
3. **Follow the event-driven pattern** - use MessageBus
4. **Test your changes** - write unit tests
5. **Check existing patterns** - look for similar code first
6. **Use the SDK** - for extensions, use `@google/gemini-cli-sdk`

---

**Last Updated:** 2026-02-17
**Version:** 0.30.0

---

*This quick reference is a living document. Update it as the codebase evolves!*
