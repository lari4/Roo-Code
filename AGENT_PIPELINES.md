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

## 3. Tool Execution Pipeline

**Файлы:** `src/core/tools/BaseTool.ts`, `src/core/tools/parser/NativeToolCallParser.ts`

**Назначение:** Парсинг, валидация и выполнение инструментов из ответов AI.

### Схема Pipeline

```
┌────────────────────────────────────────────────────────────┐
│         Tool Call Received from API                        │
│  (In assistant message content)                            │
└──────────────────┬─────────────────────────────────────────┘
                   │
                   ▼
┌────────────────────────────────────────────────────────────┐
│  Phase 1: Protocol Detection & Parsing                     │
│                                                            │
│  IF Native Protocol (OpenAI-style):                        │
│  ┌──────────────────────────────────────┐                 │
│  │ NativeToolCallParser.parse()         │                 │
│  │  • Extract tool_call_id              │                 │
│  │  • Extract name                      │                 │
│  │  • Parse JSON arguments              │                 │
│  │  • Validate against Zod schemas      │                 │
│  │  • Return typed nativeArgs           │                 │
│  └──────────────────────────────────────┘                 │
│                                                            │
│  IF XML Protocol (Legacy):                                 │
│  ┌──────────────────────────────────────┐                 │
│  │ XML Parser                           │                 │
│  │  • Parse XML tags                    │                 │
│  │  • Extract parameters                │                 │
│  │  • Return params object              │                 │
│  └──────────────────────────────────────┘                 │
└──────────────────┬─────────────────────────────────────────┘
                   │
                   ▼
┌────────────────────────────────────────────────────────────┐
│  Phase 2: Tool Validation                                  │
│                                                            │
│  1. Check Tool Availability                                │
│     • Is tool in available tools list?                     │
│     • Is tool enabled for current mode?                    │
│                                                            │
│  2. Mode-Based Restrictions                                │
│     ┌────────────────────────────────────┐                │
│     │ validateToolUse(tool, mode)        │                │
│     │  • Check mode.groups                │                │
│     │  • Verify tool group allowed        │                │
│     │  • For edit tools:                  │                │
│     │    └─> Check fileRegex restrictions │                │
│     └────────────────────────────────────┘                │
│                                                            │
│  3. Parameter Validation                                   │
│     • Required params present?                             │
│     • Correct types?                                       │
│     • Valid values?                                        │
└──────────────────┬─────────────────────────────────────────┘
                   │
                   ▼
┌────────────────────────────────────────────────────────────┐
│  Phase 3: User Approval (if needed)                        │
│                                                            │
│  Check if tool requires approval:                          │
│  ├─ alwaysAllow tools → Skip approval                      │
│  ├─ Dangerous operations → Require approval                │
│  └─ User settings → Check preferences                      │
│                                                            │
│  If approval needed:                                       │
│    └─> Show confirmation dialog                            │
│       ├─ Approve → Continue                                │
│       ├─ Reject → Return error                             │
│       └─ Modify → Update params and continue               │
└──────────────────┬─────────────────────────────────────────┘
                   │
                   ▼
┌────────────────────────────────────────────────────────────┐
│  Phase 4: Tool Execution                                   │
│                                                            │
│  Tool Instance Lifecycle:                                  │
│  ┌──────────────────────────────────────┐                 │
│  │ BaseTool Abstract Pattern            │                 │
│  │                                      │                 │
│  │ 1. constructor(cwd, ...)             │                 │
│  │    • Initialize tool instance        │                 │
│  │                                      │                 │
│  │ 2. handle(params)                    │                 │
│  │    • Public interface                │                 │
│  │    • Parameter preprocessing         │                 │
│  │                                      │                 │
│  │ 3. execute(params) [abstract]        │                 │
│  │    • Actual tool logic               │                 │
│  │    • Implemented by each tool        │                 │
│  │                                      │                 │
│  │ 4. Return ToolResult                 │                 │
│  │    • Success/failure                 │                 │
│  │    • Output data                     │                 │
│  │    • Error messages                  │                 │
│  └──────────────────────────────────────┘                 │
└──────────────────┬─────────────────────────────────────────┘
                   │
                   ▼
┌────────────────────────────────────────────────────────────┐
│  Phase 5: Result Formatting                                │
│                                                            │
│  Format for API:                                           │
│  ┌──────────────────────────────────────┐                 │
│  │ IF Native Protocol:                  │                 │
│  │ {                                    │                 │
│  │   type: "tool_result",               │                 │
│  │   tool_use_id: "...",                │                 │
│  │   content: [                         │                 │
│  │     { type: "text", text: "..." },   │                 │
│  │     { type: "image", source: {...} } │                 │
│  │   ]                                  │                 │
│  │ }                                    │                 │
│  └──────────────────────────────────────┘                 │
│                                                            │
│  ┌──────────────────────────────────────┐                 │
│  │ IF XML Protocol:                     │                 │
│  │ <tool_result>                        │                 │
│  │   <output>...</output>               │                 │
│  │   <error>...</error>                 │                 │
│  │ </tool_result>                       │                 │
│  └──────────────────────────────────────┘                 │
└──────────────────┬─────────────────────────────────────────┘
                   │
                   ▼
┌────────────────────────────────────────────────────────────┐
│  Return to Task Pipeline                                   │
│  • Add to userMessageContent                               │
│  • Continue with next tool or next API request             │
└────────────────────────────────────────────────────────────┘
```

