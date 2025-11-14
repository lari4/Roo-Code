# Roo-Code Agent Pipelines and Workflows Analysis

This document provides a detailed breakdown of the major agent pipelines in the Roo-Code codebase.

---

## 1. SYSTEM PROMPT GENERATION PIPELINE

### Entry Point
- **File**: `/home/user/Roo-Code/src/core/prompts/system.ts`
- **Main Function**: `SYSTEM_PROMPT()` (lines 148-241)

### Order of Prompt Sections (in `generatePrompt()`)

The system prompt is assembled in this specific order:

1. **Role Definition** (line 119)
   - Sets the agent's persona and role
   - Source: `getModeSelection()` from mode configuration

2. **Markdown Formatting Section** (line 121)
   - Defines how the agent should format markdown responses
   - Function: `markdownFormattingSection()`
   - File: `/home/user/Roo-Code/src/core/prompts/sections/markdown-formatting.ts`

3. **Shared Tool Use Section** (line 123)
   - XML protocol header or native protocol guidance
   - Function: `getSharedToolUseSection(effectiveProtocol)`
   - File: `/home/user/Roo-Code/src/core/prompts/sections/tool-use.ts`

4. **Tools Catalog** (lines 101-117)
   - Only for XML protocol (not native protocol)
   - Function: `getToolDescriptionsForMode()`
   - File: `/home/user/Roo-Code/src/core/prompts/tools/index.ts`
   - Includes all available tools for the current mode with their descriptions

5. **Tool Use Guidelines** (line 125)
   - Instructions for proper tool usage
   - Function: `getToolUseGuidelinesSection()`
   - File: `/home/user/Roo-Code/src/core/prompts/sections/tool-use-guidelines.ts`

6. **MCP Servers Section** (line 127)
   - Available MCP tools (if MCP is enabled and servers are connected)
   - Function: `getMcpServersSection()`
   - File: `/home/user/Roo-Code/src/core/prompts/sections/mcp-servers.ts`
   - Conditional: Only included if MCP group exists AND servers are available

7. **Capabilities Section** (line 129)
   - What the agent can do (file editing, bash, browser, etc.)
   - Function: `getCapabilitiesSection()`
   - File: `/home/user/Roo-Code/src/core/prompts/sections/capabilities.ts`

8. **Modes Section** (line 131)
   - Available modes and how to switch between them
   - Function: `getModesSection()`
   - File: `/home/user/Roo-Code/src/core/prompts/sections/modes.ts`

9. **Rules Section** (line 133)
   - Behavioral constraints and rules
   - Function: `getRulesSection()`
   - File: `/home/user/Roo-Code/src/core/prompts/sections/rules.ts`

10. **System Info Section** (line 135)
    - Current workspace, OS, shell info
    - Function: `getSystemInfoSection()`
    - File: `/home/user/Roo-Code/src/core/prompts/sections/system-info.ts`

11. **Objective Section** (line 137)
    - Code indexing objectives and capabilities
    - Function: `getObjectiveSection()`
    - File: `/home/user/Roo-Code/src/core/prompts/sections/objective.ts`

12. **Custom Instructions** (lines 139-143)
    - Mode-specific, file-based, or global custom instructions
    - Function: `addCustomInstructions()`
    - File: `/home/user/Roo-Code/src/core/prompts/sections/custom-instructions.ts`
    - Includes rooIgnore instructions and settings

### Data Transformations Between Steps

1. **Mode Resolution**
   - Takes mode slug → resolves to full ModeConfig (built-in or custom)
   - Custom modes override built-in modes
   - Default fallback: `defaultModeSlug`

2. **Protocol Determination**
   - Settings `toolProtocol` → determines if XML or Native protocol
   - Affects tool descriptions generation

3. **Context Preparation**
   - Extension context → CodeIndexManager instance
   - Workspace path → Code index state/orchestrator
   - MCP Hub → Available servers and tools

### Key Files and Functions

| Component | File | Function |
|-----------|------|----------|
| Main entry | `system.ts` | `SYSTEM_PROMPT()` |
| Generation | `system.ts` | `generatePrompt()` |
| File-based prompts | `custom-system-prompt.ts` | `loadSystemPromptFile()` |
| Mode selection | `modes.ts` | `getModeSelection()` |
| Custom instructions | `custom-instructions.ts` | `addCustomInstructions()` |

### Conditional Logic / Branching

