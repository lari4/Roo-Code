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