### Dual Protocol Support

```
Protocol Handling:

Native Protocol (OpenAI):
├── Tool Call Structure:
│   {
│     id: "call_abc123",
│     type: "function",
│     function: {
│       name: "read_file",
│       arguments: "{\"path\":\"src/app.ts\"}"
│     }
│   }
│
├── Parsing:
│   └─> NativeToolCallParser.parse()
│       • JSON.parse(arguments)
│       • Zod validation
│       • Type-safe nativeArgs
│
└── Benefits:
    • Type safety
    • Better error messages
    • Standard format

XML Protocol (Legacy):
├── Tool Call Structure:
│   <read_file>
│     <path>src/app.ts</path>
│   </read_file>
│
├── Parsing:
│   └─> XML string parsing
│       • Extract tags
│       • Build params object
│
└── Compatibility:
    • Backward compatible
    • Custom format
    • More verbose
```

### Tool Instance Management

```
Tool Singleton Pattern:
(in presentAssistantMessage)

const tools = {
  execute_command: new ExecuteCommandTool(cwd, ...),
  read_file: new ReadFileTool(cwd, ...),
  write_to_file: new WriteToFileTool(cwd, ...),
  apply_diff: new ApplyDiffTool(cwd, ...),
  search_files: new SearchFilesTool(cwd, ...),
  list_files: new ListFilesTool(cwd, ...),
  list_code_definition_names: new ListCodeDefinitionNamesTool(cwd, ...),
  codebase_search: new CodebaseSearchTool(cwd, ...),
  ask_followup_question: new AskFollowupQuestionTool(...),
  attempt_completion: new AttemptCompletionTool(...),
  // ... 20+ tools total
}

For each tool call:
  1. Get tool instance: tools[toolName]
  2. Call: await tool.handle(params)
  3. Get result
```

### File Restriction Validation

```
Edit Tool Restrictions:
(Example from Mode Config)

Mode: architect
  groups:
    - read
    - - edit
      - fileRegex: \\.md$
        description: Markdown files only

Validation Flow:
  User asks to edit "app.js"
    │
    ▼
  Check mode.groups for edit
    │
    ▼
  Find edit group with fileRegex: \\.md$
    │
    ▼
  Test: "app.js" matches /\\.md$/
    │
    ▼
  Result: NO MATCH
    │
    ▼
  Return FileRestrictionError:
    "Cannot edit app.js in architect mode.
     Only files matching \\.md$ are allowed."
```

### Ключевые Компоненты

**Parsing:**
- `NativeToolCallParser.parse()` - парсинг native tool calls
- `parseXmlToolCall()` - парсинг XML tool calls

**Validation:**
- `validateToolUse()` - проверка permissions
- `checkFileRestrictions()` - проверка file patterns
- `validateParams()` - проверка параметров

**Execution:**
- `BaseTool.handle()` - публичный interface
- `BaseTool.execute()` - abstract метод реализации
- `formatToolResult()` - форматирование результата

**Tools (20+):**
- File Operations: read_file, write_to_file, apply_diff, insert_content
- Search: search_files, codebase_search, list_code_definition_names
- System: execute_command, browser_action
- Interaction: ask_followup_question, attempt_completion
- Mode: new_task, switch_mode, fetch_instructions
- MCP: use_mcp_tool, access_mcp_resource
- Utilities: update_todo_list, generate_image

---

## 4. Mode Selection & Switching Pipeline

**Файл:** `src/shared/modes.ts`

**Назначение:** Определение, загрузка и переключение режимов работы агента.

### Схема Mode Resolution