1. **File-based Custom System Prompt Check** (lines 190-214)
   - If `roo-system-prompt.md` exists for the mode, use it instead of generated prompt
   - Falls back to generated prompt if file doesn't exist or is invalid

2. **MCP Inclusion Decision** (lines 78-81)
   - Check if mode has "mcp" group
   - Check if MCP servers are available
   - Only include if both conditions are true

3. **Protocol-based Tool Catalog** (lines 101-102)
   - XML protocol: Include full tool descriptions
   - Native protocol: Skip tool descriptions (already in tool definitions)

4. **Diff Strategy Application** (line 72)
   - If diff is disabled: `diffStrategy = undefined`
   - Otherwise: Pass actual strategy to capability sections

### Prompt Regeneration Triggers

The prompt needs to be regenerated when:
1. Mode is switched (new role/tools)
2. Custom modes are updated
3. Custom instructions change (file or global)
4. MCP servers connect/disconnect
5. Settings change (diff, protocol, experiments)
6. Code index state changes (for objectives section)
7. Workspace changes

The prompt is regenerated in `Task.getSystemPrompt()` at the beginning of each API request.

---

## 2. TASK EXECUTION PIPELINE

### Entry Points

1. **New Task**: `Task.startTask()` (line 1252)
   - Receives: task message, images
   - Initializes conversation history

2. **Resumed Task**: `Task.resumeTaskFromHistory()` (line 1293)
   - Loads saved conversation history
   - Asks user if they want to resume or start fresh

3. **Subtask**: `Task.startSubtask()` (line 1662)
   - Creates child task with specific message and mode

### Main Task Loop

**File**: `/home/user/Roo-Code/src/core/task/Task.ts`

#### Phase 1: Loop Initialization
```
initiateTaskLoop(userContent) → while (!abort)
  ↓
recursivelyMakeClineRequests(userContent, includeFileDetails)
```

**Function**: `initiateTaskLoop()` (lines 1727-1760)

Steps:
1. Initialize checkpoints service in background (line 1729)
2. Emit `TaskStarted` event (line 1734)
3. Enter while loop that processes API requests until:
   - User aborts
   - Task completion is attempted
   - Max requests reached

#### Phase 2: Message Processing Loop
**Function**: `recursivelyMakeClineRequests()` (lines 1762-2600)

Key Steps:
1. **Subtask Pause Check** (lines 1810-1827)
   - If paused waiting for subtask, wait and resume
   - Restore mode if changed during subtask

2. **User Content Mentions Processing** (lines 1852-1860)
   - Processes @mentions in user messages
   - Handles file content expansion
   - Processes diagnostic messages

3. **System Prompt Generation** (line 2622)
   - Calls `getSystemPrompt()` before each request
   - Includes current mode, settings, custom instructions

4. **API Request** (lines 1983-2576)
   - Calls `attemptApiRequest()` which yields stream chunks
   - Processes chunks in real-time

5. **Context Management** (per request)
   - Context condensing on demand
   - Token counting and budget management
   - Error handling for context window exceeded

#### Phase 3: Message Streaming
**Function**: `attemptApiRequest()` (lines 2784-3053)

Processes stream chunks:
- `"reasoning"` chunks → wisdom blocks
- `"text"` chunks → parsed into content blocks
- `"tool_call"` chunks → native protocol tool calls
- `"usage"` chunks → token tracking
- `"grounding"` chunks → source citations

#### Phase 4: Tool Execution & Feedback
**Function**: `presentAssistantMessage()` (lines 61-763 in presentAssistantMessage.ts)

For each content block from assistant:
1. **Text blocks**: Display to user
2. **Tool use blocks**:
   - Validate tool is allowed for mode
   - Check tool repetition
   - Get user approval (if required)
   - Execute tool via appropriate handler
   - Collect result
   - Add to user message content

#### Phase 5: Loop Control
- If no tools used: Ask agent to consider completion or use tools
- If tools used: Loop back with tool results
- If attempt_completion: Exit loop and complete task

### Message Flow

```
User Message
    ↓
[Extensions: @mentions, file content, diagnostics]
    ↓
System Prompt + User Content → API
    ↓
[Stream processing]
    ↓
Content Blocks (text, tool_calls)
    ↓
Present to User
    ↓
[Tool Execution]
    ↓
Tool Results → User Message Content
    ↓
[Add to conversation history]
    ↓
Next iteration or completion
```

### Key Data Structures

1. **ApiConversationHistory**: `Anthropic.MessageParam[]`
   - Messages sent to API
   - Persisted between sessions

