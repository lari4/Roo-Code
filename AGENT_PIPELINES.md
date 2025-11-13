# Agent Pipelines Documentation

Документация всех схем работы агента, пайплайнов обработки данных и последовательностей вызовов промтов.

## Содержание

1. [System Prompt Generation Pipeline](#1-system-prompt-generation-pipeline)
2. [Task Execution Pipeline](#2-task-execution-pipeline)
3. [Tool Execution Pipeline](#3-tool-execution-pipeline)
4. [Mode Selection & Switching Pipeline](#4-mode-selection--switching-pipeline)
5. [MCP Integration Pipeline](#5-mcp-integration-pipeline)
6. [Codebase Search Pipeline](#6-codebase-search-pipeline)

---

## 1. System Prompt Generation Pipeline

**Файл:** `src/core/prompts/system.ts:148-241`

**Назначение:** Сборка полного системного промта из множественных компонентов в строго определенном порядке.

### Схема Pipeline

```
┌─────────────────────────────────────────────────────────────────┐
│                    SYSTEM_PROMPT()                               │
│                  (Main Entry Point)                              │
└────────────────────┬────────────────────────────────────────────┘
                     │
                     ▼
┌─────────────────────────────────────────────────────────────────┐
│              Initialization Phase                                │
│  • Get cwd, supportsComputerUse                                 │
│  • Check for codeIndexManager, mcpHub, diffStrategy             │
│  • Get SystemPromptSettings (maxConcurrentFileReads, etc)       │
└────────────────────┬────────────────────────────────────────────┘
                     │
                     ▼
┌─────────────────────────────────────────────────────────────────┐
│        Check for File-Based Custom System Prompt                │
│  • loadSystemPromptFile(context, currentMode)                   │
│  • If exists → Return custom prompt with interpolated vars      │
│  • Else → Continue to default generation                        │
└────────────────────┬────────────────────────────────────────────┘
                     │
                     ▼
┌─────────────────────────────────────────────────────────────────┐
│          generatePrompt() - 12 Section Assembly                  │
│                                                                  │
│  Section 1: ROLE (mode.roleDefinition)                          │
│  Section 2: Markdown Formatting                                 │
│  Section 3: Tool Descriptions Catalog                           │
│           ┌────────────────────────────────────┐                │
│           │ If Native Protocol:                │                │
│           │   → Include native tool JSONs      │                │
│           │ Else (XML):                        │                │
│           │   → Include XML tool descriptions  │                │
│           └────────────────────────────────────┘                │
│  Section 4: MCP Servers Info (if mcpHub)                        │
│  Section 5: Shared Tool Use Guidelines                          │
│  Section 6: Capabilities                                        │
│  Section 7: System Information                                  │
│  Section 8: Tool Use Guidelines (detailed)                      │
│  Section 9: Modes List                                          │
│  Section 10: Rules                                              │
│  Section 11: Objective                                          │
│  Section 12: Custom Instructions                                │
│            ┌─────────────────────────────────────┐              │
│            │ • Language preferences               │              │
│            │ • Global instructions                │              │
│            │ • Mode custom instructions           │              │
│            │ • .roo/rules/*.md files              │              │
│            │ • .roo/rules-{mode}/*.md files       │              │
│            │ • AGENTS.md                          │              │
│            │ • rules.md/.clinerules               │              │
│            └─────────────────────────────────────┘              │
└────────────────────┬────────────────────────────────────────────┘
                     │
                     ▼
┌─────────────────────────────────────────────────────────────────┐
│                   Return Assembled Prompt                        │
│  • All sections joined with newlines                            │
│  • Ready for API request                                        │
└─────────────────────────────────────────────────────────────────┘
```

### Триггеры Регенерации

```
Regeneration Triggers:
├── Mode Switch
│   └── Mode change → New roleDefinition + customInstructions
├── Custom Instructions Change
│   └── .roo/rules/ files modified → Reload custom instructions
├── MCP Server Update
│   └── MCP config changed → New tool descriptions in catalog
└── Settings Change
    └── maxConcurrentFileReads, toolProtocol, etc. → Rebuild sections
```

### Условная Логика

```
Conditional Sections:
│
├─ IF codeIndexManager available
│  └─> Include codebase_search in tools
│     └─> Add semantic search instructions in Rules/Objective
│
├─ IF mcpHub available
│  └─> Include MCP servers section
│     └─> Add MCP tools to tool catalog
│
├─ IF supportsComputerUse
│  └─> Include browser_action tool
│     └─> Add browser-related rules
│
├─ IF diffStrategy available
│  └─> Include apply_diff tool
│     └─> Add surgical editing instructions
│
└─ IF toolProtocol == "native"
   └─> Use native tool JSON format
   └─> Else: Use XML tool descriptions
```

### Ключевые Функции

- **SYSTEM_PROMPT()** - главная функция генерации
- **generatePrompt()** - сборка секций
- **loadSystemPromptFile()** - загрузка custom system prompts
- **addCustomInstructions()** - загрузка всех custom rules

### Данные Между Шагами

```
Input:
  • context: vscode.ExtensionContext
  • currentMode: string (slug режима)

Processing:
  • roleDefinition → Section 1
  • availableTools → Section 3
  • mcpServers → Section 4
  • customRules → Section 12

Output:
  • Complete system prompt string
  • Ready for API request
```

---

## 2. Task Execution Pipeline

**Файл:** `src/core/task/Task.ts`

**Назначение:** Основной цикл выполнения задач - обработка сообщений пользователя, вызовы API, выполнение инструментов и управление состоянием.

### Общая Схема Pipeline

```
┌────────────────────────────────────────────────────────────┐
│                   Task Initialization                       │
│  Entry Points:                                             │
│  • startTask(task: string)                                 │
│  • resumeTaskFromHistory(historyItem)                      │
│  • startSubtask(content)                                   │
└──────────────────┬─────────────────────────────────────────┘
                   │
                   ▼
┌────────────────────────────────────────────────────────────┐
│           Setup Phase                                       │
│  1. Load/Initialize State                                  │
│     • this.taskId = generateId()                           │
│     • this.dirAbsolutePath = cwd                           │
│     • this.apiConversationHistory = []                     │
│  2. Setup Provider (Anthropic/OpenRouter/etc)              │
│  3. Setup Mode                                             │
│     • await this.taskModeReady promise                     │
│  4. Add User Message                                       │
│     • pushToolResult(userMessage)                          │
└──────────────────┬─────────────────────────────────────────┘
                   │
                   ▼
┌────────────────────────────────────────────────────────────┐
│        Main Task Loop - initiateTaskLoop()                 │
│                                                            │
│  Loop Until: attempt_completion OR user intervention      │
│                                                            │
│  ┌─────────────────────────────────────────────┐          │
│  │ 1. Regenerate System Prompt                 │          │
│  │    • SYSTEM_PROMPT(context, currentMode)    │          │
│  │                                             │          │
│  │ 2. Build API Request                        │          │
│  │    • systemPrompt                           │          │
│  │    • apiConversationHistory                 │          │
│  │    • tools catalog                          │          │
│  │                                             │          │
│  │ 3. Call recursivelyMakeClineRequests()      │          │
│  │    ├─> Make API request                     │          │
│  │    ├─> Stream response                      │          │
│  │    ├─> Execute tools                        │          │
│  │    └─> Accumulate results                   │          │
│  │                                             │          │
│  │ 4. Check Loop Continuation                  │          │
│  │    • If attempt_completion → Break          │          │
│  │    • If user feedback → Continue            │          │
│  │    • If error → Handle and continue/break   │          │
│  └─────────────────────────────────────────────┘          │
└──────────────────┬─────────────────────────────────────────┘
                   │
                   ▼
┌────────────────────────────────────────────────────────────┐
│                Task Completion                             │
│  • Save state                                              │
│  • Update UI                                               │
│  • Return result                                           │
└────────────────────────────────────────────────────────────┘
```

### Детальная Схема recursivelyMakeClineRequests()

```
┌────────────────────────────────────────────────────────────────────┐
│       recursivelyMakeClineRequests()                               │
│       (Core Request/Response Loop)                                 │
└────────────────┬───────────────────────────────────────────────────┘
                 │
                 ▼
┌────────────────────────────────────────────────────────────────────┐
│  Phase 1: Pre-Request Preparation                                  │
│  • Lock presentAssistantMessageLocked                              │
│  • Check userMessageContentReady flag                              │
│  • If not ready → Wait for user to provide content                 │
└────────────────┬───────────────────────────────────────────────────┘
                 │
                 ▼
┌────────────────────────────────────────────────────────────────────┐
│  Phase 2: API Request                                              │
│  • attemptApiRequest(previousApiReqIndex)                          │
│    ├─ Build request params                                         │
│    │  • systemPrompt (regenerated each time!)                      │
│    │  • messages: apiConversationHistory                           │
│    │  • tools: available tools for current mode                    │
│    ├─ Call API provider                                            │
│    │  └─> Returns streaming response                               │
│    └─ Return ApiHistoryItem with response                          │
└────────────────┬───────────────────────────────────────────────────┘
                 │
                 ▼
┌────────────────────────────────────────────────────────────────────┐
│  Phase 3: Response Streaming & Processing                          │
│  • presentAssistantMessage(apiHistoryItem)                         │
│    ├─ Stream assistant's text response                             │
│    ├─ Parse tool calls (XML or Native protocol)                    │
│    │  └─> Extract tool name, parameters                            │
│    ├─ Display in UI incrementally                                  │
│    └─ Wait for stream completion                                   │
└────────────────┬───────────────────────────────────────────────────┘
                 │
                 ▼
┌────────────────────────────────────────────────────────────────────┐
│  Phase 4: Tool Execution                                           │
│  • For each tool call in response:                                 │
│    ┌──────────────────────────────────────────────┐               │
│    │ 1. Validate Tool Use                         │               │
│    │    • Check mode permissions                  │               │
│    │    • Check file restrictions                 │               │
│    │                                              │               │
│    │ 2. Get User Approval (if needed)             │               │
│    │    • Some tools require confirmation         │               │
│    │    • alwaysAllow tools skip this             │               │
│    │                                              │               │
│    │ 3. Execute Tool                              │               │
│    │    • Call tool.execute(params)               │               │
│    │    • Get result                              │               │
│    │                                              │               │
│    │ 4. Accumulate Result                         │               │
│    │    • Add to userMessageContent array         │               │
│    │    • Format for API (text + images)          │               │
│    └──────────────────────────────────────────────┘               │
└────────────────┬───────────────────────────────────────────────────┘
                 │
                 ▼
┌────────────────────────────────────────────────────────────────────┐
│  Phase 5: Loop Decision                                            │
│  • Check if attempt_completion was used                            │
│  │  └─> YES: Task complete, exit loop                             │
│  │  └─> NO: Continue to next iteration                            │
│  •                                                                 │
│  • If continuing:                                                  │
│  │  1. Add userMessageContent to apiConversationHistory            │
│  │  2. Set userMessageContentReady = true                          │
│  │  3. RECURSIVE CALL: recursivelyMakeClineRequests()             │
│  │     └─> Creates queue-based stack of requests                  │
│  │                                                                 │
│  • If error or user intervention:                                 │
│  │  └─> Handle appropriately and decide continuation              │
└────────────────┬───────────────────────────────────────────────────┘
                 │
                 ▼
┌────────────────────────────────────────────────────────────────────┐
│  Return to initiateTaskLoop() or Exit                              │
└────────────────────────────────────────────────────────────────────┘
```

### Data Flow Diagram

```
User Input
    │
    ▼
┌─────────────────────────────────────┐
│  Initial userMessage                │
│  { type: "text", text: "..." }      │
└──────────────┬──────────────────────┘
               │
               ▼
┌─────────────────────────────────────┐
│  apiConversationHistory             │
│  [                                  │
│    { role: "user",                  │
│      content: [userMessage] }       │
│  ]                                  │
└──────────────┬──────────────────────┘
               │
               ▼
       ┌──────────────┐
       │  API Request │
       └──────┬───────┘
              │
              ▼
┌─────────────────────────────────────┐
│  API Response                       │
│  {                                  │
│    role: "assistant",               │
│    content: [                       │
│      { type: "text", text: "..." }, │
│      { type: "tool_use",            │
│        name: "read_file",           │
│        input: {...} }               │
│    ]                                │
│  }                                  │
└──────────────┬──────────────────────┘
               │
               ▼
┌─────────────────────────────────────┐
│  Tool Execution Results             │
│  userMessageContent = [             │
│    { type: "text",                  │
│      text: "Tool executed..." },    │
│    { type: "tool_result",           │
│      tool_use_id: "...",            │
│      content: "file contents..." }  │
│  ]                                  │
└──────────────┬──────────────────────┘
               │
               ▼
┌─────────────────────────────────────┐
│  Add to apiConversationHistory      │
│  [                                  │
│    ... previous messages,           │
│    { role: "assistant", ... },      │
│    { role: "user",                  │
│      content: userMessageContent }  │
│  ]                                  │
└──────────────┬──────────────────────┘
               │
               ▼
          Next API Request
          (loop continues)
```

### Critical Synchronization Points

```
Synchronization Mechanisms:

1. presentAssistantMessageLocked (Mutex)
   └─> Prevents concurrent streaming
       Only one API response can be processed at a time

2. userMessageContentReady (Flag)
   └─> Gates next API request
       Must wait for all tool executions to complete

3. taskModeReady (Promise)
   └─> Ensures mode is initialized before starting task
       Waits for mode configuration to load

4. Streaming Completion
   └─> Must wait for full response before executing tools
       Prevents partial tool calls
```

### Ключевые Функции

**Initialization:**
- `startTask(task: string)` - начало новой задачи
- `resumeTaskFromHistory(item)` - возобновление из истории
- `startSubtask(content)` - создание подзадачи

**Main Loop:**
- `initiateTaskLoop()` - главный цикл задачи
- `recursivelyMakeClineRequests()` - рекурсивный request/response loop

**API Communication:**
- `attemptApiRequest(index)` - вызов API провайдера
- `buildApiRequest()` - построение request params

**Message Processing:**
- `presentAssistantMessage(item)` - обработка и отображение ответа
- `addToApiConversationHistory()` - добавление в историю

**Tool Execution:**
- `executeTool(tool, params)` - выполнение инструмента
- `formatToolResult(result)` - форматирование результата

### State Management

```
Task State:
├── taskId: string (unique identifier)
├── dirAbsolutePath: string (working directory)
├── apiConversationHistory: Message[] (full conversation)
├── clineMessages: ClineMessage[] (UI display history)
├── userMessageContent: MessageContent[] (accumulated tool results)
├── userMessageContentReady: boolean (ready for next request)
└── presentAssistantMessageLocked: boolean (streaming in progress)
```

---