```
┌────────────────────────────────────────────────────────────┐
│         Mode Resolution Request                            │
│  • getModeBySlug(slug, context)                            │
│  • getAllModesWithPrompts(context)                         │
└──────────────────┬─────────────────────────────────────────┘
                   │
                   ▼
┌────────────────────────────────────────────────────────────┐
│  Phase 1: Load Mode Configurations                         │
│                                                            │
│  Priority Order:                                           │
│  ┌──────────────────────────────────────┐                 │
│  │ 1. Custom Modes (Highest Priority)   │                 │
│  │    Sources:                          │                 │
│  │    ├─ .roomodes (workspace)          │                 │
│  │    │  • Project-specific modes       │                 │
│  │    │  • Overrides global modes       │                 │
│  │    │                                 │                 │
│  │    └─ ~/.roo/custom-modes.yaml       │                 │
│  │       • User global modes            │                 │
│  │       • Shared across projects       │                 │
│  └──────────────────────────────────────┘                 │
│                                                            │
│  ┌──────────────────────────────────────┐                 │
│  │ 2. Prompt Components (Optional)      │                 │
│  │    • Custom prompt additions         │                 │
│  │    • Extend built-in modes           │                 │
│  └──────────────────────────────────────┘                 │
│                                                            │
│  ┌──────────────────────────────────────┐                 │
│  │ 3. Built-in Modes (Fallback)         │                 │
│  │    • Default modes shipped with app  │                 │
│  │    • code, architect, ask, etc.      │                 │
│  └──────────────────────────────────────┘                 │
└──────────────────┬─────────────────────────────────────────┘
                   │
                   ▼
┌────────────────────────────────────────────────────────────┐
│  Phase 2: Mode Merging & Validation                        │
│                                                            │
│  1. Merge Modes by Slug                                    │
│     • .roomodes overrides ~/.roo/custom-modes.yaml        │
│     • Custom overrides built-in                            │
│                                                            │
│  2. Validate Mode Structure                                │
│     ✓ Required fields:                                     │
│       • slug (unique identifier)                           │
│       • name (display name)                                │
│       • roleDefinition (non-empty)                         │
│       • groups (array, can be empty)                       │
│     ✓ Optional fields:                                     │
│       • description                                        │
│       • whenToUse                                          │
│       • customInstructions                                 │
│                                                            │
│  3. Build Mode Object                                      │
│     {                                                      │
│       slug: "test",                                        │
│       name: "🧪 Test",                                     │
│       roleDefinition: "...",                               │
│       groups: ["read", ["edit", {fileRegex: "..."}]],     │
│       customInstructions: "...",                           │
│       whenToUse: "..."                                     │
│     }                                                      │
└──────────────────┬─────────────────────────────────────────┘
                   │
                   ▼
┌────────────────────────────────────────────────────────────┐
│  Phase 3: Tool Filtering                                   │
│                                                            │
│  Filter Available Tools Based on Mode Groups:              │
│  ┌──────────────────────────────────────┐                 │
│  │ Mode Groups → Tool Availability       │                 │
│  │                                      │                 │
│  │ groups: ["read", "edit", "command"]  │                 │
│  │    ↓                                 │                 │
│  │ Available Tools:                     │                 │
│  │  • read_file                         │                 │
│  │  • search_files                      │                 │
│  │  • list_files                        │                 │
│  │  • list_code_definition_names        │                 │
│  │  • apply_diff (if fileRegex matches) │                 │
│  │  • write_to_file (if fileRegex...)   │                 │
│  │  • execute_command                   │                 │
│  │  • ask_followup_question (always)    │                 │
│  │  • attempt_completion (always)       │                 │
│  └──────────────────────────────────────┘                 │
│                                                            │
│  Group Mappings:                                           │
│  • "read" → read_file, search_files, list_files, etc.     │
│  • "edit" → apply_diff, write_to_file, insert_content     │
│  • "command" → execute_command                             │
│  • "browser" → browser_action                              │
│  • "mcp" → use_mcp_tool, access_mcp_resource              │
└──────────────────┬─────────────────────────────────────────┘
                   │
                   ▼
┌────────────────────────────────────────────────────────────┐
│  Return Mode Configuration                                 │
│  • Ready for use in task                                   │
│  • Applied to system prompt generation                     │
└────────────────────────────────────────────────────────────┘
```

### Mode Switching Flow