2. **ClineMessages**: `ClineMessage[]`
   - UI representation of conversation
   - Saved to disk for restoration

3. **AssistantMessageContent**: Content blocks from streaming
   - Parsed in real-time
   - Executed sequentially

4. **UserMessageContent**: `Anthropic.Messages.ContentBlockParam[]`
   - Accumulates during tool execution
   - Sent in next request

---

## 3. TOOL EXECUTION PIPELINE

### Entry Point
**File**: `/home/user/Roo-Code/src/core/assistant-message/presentAssistantMessage.ts`
**Function**: `presentAssistantMessage()` (line 61)

### Tool Parsing

#### Native Protocol Tool Call Parsing
**File**: `/home/user/Roo-Code/src/core/assistant-message/NativeToolCallParser.ts`
**Class**: `NativeToolCallParser`

Process:
1. **Raw Tool Call** (from stream): `{ id, name, arguments }`
2. **Validation** (lines 26-41)
   - Check tool name is valid
   - Handle dynamic MCP tools (prefix: `mcp_`)
3. **Parameter Parsing** (lines 43-69)
   - Parse JSON arguments string
   - Build legacy params for backward compatibility
4. **Native Args Construction** (lines 71-230+)
   - For each tool: validate required params
   - Build properly typed `nativeArgs` object
   - Example tools: `read_file`, `execute_command`, `browser_action`
5. **Result**: `ToolUse<TName>` object with:
   - `params`: Legacy string parameters
   - `nativeArgs`: Properly typed parameters (if available)
   - `id`: Tool call ID
   - `name`: Tool name
   - `partial`: Whether this is incomplete

### Tool Validation

**File**: `/home/user/Roo-Code/src/core/tools/validateToolUse.ts`
**Function**: `validateToolUse()` (lines 5-15)

Checks:
1. Tool is allowed in current mode
2. Tool requirements are met (based on capabilities)
3. Tool parameters are valid for tool+mode combo

**Mode-based Validation**: `/home/user/Roo-Code/src/shared/modes.ts`
**Function**: `isToolAllowedForMode()` (lines 167-250+)

### Tool Execution Flow

For each tool in `switch(block.name)` (lines 484-705):

1. **Tool Handler Instantiation**
   - Each tool has a singleton instance (e.g., `writeToFileTool`)
   - Tool implements `BaseTool<ToolName>` interface

2. **Execution Pattern** (via `handle()` method):
   ```typescript
   toolHandler.handle(task, block, {
       askApproval,      // Get user consent
       handleError,      // Log and report errors
       pushToolResult,   // Add result to user message
       removeClosingTag  // Clean partial XML tags
   })
   ```

3. **Tool Base Class** (`BaseTool<TName>`)
   **File**: `/home/user/Roo-Code/src/core/tools/BaseTool.ts`
   
   Methods:
   - `handle()` (lines 133-169): Main entry point
     - Handles partial content
     - Routes to parser or native args
     - Calls execute()
   - `parseLegacy()`: Convert XML string params to typed params
   - `execute()`: Protocol-agnostic core logic
   - `handlePartial()`: Optional streaming UI updates

4. **Parameter Handling**:
   - **Native protocol**: Use `block.nativeArgs` directly (properly typed)
   - **XML protocol**: Parse `block.params` via `parseLegacy()`

### Tool Result Handling

Result Format:
- **Native protocol** (lines 290-314): Add `tool_result` block with `tool_use_id`
- **XML protocol** (lines 316-326): Add text blocks with tool description

Results are accumulated in `task.userMessageContent` and sent back to API in next request.

### Tools Registry

**File**: `/home/user/Roo-Code/src/core/assistant-message/presentAssistantMessage.ts`

All tools are instantiated as singletons:
- `writeToFileTool` → `WriteToFileTool`
- `readFileTool` → `ReadFileTool`
- `applyDiffToolClass` → `ApplyDiffTool` (native protocol)
- `executeCommandTool` → `ExecuteCommandTool`
- `browserActionTool` → `BrowserActionTool`
- `useMcpToolTool` → `UseMcpToolTool`
- And 15+ more...

Each tool is in `/home/user/Roo-Code/src/core/tools/{ToolName}.ts`

### Special Cases

#### Multi-File Apply Diff
- Check experiment flag `MULTI_FILE_APPLY_DIFF` (lines 521-529)
- If enabled: Use legacy `applyDiffTool` function
- Otherwise: Use `ApplyDiffTool` class