```
┌────────────────────────────────────────────────────────────┐
│  User Requests Mode Switch                                 │
│  • switch_mode tool called                                 │
│  • Or manual mode selection in UI                          │
└──────────────────┬─────────────────────────────────────────┘
                   │
                   ▼
┌────────────────────────────────────────────────────────────┐
│  Step 1: Validate Switch Request                           │
│  • Check if target mode exists                             │
│  • Validate mode slug                                      │
│  • If invalid → Return error                               │
└──────────────────┬─────────────────────────────────────────┘
                   │
                   ▼
┌────────────────────────────────────────────────────────────┐
│  Step 2: Get User Approval (if needed)                     │
│  • Show mode switch confirmation                           │
│  • Display mode description                                │
│  • User approves/rejects                                   │
└──────────────────┬─────────────────────────────────────────┘
                   │
                   ▼
┌────────────────────────────────────────────────────────────┐
│  Step 3: Switch Provider & Mode                            │
│  ┌──────────────────────────────────────┐                 │
│  │ Task.switchToMode(newMode)           │                 │
│  │  1. Update currentMode                │                 │
│  │  2. Update provider (if needed)       │                 │
│  │  3. Update available tools            │                 │
│  │  4. Trigger prompt regeneration       │                 │
│  │  5. Update UI                         │                 │
│  └──────────────────────────────────────┘                 │
└──────────────────┬─────────────────────────────────────────┘
                   │
                   ▼
┌────────────────────────────────────────────────────────────┐
│  Step 4: Continue Task with New Mode                       │
│  • Next API request uses new system prompt                 │
│  • New tool restrictions apply                             │
│  • Mode-specific instructions active                       │
└────────────────────────────────────────────────────────────┘
```

### File Restriction Example

```
Mode: test (🧪 Test)
  groups:
    - read
    - - edit
      - fileRegex: (__tests__/.*|\.test\.(ts|tsx)$|vitest\.config\.ts$)
        description: Test files, mocks, and Vitest configuration

Allowed Files:
  ✓ src/__tests__/user.test.ts
  ✓ components/Button.test.tsx
  ✓ vitest.config.ts
  ✗ src/app.ts (rejected - not a test file)
  ✗ components/Button.tsx (rejected - not a test file)

When agent tries to edit src/app.ts in test mode:
  → FileRestrictionError
  → Message: "Cannot edit src/app.ts in test mode.
              Only files matching (__tests__/.*|\.test\.(ts|tsx)$|...) are allowed."
```

### Ключевые Функции

- `getModeBySlug(slug, context)` - получение режима по slug
- `getAllModesWithPrompts(context)` - все режимы с промтами
- `loadCustomModes(path)` - загрузка custom modes из файла
- `mergeModesWithPromptComponents()` - слияние с prompt components
- `validateModeConfig(mode)` - валидация конфигурации
- `filterToolsByModeGroups(mode)` - фильтрация инструментов

---

## 5. MCP Integration Pipeline

**Файл:** `src/services/mcp/McpHub.ts`

**Назначение:** Управление MCP (Model Context Protocol) серверами - подключение, lifecycle, и доступ к их инструментам/ресурсам.

### Схема MCP Lifecycle