#### Simple vs Full Read File
- Check model capabilities via `shouldUseSingleFileRead()` (line 555)
- If true: Use `simpleReadFileTool` (single file)
- Otherwise: Use `readFileTool` (batch read)

#### Tool Repetition Detection
**File**: `/home/user/Roo-Code/src/core/tools/ToolRepetitionDetector.ts`

- Check for identical consecutive tool calls (lines 444-481)
- Limit repetitions per tool
- Ask user if limit reached

### Error Handling

1. **Parameter Parsing Error** (lines 159-165 in BaseTool)
   - Catch parse exception
   - Call `handleError()`
   - Push error result
   - Continue to next block

2. **Tool Validation Error** (lines 437-440 in presentAssistantMessage)
   - Increment `consecutiveMistakeCount`
   - Push tool error result

3. **Tool Execution Error** (via `handleError` callback)
   - Log error
   - Say error to user
   - Push formatted error result

---

## 4. MODE SELECTION AND SWITCHING PIPELINE

### Mode Selection

**File**: `/home/user/Roo-Code/src/shared/modes.ts`

#### Phase 1: Mode Resolution
**Function**: `getModeBySlug()` (lines 70-78)

Logic:
1. Check custom modes first (override built-in)
2. Check built-in modes
3. Return undefined if not found

#### Phase 2: Mode Configuration
**Function**: `getModeConfig()` (lines 80-86)

Returns full `ModeConfig` with:
- `slug`: Unique identifier
- `name`: Display name
- `description`: Purpose
- `groups`: Tool groups included in mode
- `roleDefinition`: Agent persona
- `customInstructions`: Mode-specific instructions

#### Phase 3: Mode Selection with Prompts
**Function**: `getModeSelection()` (lines 130-151)

Returns:
- `roleDefinition`: From custom mode or prompt component or built-in
- `baseInstructions`: From custom mode or prompt component or built-in
- `description`: From mode

Priority order:
1. Custom mode (if exists)
2. Prompt component (if exists)
3. Built-in mode (fallback)

### Mode Switching Pipeline

**Entry Point**: `SwitchModeTool.execute()`
**File**: `/home/user/Roo-Code/src/core/tools/SwitchModeTool.ts`

Steps:
1. **Validate Parameters** (lines 29-34)
   - Check mode_slug is provided

2. **Verify Mode Exists** (lines 38-45)
   - Get target mode via `getModeBySlug()`
   - Return error if not found

3. **Check Current Mode** (lines 48-54)
   - Get current mode from provider state
   - Return error if already in target mode

4. **Get User Approval** (lines 56-61)
   - Ask user to confirm mode switch

5. **Execute Switch** (line 64)
   - Call `provider.handleModeSwitch(mode_slug)`
   - This updates provider state and regenerates system prompt

6. **Delay for Effect** (line 72)
   - 500ms delay to allow UI update

### Tool Availability by Mode

**File**: `/home/user/Roo-Code/src/core/prompts/tools/index.ts`
**Function**: `getToolDescriptionsForMode()` (lines 63-118)

Process:
1. Get mode config
2. Iterate over mode's `groups`
3. For each group: get `TOOL_GROUPS[groupName].tools`
4. Filter by tool validation (`isToolAllowedForMode`)
5. Generate descriptions for each tool

**Key Data**: `/home/user/Roo-Code/src/shared/tools.ts`
- `TOOL_GROUPS`: Maps group name → tools
- `ALWAYS_AVAILABLE_TOOLS`: Tools available in all modes

### Mode Validation

**Function**: `isToolAllowedForMode()` (lines 167-250+ in modes.ts)

Checks:
1. Is tool in `ALWAYS_AVAILABLE_TOOLS`? → Allow
2. Is tool disabled by experiment? → Deny
3. Does mode require tool capability? → Check requirements
4. Is tool in any of mode's groups? → Allow
5. Does tool+mode combo have file restrictions? → Validate files

---

## 5. MCP INTEGRATION PIPELINE

### MCP Hub Initialization

**File**: `/home/user/Roo-Code/src/services/mcp/McpHub.ts`

#### Constructor Phase (lines 158-165)

Steps:
1. Store provider weak reference
2. Watch global MCP settings file
3. Watch project MCP settings file
4. Setup workspace folders watcher
5. Initialize global MCP servers
6. Initialize project MCP servers

#### Configuration Sources (Priority Order)

1. **Global Settings** (watched)
   - User's global Roo-Code settings
   - File: `.config/roo-code/mcp-settings.json` (typical)

2. **Project Settings** (watched)
   - Per-project: `.roo/mcp-settings.json`

3. **Environment Variables**
   - Variable substitution via `injectVariables()`

### Server Configuration Validation

**Schema Definition** (lines 60-137):

Configuration types:
- **Stdio**: Command-based servers
  - Required: `command`
  - Optional: `args`, `env`, `cwd`
  - Type inference: Auto-set to "stdio"

- **SSE**: Server-Sent Events
  - Required: `url`, `type: "sse"`
  - Optional: `headers`

- **Streamable HTTP**: HTTP with streaming
  - Required: `url`, `type: "streamable-http"`
  - Optional: `headers`

Common fields (all types):
- `disabled`: Boolean
- `timeout`: 1-3600 seconds (default: 60)
- `alwaysAllow`: Array of tool names (auto-approve)
- `watchPaths`: Paths to watch for restart
- `disabledTools`: Tools to disable on this server

### Server Connection Lifecycle

#### Phase 1: Config Loading
1. Watch file for changes
2. Debounce config changes (500ms)
3. Parse JSON
4. Validate schema
5. Handle errors with user messages

#### Phase 2: Server Connection
**Function**: `updateServerConnections()` (called when config changes)

For each server:
1. Create appropriate transport:
   - Stdio: `StdioClientTransport`
   - SSE: `SSEClientTransport`
   - HTTP: `StreamableHTTPClientTransport`

2. Initialize MCP Client
3. List tools via `tools/list`
4. List resources via `resources/list`
5. Store in `connections` array

#### Phase 3: Tool Discovery
After connection:
1. Fetch tool list from server
2. Parse tool definitions
3. Create dynamic tool names: `mcp_{serverName}_{toolName}`
4. Include in system prompt

**Function**: `getMcpServersSection()`
**File**: `/home/user/Roo-Code/src/core/prompts/sections/mcp-servers.ts`

Generates:
- Server descriptions
- Available tools per server
- Resource templates (if any)
- Usage instructions

### MCP Tool Execution

**Entry Point**: `UseMcpToolTool.execute()`
**File**: `/home/user/Roo-Code/src/core/tools/UseMcpToolTool.ts`

Steps:
1. **Parse Parameters**
   - `server_name`: MCP server name
   - `tool_name`: Tool name on that server
   - `arguments`: Tool arguments (JSON)

2. **Validation** (lines 99-180+)
   - Check server exists and is connected
   - Check tool exists on server
   - Validate argument schema

3. **User Approval** (line 67)
   - Ask user to approve tool call
   - Show server name, tool name, arguments

4. **Tool Execution** (line 74-81)
   - Call `executeToolAndProcessResult()`
   - Make MCP `tools/call` request

5. **Result Processing**
   - Format result
   - Handle streaming if applicable
   - Push result to task

### MCP Resource Access

**Entry Point**: `accessMcpResourceTool()`
**File**: `/home/user/Roo-Code/src/core/tools/accessMcpResourceTool.ts`

Steps:
1. **Parse Parameters**
   - `server_name`: Server
   - `resource_uri`: Resource identifier
   - Optional: template name

2. **Resource Reading**
   - Fetch resource content via MCP
   - Handle different content types
   - Support binary data (base64)

3. **Template Rendering** (if applicable)
   - Render resource with template

4. **Result to Agent**
   - Return formatted resource content

### Reference Counting

**Reference Counter** (lines 153, 170-187):

Purpose: Manage lifecycle of hub

- Increment on client register (line 171)
- Decrement on client unregister (line 180)
- Auto-dispose when count reaches 0 (line 186)

This prevents premature disconnection when multiple clients use the hub.

### Configuration Update Handling

**Debounce Pattern** (lines 283-304):

- Delay: 500ms between config change and processing
- Purpose: Avoid multiple restarts during rapid edits
- Programmatic flag: Skip if update is from code (prevent loops)

---

## 6. CODEBASE SEARCH PIPELINE

### Initialization

**Manager**: `/home/user/Roo-Code/src/services/code-index/manager.ts`
**Class**: `CodeIndexManager` (singleton per workspace)

#### Instance Management (lines 20-57)

- Static map: `workspace path → CodeIndexManager instance`
- Factory: `getInstance(context, workspacePath)`
- Cleanup: `disposeAll()`

#### Lazy Initialization Pattern