```
┌────────────────────────────────────────────────────────────┐
│         McpHub Initialization                              │
│  • new McpHub()                                            │
│  • Load configuration                                      │
└──────────────────┬─────────────────────────────────────────┘
                   │
                   ▼
┌────────────────────────────────────────────────────────────┐
│  Phase 1: Configuration Loading                            │
│                                                            │
│  3 Configuration Sources (merged):                         │
│  ┌──────────────────────────────────────┐                 │
│  │ 1. Global Settings                   │                 │
│  │    • VSCode user settings             │                 │
│  │    • ~/.roo/mcp-config.json           │                 │
│  │                                      │                 │
│  │ 2. Project Settings                  │                 │
│  │    • .roo/mcp-config.json             │                 │
│  │    • Workspace-specific config        │                 │
│  │                                      │                 │
│  │ 3. Environment Variables             │                 │
│  │    • Process.env overrides            │                 │
│  │    • Sensitive credentials            │                 │
│  └──────────────────────────────────────┘                 │
│                                                            │
│  Configuration Format:                                     │
│  {                                                         │
│    "mcpServers": {                                         │
│      "weather": {                                          │
│        "command": "node",                                  │
│        "args": ["/path/to/server.js"],                    │
│        "env": { "API_KEY": "..." },                       │
│        "disabled": false,                                  │
│        "timeout": 60,                                      │
│        "alwaysAllow": ["get_weather"],                    │
│        "disabledTools": []                                 │
│      }                                                     │
│    }                                                       │
│  }                                                         │
└──────────────────┬─────────────────────────────────────────┘
                   │
                   ▼
┌────────────────────────────────────────────────────────────┐
│  Phase 2: Server Connection                                │
│                                                            │
│  For each server in config:                                │
│  ┌──────────────────────────────────────┐                 │
│  │ Connection Type Detection            │                 │
│  │                                      │                 │
│  │ IF has "command" field:              │                 │
│  │   → Stdio Transport                  │                 │
│  │      • Spawn process                 │                 │
│  │      • Connect via stdin/stdout      │                 │
│  │      • Local server                  │                 │
│  │                                      │                 │
│  │ ELSE IF has "url" field:             │                 │
│  │   → SSE Transport                    │                 │
│  │      • HTTP connection               │                 │
│  │      • Server-Sent Events            │                 │
│  │      • Remote server                 │                 │
│  │                                      │                 │
│  │ ELSE IF has "streamableUrl":         │                 │
│  │   → Streamable HTTP Transport        │                 │
│  │      • HTTP with streaming           │                 │
│  │      • Remote server                 │                 │
│  └──────────────────────────────────────┘                 │
└──────────────────┬─────────────────────────────────────────┘
                   │
                   ▼
┌────────────────────────────────────────────────────────────┐
│  Phase 3: Capability Discovery                             │
│                                                            │
│  After successful connection:                              │
│  ┌──────────────────────────────────────┐                 │
│  │ 1. List Tools                        │                 │
│  │    • client.listTools()              │                 │
│  │    • Get tool schemas                │                 │
│  │    • Store tool definitions          │                 │
│  │                                      │                 │
│  │ 2. List Resources                    │                 │
│  │    • client.listResources()          │                 │
│  │    • Get resource URIs               │                 │
│  │    • Store resource info             │                 │
│  │                                      │                 │
│  │ 3. List Prompts (if supported)       │                 │
│  │    • client.listPrompts()            │                 │
│  │    • Get prompt templates            │                 │
│  └──────────────────────────────────────┘                 │
│                                                            │
│  Example Discovered Tools:                                 │
│  [                                                         │
│    {                                                       │
│      name: "get_weather",                                 │
│      description: "Get current weather...",               │
│      inputSchema: { type: "object", ... }                 │
│    }                                                       │
│  ]                                                         │
└──────────────────┬─────────────────────────────────────────┘
                   │
                   ▼
┌────────────────────────────────────────────────────────────┐
│  Phase 4: Integration with System                          │
│                                                            │
│  1. Add to System Prompt                                   │
│     • Include MCP servers section                          │
│     • List available tools                                 │
│     • Document tool usage                                  │
│                                                            │
│  2. Make Tools Available                                   │
│     • Add use_mcp_tool to agent tools                      │
│     • Add access_mcp_resource to agent tools               │
│                                                            │
│  3. Reference Counting                                     │
│     • Track tool usage                                     │
│     • Manage server lifecycle                              │
│     • Prevent premature disposal                           │
└──────────────────┬─────────────────────────────────────────┘
                   │
                   ▼
┌────────────────────────────────────────────────────────────┐
│  Runtime: Tool Execution                                   │
│  (When agent uses MCP tool)                                │
└────────────────────────────────────────────────────────────┘
```

### MCP Tool Call Flow

```
┌────────────────────────────────────────────────────────────┐
│  Agent calls use_mcp_tool                                  │
│  {                                                         │
│    server_name: "weather",                                │
│    tool_name: "get_weather",                              │
│    arguments: { city: "San Francisco" }                   │
│  }                                                         │
└──────────────────┬─────────────────────────────────────────┘
                   │
                   ▼
┌────────────────────────────────────────────────────────────┐
│  Step 1: Resolve Server                                    │
│  • mcpHub.getServer("weather")                             │
│  • Check if server is connected                            │
│  • Verify tool exists on server                            │
└──────────────────┬─────────────────────────────────────────┘
                   │
                   ▼
┌────────────────────────────────────────────────────────────┐
│  Step 2: User Approval Check                               │
│  • Check if tool in alwaysAllow list                       │
│  • If not → Show confirmation dialog                       │
│  • User approves/rejects                                   │
└──────────────────┬─────────────────────────────────────────┘
                   │
                   ▼
┌────────────────────────────────────────────────────────────┐
│  Step 3: Call MCP Server                                   │
│  • client.callTool(tool_name, arguments)                   │
│  • Wait for response (with timeout)                        │
│  • Handle errors                                           │
└──────────────────┬─────────────────────────────────────────┘
                   │
                   ▼
┌────────────────────────────────────────────────────────────┐
│  Step 4: Process Response                                  │
│  • Parse tool result                                       │
│  • Format for agent                                        │
│  • Return to task pipeline                                 │
└────────────────────────────────────────────────────────────┘
```

### Configuration Update Flow

```
Config File Changed
    │
    ▼
┌─────────────────────────────────────┐
│  Debounced Config Update (500ms)    │
│  • Prevents rapid restarts          │
│  • Accumulates multiple changes     │
└──────────────┬──────────────────────┘
               │
               ▼
┌─────────────────────────────────────┐
│  Compare Old vs New Config          │
│  • Detect added servers             │
│  • Detect removed servers            │
│  • Detect modified servers           │
└──────────────┬──────────────────────┘
               │
               ▼
┌─────────────────────────────────────┐
│  Update Servers                     │
│  • Stop removed servers             │
│  • Start new servers                │
│  • Restart modified servers         │
└──────────────┬──────────────────────┘
               │
               ▼
┌─────────────────────────────────────┐
│  Trigger System Prompt Regeneration │
│  • Update MCP tools in prompt       │
│  • Ready for next task              │
└─────────────────────────────────────┘
```

### Reference Counting

```
Server Lifecycle Management:

acquire(serverId):
  refCount[serverId]++
  └─> Prevents disposal while in use

release(serverId):
  refCount[serverId]--
  if refCount[serverId] === 0:
    └─> Safe to dispose server

Example:
  Task starts:
    └─> acquire("weather")     // refCount = 1
  MCP tool used:
    └─> (already acquired)     // refCount = 1
  Task completes:
    └─> release("weather")     // refCount = 0
                               // → Safe to dispose
```

---

## 6. Codebase Search Pipeline

**Файл:** `src/services/code-index/manager.ts`

**Назначение:** Семантический поиск по кодовой базе с использованием векторных embeddings.

### Схема Indexing Pipeline

```
┌────────────────────────────────────────────────────────────┐
│         CodeIndexManager Initialization                    │
│  • new CodeIndexManager(cwd)                               │
│  • Check feature enabled & configured                      │
└──────────────────┬─────────────────────────────────────────┘
                   │
                   ▼
┌────────────────────────────────────────────────────────────┐
│  State: STANDBY → INITIALIZING                             │
│  • Validate configuration                                  │
│  • Setup embedding provider                                │
│  • Check for existing index                                │
└──────────────────┬─────────────────────────────────────────┘
                   │
                   ▼
┌────────────────────────────────────────────────────────────┐
│  Phase 1: Initial Scan                                     │
│                                                            │
│  Scan Workspace:                                           │
│  ┌──────────────────────────────────────┐                 │
│  │ 1. List all files in workspace       │                 │
│  │    • Recursive directory traversal   │                 │
│  │    • Respect .gitignore              │                 │
│  │    • Filter by supported extensions  │                 │
│  │                                      │                 │
│  │ 2. Filter files                      │                 │
│  │    • Skip node_modules, .git, etc.   │                 │
│  │    • Only index code files           │                 │
│  │    • Apply size limits               │                 │
│  │                                      │                 │
│  │ 3. Collect file metadata             │                 │
│  │    • File path                       │                 │
│  │    • Last modified time              │                 │
│  │    • File size                       │                 │
│  └──────────────────────────────────────┘                 │
└──────────────────┬─────────────────────────────────────────┘
                   │
                   ▼
┌────────────────────────────────────────────────────────────┐
│  Phase 2: File Processing & Chunking                       │
│                                                            │
│  For each file:                                            │
│  ┌──────────────────────────────────────┐                 │
│  │ 1. Read file content                 │                 │
│  │    • Load file into memory           │                 │
│  │    • Parse as text                   │                 │
│  │                                      │                 │
│  │ 2. Extract code structure            │                 │
│  │    • Parse AST (if possible)         │                 │
│  │    • Identify functions, classes     │                 │
│  │    • Extract comments                │                 │
│  │                                      │                 │
│  │ 3. Create chunks                     │                 │
│  │    • Split into semantic units       │                 │
│  │    • ~200-500 token chunks           │                 │
│  │    • Maintain context overlap        │                 │
│  │                                      │                 │
│  │ 4. Generate chunk metadata           │                 │
│  │    {                                 │                 │
│  │      id: "file-path:chunk-123",      │                 │
│  │      filePath: "src/app.ts",         │                 │
│  │      startLine: 10,                  │                 │
│  │      endLine: 50,                    │                 │
│  │      content: "...",                 │                 │
│  │      type: "function"                │                 │
│  │    }                                 │                 │
│  └──────────────────────────────────────┘                 │
└──────────────────┬─────────────────────────────────────────┘
                   │
                   ▼
┌────────────────────────────────────────────────────────────┐
│  Phase 3: Embedding Generation                             │
│                                                            │
│  Batch Processing:                                         │
│  ┌──────────────────────────────────────┐                 │
│  │ For each batch of chunks (e.g., 100) │                 │
│  │                                      │                 │
│  │ 1. Call Embedding Provider           │                 │
│  │    • Local: Transformers.js          │                 │
│  │    • Cloud: OpenAI, Voyage, etc.     │                 │
│  │                                      │                 │
│  │ 2. Generate embeddings               │                 │
│  │    • Input: chunk.content            │                 │
│  │    • Output: float[] (e.g., 768-dim) │                 │
│  │                                      │                 │
│  │ 3. Store embeddings                  │                 │
│  │    • Vector database (or in-memory)  │                 │
│  │    • Map: chunkId → embedding        │                 │
│  └──────────────────────────────────────┘                 │
│                                                            │
│  State: INDEXING                                           │
│  • Progress updates to UI                                  │
│  • Cancellable                                             │
└──────────────────┬─────────────────────────────────────────┘
                   │
                   ▼
┌────────────────────────────────────────────────────────────┐
│  Phase 4: Index Storage & Caching                          │
│                                                            │
│  1. Save Index to Disk                                     │
│     • .roo/code-index/vectors.json                         │
│     • .roo/code-index/metadata.json                        │
│     • .roo/code-index/cache.json                           │
│                                                            │
│  2. Create Cache Entries                                   │
│     • File hash → embedding mapping                        │
│     • Avoid re-indexing unchanged files                    │
│                                                            │
│  3. Setup File Watchers                                    │
│     • Watch for file changes                               │
│     • Incremental updates                                  │
│                                                            │
│  State: INDEXED                                            │
└────────────────────────────────────────────────────────────┘
```