1. **Config Manager**: Loads embedding provider config
2. **Service Factory**: Creates embedder service
3. **Orchestrator**: Manages indexing workflow
4. **Vector Store**: Stores embeddings
5. **Search Service**: Handles search queries

### Orchestration

**File**: `/home/user/Roo-Code/src/services/code-index/orchestrator.ts`
**Class**: `CodeIndexOrchestrator`

#### Indexing Workflow

**Phase 1: Initial Scan**
`startIndexing()` (lines 97-200+):

1. Check workspace available
2. Initialize vector store
3. Scan directory structure
4. Process each file:
   - Read file content
   - Generate embeddings
   - Store in vector store
5. Mark as indexed

**Phase 2: File Watching**
`_startWatcher()` (lines 32-88):

1. Initialize file watcher
2. Monitor for changes:
   - New files
   - Modified files
   - Deleted files

3. Batch processing on changes:
   - Accumulate changes
   - Process in batches
   - Update index
   - Emit progress

**Phase 3: State Management**

States:
- `"Standby"`: Feature disabled or not configured
- `"Initializing"`: Starting up
- `"Indexing"`: Initial scan in progress
- `"Indexed"`: Index up-to-date
- `"Error"`: Error occurred

State transitions trigger UI updates via `stateManager.onProgressUpdate`.

### Search Execution

**Entry Point**: `CodebaseSearchTool.execute()`
**File**: `/home/user/Roo-Code/src/core/tools/CodebaseSearchTool.ts`

Steps:
1. **Parse Query**
   - Extract search term
   - Optional file path filter
   - Optional language filter

2. **Get Search Service**
   - From CodeIndexManager
   - Fallback to ripgrep if indexing disabled

3. **Execute Search**
   - Send query to embedder
   - Get query embedding
   - Search vector store (semantic)
   - Fallback to keyword search

4. **Result Formatting**
   - Group by file
   - Include snippets
   - Rank by relevance
   - Return as text blocks

### Search Service

**File**: `/home/user/Roo-Code/src/services/code-index/search-service.ts`

Method: `search(query: string, options)`: Promise<SearchResult[]>

Returns:
- File path
- Line number
- Snippet (context)
- Relevance score
- Match type (semantic/keyword)

### Caching

**Manager**: `/home/user/Roo-Code/src/services/code-index/cache-manager.ts`

Purpose: Avoid re-indexing unchanged files

Components:
- File hash tracking
- Embedding cache
- Invalidation on changes
- TTL-based cleanup

### Embedding Providers

**Directory**: `/home/user/Roo-Code/src/services/code-index/embedders`

Supported:
- Local models (via ollama, etc.)
- Cloud APIs (OpenAI, Anthropic)
- Custom embedders

Configuration:
- Model selection
- API key management
- Batch size tuning
- Timeout handling

### Integration with Prompts

**Tool Description**: `getCodebaseSearchDescription()`
**File**: `/home/user/Roo-Code/src/core/prompts/tools/codebase-search.ts`

Includes in prompt:
- Search capabilities
- Example queries
- Result format
- Limitations

---

## SUMMARY TABLE

| Pipeline | Entry Point | Main Loop | Key Data | Output |
|----------|-------------|-----------|----------|--------|
| System Prompt | `SYSTEM_PROMPT()` | Section concatenation | PromptComponent, Mode | String prompt |
| Task Execution | `initiateTaskLoop()` | `recursivelyMakeClineRequests()` | apiConversationHistory | Task completion |
| Tool Execution | `presentAssistantMessage()` | Per-content-block switch | AssistantMessageContent | ToolResponse |
| Mode Switching | `SwitchModeTool.execute()` | Validation → Approval → Switch | ModeConfig | Mode change event |
| MCP Integration | `McpHub.constructor()` | File watch → Config parse → Connect | McpConnection[] | Dynamic tools |
| Code Search | `CodebaseSearchTool.execute()` | Embed → Vector search → Format | VectorStoreSearchResult[] | Search results |

---

## CRITICAL SYNCHRONIZATION POINTS

1. **System Prompt Generation**: Before each API request in `recursivelyMakeClineRequests()`
2. **User Message Content Ready**: After all tools executed, triggers next API request
3. **Streaming Lock**: `presentAssistantMessageLocked` prevents concurrent content processing
4. **Mode Initialization**: `taskModeReady` promise ensures mode is available before task use
5. **MCP Client Refcount**: Prevents disposal while clients are active