### Search Execution Pipeline

```
┌────────────────────────────────────────────────────────────┐
│  Agent calls codebase_search tool                          │
│  {                                                         │
│    query: "authentication implementation",                │
│    max_results: 5                                         │
│  }                                                         │
└──────────────────┬─────────────────────────────────────────┘
                   │
                   ▼
┌────────────────────────────────────────────────────────────┐
│  Step 1: Check Index State                                 │
│  • Is index initialized?                                   │
│  • If not indexed → Return error or auto-index             │
└──────────────────┬─────────────────────────────────────────┘
                   │
                   ▼
┌────────────────────────────────────────────────────────────┐
│  Step 2: Query Embedding                                   │
│  • Generate embedding for query                            │
│  • embedding = embedProvider.embed("authentication...")    │
│  • Result: float[] (same dimension as index)               │
└──────────────────┬─────────────────────────────────────────┘
                   │
                   ▼
┌────────────────────────────────────────────────────────────┐
│  Step 3: Vector Search                                     │
│                                                            │
│  Similarity Search:                                        │
│  ┌──────────────────────────────────────┐                 │
│  │ 1. Compute Cosine Similarity         │                 │
│  │    For each indexed chunk:           │                 │
│  │      similarity = cosine(            │                 │
│  │        queryEmbedding,               │                 │
│  │        chunkEmbedding                │                 │
│  │      )                               │                 │
│  │                                      │                 │
│  │ 2. Rank by Similarity                │                 │
│  │    • Sort descending                 │                 │
│  │    • Filter by threshold (e.g., >0.7)│                 │
│  │                                      │                 │
│  │ 3. Take Top N Results                │                 │
│  │    • max_results from query          │                 │
│  │    • Return chunk metadata + score   │                 │
│  └──────────────────────────────────────┘                 │
└──────────────────┬─────────────────────────────────────────┘
                   │
                   ▼
┌────────────────────────────────────────────────────────────┐
│  Step 4: Keyword Fallback (if needed)                      │
│  • If vector search returns < threshold results            │
│  • Fall back to keyword search (BM25 or regex)             │
│  • Combine results                                         │
└──────────────────┬─────────────────────────────────────────┘
                   │
                   ▼
┌────────────────────────────────────────────────────────────┐
│  Step 5: Format Results                                    │
│                                                            │
│  For each result:                                          │
│  {                                                         │
│    filePath: "src/auth/login.ts",                         │
│    startLine: 25,                                         │
│    endLine: 45,                                           │
│    score: 0.89,                                           │
│    content: "...",                                        │
│    context: "function handleLogin(...) { ... }"           │
│  }                                                         │
│                                                            │
│  Return formatted results to agent                         │
└────────────────────────────────────────────────────────────┘
```

### Incremental Update Flow

```
File Changed (via Watcher)
    │
    ▼
┌─────────────────────────────────────┐
│  Detect Change Type                 │
│  • File modified                    │
│  • File added                       │
│  • File deleted                     │
└──────────────┬──────────────────────┘
               │
               ▼
┌─────────────────────────────────────┐
│  Batch Updates (Debounced)          │
│  • Wait 500ms for more changes      │
│  • Accumulate multiple files        │
└──────────────┬──────────────────────┘
               │
               ▼
┌─────────────────────────────────────┐
│  Process Changed Files              │
│  • Re-chunk modified files          │
│  • Generate new embeddings          │
│  • Update index                     │
│  • Update cache                     │
└──────────────┬──────────────────────┘
               │
               ▼
┌─────────────────────────────────────┐
│  Save Updated Index                 │
│  • Persist to disk                  │
│  • Ready for next search            │
└─────────────────────────────────────┘
```

### State Machine

```
State Transitions:

STANDBY
  │
  ├─> [initialize()] → INITIALIZING
  │                       │
  │                       └─> [validation success] → INDEXING
  │                                                     │
  └─> [disable feature]                                │
                                                        ▼
                                                    INDEXED
                                                        │
                                                        ├─> [file changed] → UPDATING
                                                        │                       │
                                                        │                       └─> INDEXED
                                                        │
                                                        └─> [error] → ERROR
                                                                        │
                                                                        └─> [retry] → INDEXING
```

### Ключевые Компоненты

**Initialization:**
- `CodeIndexManager.initialize()` - запуск индексации
- `validateConfiguration()` - проверка настроек
- `setupEmbeddingProvider()` - настройка embedding provider

**Indexing:**
- `scanWorkspace()` - сканирование файлов
- `processFile(path)` - обработка файла
- `chunkCode(content)` - разбиение на chunks
- `generateEmbeddings(chunks)` - генерация embeddings
- `saveIndex()` - сохранение индекса

**Searching:**
- `search(query, options)` - главная функция поиска
- `embedQuery(query)` - embedding запроса
- `vectorSearch(embedding)` - векторный поиск
- `rankResults(results)` - ранжирование
- `formatResults(results)` - форматирование

**Incremental Updates:**
- `watchFiles()` - мониторинг изменений
- `handleFileChange(path)` - обработка изменения
- `updateIndex(file)` - обновление индекса

---

## Заключение

Документация покрывает 6 основных пайплайнов Roo Code агента:

1. **System Prompt Generation** - 12-секционная сборка промтов
2. **Task Execution** - главный request/response loop
3. **Tool Execution** - парсинг, валидация и выполнение инструментов
4. **Mode Selection & Switching** - управление режимами
5. **MCP Integration** - подключение внешних серверов
6. **Codebase Search** - семантический поиск по коду

### Ключевые Паттерны

**Synchronization:**
- Mutexes (presentAssistantMessageLocked)
- Flags (userMessageContentReady)
- Promises (taskModeReady)
- Reference counting (MCP servers)

**Data Flow:**
- Queue-based stack (recursivelyMakeClineRequests)
- Accumulation patterns (userMessageContent)
- Streaming (API responses)

**Protocol Support:**
- Dual protocol (Native + XML)
- Type safety (Zod validation)
- Backward compatibility

**Lifecycle Management:**
- State machines (CodeIndexManager)
- Reference counting (McpHub)
- Debouncing (config updates, file watching)

### Integration Points

```
High-Level Flow:

User Input
    ↓
Task Initialization
    ↓
Mode Selection ──→ System Prompt Generation
    ↓                      ↓
Task Loop ←───────────────┘
    ↓
API Request
    ↓
Response Streaming
    ↓
Tool Parsing & Validation
    ↓
    ├─→ File Operations
    ├─→ Codebase Search
    ├─→ MCP Tool Calls
    └─→ User Interaction
         ↓
Tool Results Accumulation
    ↓
Next API Request (recursive)
    ↓
attempt_completion
    ↓
Task Complete
```

Все пайплайны работают вместе, формируя единую систему обработки задач с гибкой конфигурацией, мощными возможностями поиска и интеграцией с внешними сервисами.

---
