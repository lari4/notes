# Cline Agent Pipelines Documentation

> Полная документация схем работы AI-агента Cline: пайплайны, последовательность вызова промптов и передача данных

**Дата создания:** 2025-11-13
**Источник:** [github.com/cline/cline](https://github.com/cline/cline)
**Версия:** main branch

---

## Оглавление

1. [Введение](#введение)
2. [Общая архитектура](#общая-архитектура)
3. [Pipeline 1: System Prompt Assembly](#pipeline-1-system-prompt-assembly)
4. [Pipeline 2: Basic Task Execution](#pipeline-2-basic-task-execution)
5. [Pipeline 3: Plan Mode → Act Mode](#pipeline-3-plan-mode--act-mode)
6. [Pipeline 4: File Editing Workflow](#pipeline-4-file-editing-workflow)
7. [Pipeline 5: Context Management & Continuation](#pipeline-5-context-management--continuation)
8. [Pipeline 6: CLI Subagent Invocation](#pipeline-6-cli-subagent-invocation)
9. [Pipeline 7: MCP Tool Usage](#pipeline-7-mcp-tool-usage)
10. [Pipeline 8: New Task Creation](#pipeline-8-new-task-creation)
11. [Data Flow Patterns](#data-flow-patterns)

---

## Введение

Cline использует сложную систему пайплайнов для обработки пользовательских запросов. Каждый пайплайн представляет собой последовательность шагов, где результат одного шага передается в следующий.

### Ключевые компоненты

1. **PromptRegistry** - Управление вариантами промптов
2. **PromptBuilder** - Сборка системного промпта
3. **ClineToolSet** - Управление доступными инструментами
4. **TemplateEngine** - Рендеринг шаблонов с плейсхолдерами
5. **ContextManager** - Управление контекстом беседы

---

## Общая архитектура

```
┌─────────────────────────────────────────────────────────────────┐
│                        User Request                              │
└────────────────────────────┬────────────────────────────────────┘
                             │
                             ▼
┌─────────────────────────────────────────────────────────────────┐
│                    Context Analysis                              │
│  • Provider detection (OpenAI, Anthropic, etc.)                 │
│  • Model family identification                                   │
│  • Available capabilities check                                  │
│  • Mode determination (Plan/Act/YOLO)                           │
└────────────────────────────┬────────────────────────────────────┘
                             │
                             ▼
┌─────────────────────────────────────────────────────────────────┐
│                    Variant Selection                             │
│  PromptRegistry.get(context)                                    │
│  • Match model family → variant                                 │
│  • Fallback to GENERIC if no match                             │
└────────────────────────────┬────────────────────────────────────┘
                             │
                             ▼
┌─────────────────────────────────────────────────────────────────┐
│                 System Prompt Assembly                           │
│  PromptBuilder.build(variant, context)                          │
│  1. Execute component functions                                 │
│  2. Prepare placeholders                                        │
│  3. Resolve template with TemplateEngine                        │
│  4. Post-process (normalize whitespace)                         │
└────────────────────────────┬────────────────────────────────────┘
                             │
                             ▼
┌─────────────────────────────────────────────────────────────────┐
│                    Tool Selection                                │
│  ClineToolSet.getEnabledTools(variant, context)                 │
│  • Filter by context requirements                               │
│  • Convert to native format for provider                        │
└────────────────────────────┬────────────────────────────────────┘
                             │
                             ▼
┌─────────────────────────────────────────────────────────────────┐
│                      AI Model Call                               │
│  { systemPrompt, tools, messages }                              │
└────────────────────────────┬────────────────────────────────────┘
                             │
                             ▼
┌─────────────────────────────────────────────────────────────────┐
│                    Response Processing                           │
│  • Tool call extraction                                         │
│  • Error handling                                               │
│  • Response formatting                                          │
└────────────────────────────┬────────────────────────────────────┘
                             │
                             ▼
┌─────────────────────────────────────────────────────────────────┐
│                    User Output                                   │
└─────────────────────────────────────────────────────────────────┘
```

---

## Pipeline 1: System Prompt Assembly

Этот пайплайн отвечает за сборку системного промпта из компонентов.

```
┌──────────────────────────────────────────────────────────────────┐
│  Input: SystemPromptContext                                      │
│  {                                                                │
│    providerInfo: { provider, modelId, family },                 │
│    ide: "vscode",                                               │
│    supportsBrowser: boolean,                                    │
│    mcpHub: McpHubInstance,                                      │
│    ruleFileInstructions: {...},                                 │
│    focusChainSettings: {...}                                    │
│  }                                                               │
└────────────────────────┬─────────────────────────────────────────┘
                         │
                         ▼
         ┌──────────────────────────────────┐
         │  PromptRegistry.get(context)    │
         │  • Match model → variant        │
         │  • Fallback: GENERIC            │
         └────────────┬─────────────────────┘
                      │
                      ▼
         ┌──────────────────────────────────┐
         │  PromptBuilder.build()          │
         │                                  │
         │  Step 1: Component Building     │
         │  ========================        │
         │  For each component in order:   │
         │    - Execute component fn       │
         │    - Capture result             │
         │    - Log warnings on failure    │
         └────────────┬─────────────────────┘
                      │
                      ▼
         ┌──────────────────────────────────────┐
         │  Step 2: Placeholder Preparation    │
         │  ============================        │
         │  Merge (priority order):             │
         │  1. Variant placeholders (lowest)   │
         │  2. System placeholders             │
         │     • CWD, OS, SHELL, HOME_DIR      │
         │     • SUPPORTS_BROWSER              │
         │     • MODEL_FAMILY, CURRENT_DATE    │
         │  3. Component sections              │
         │     • AGENT_ROLE, CAPABILITIES      │
         │     • OBJECTIVE, RULES, etc.        │
         │  4. Runtime placeholders (highest)  │
         └────────────┬─────────────────────────┘
                      │
                      ▼
         ┌──────────────────────────────────┐
         │  Step 3: Template Resolution    │
         │  ========================        │
         │  TemplateEngine.resolve(         │
         │    baseTemplate,                 │
         │    placeholders                  │
         │  )                               │
         │                                  │
         │  Replace all {{PLACEHOLDER}}    │
         │  with actual values             │
         └────────────┬─────────────────────┘
                      │
                      ▼
         ┌──────────────────────────────────┐
         │  Step 4: Post-Processing        │
         │  ====================            │
         │  • Normalize whitespace         │
         │  • Remove empty lines           │
         │  • Clean malformed headers      │
         │  • Preserve diff markers        │
         └────────────┬─────────────────────┘
                      │
                      ▼
┌────────────────────────────────────────────┐
│  Output: Assembled System Prompt           │
│  {                                          │
│    prompt: string (complete system prompt),│
│    metadata: {                             │
│      variant: "generic",                   │
│      components: ["AGENT_ROLE", ...],      │
│      placeholders: ["CWD", "OS", ...]      │
│    }                                        │
│  }                                          │
└─────────────────────────────────────────────┘
```

**Данные между шагами:**

- **Input → Registry**: Контекст с информацией о провайдере и модели
- **Registry → Builder**: Выбранный вариант промпта (generic/next-gen/xs/etc.)
- **Builder Step 1 → 2**: Объект с результатами компонентов `{ AGENT_ROLE: "...", CAPABILITIES: "..." }`
- **Builder Step 2 → 3**: Объединенный объект плейсхолдеров
- **Builder Step 3 → 4**: Строка с разрешенными плейсхолдерами
- **Builder Step 4 → Output**: Финальный системный промпт

---

## Pipeline 2: Basic Task Execution

Базовый пайплайн выполнения задачи пользователя.

```
┌──────────────────────────────────────────┐
│  User Message: "Fix the bug in auth.ts" │
└────────────────┬─────────────────────────┘
                 │
                 ▼
     ┌──────────────────────────┐
     │  System Prompt Assembly │
     │  (Pipeline 1)            │
     └──────────┬───────────────┘
                │ ╔════════════════════════════╗
                │ ║ System Prompt              ║
                │ ║ • AGENT_ROLE               ║
                │ ║ • OBJECTIVE                ║
                │ ║ • CAPABILITIES             ║
                │ ║ • RULES                    ║
                │ ║ • Available tools          ║
                │ ╚════════════════════════════╝
                ▼
     ┌──────────────────────────┐
     │  Tool Selection          │
     │  ClineToolSet.get()      │
     │  • Filter by mode        │
     │  • Filter by context     │
     └──────────┬───────────────┘
                │ ╔════════════════════════════╗
                │ ║ Available Tools:           ║
                │ ║ • read_file                ║
                │ ║ • search_files             ║
                │ ║ • replace_in_file          ║
                │ ║ • execute_command          ║
                │ ║ • attempt_completion       ║
                │ ╚════════════════════════════╝
                ▼
     ┌──────────────────────────┐
     │  AI Model Call           │
     │  {                       │
     │    systemPrompt,         │
     │    tools,                │
     │    messages: [           │
     │      {user: "Fix bug"}   │
     │    ]                     │
     │  }                       │
     └──────────┬───────────────┘
                │
                ▼
     ┌──────────────────────────────────┐
     │  AI Response                     │
     │  [Thinking]: "Need to read file" │
     │  [Tool]: <read_file>             │
     │           <path>auth.ts</path>   │
     │          </read_file>             │
     └──────────┬───────────────────────┘
                │
                ▼
     ┌──────────────────────────┐
     │  Tool Execution          │
     │  • Parse XML             │
     │  • Extract parameters    │
     │  • Execute tool          │
     │  • Format response       │
     └──────────┬───────────────┘
                │ ╔════════════════════════════╗
                │ ║ Tool Result:               ║
                │ ║ [File content of auth.ts]  ║
                │ ╚════════════════════════════╝
                ▼
     ┌──────────────────────────┐
     │  Add to Conversation     │
     │  messages.push({         │
     │    assistant: tool_call, │
     │    tool: result          │
     │  })                      │
     └──────────┬───────────────┘
                │
                ▼
     ┌──────────────────────────┐
     │  Next AI Call            │
     │  • Same system prompt    │
     │  • Updated messages      │
     └──────────┬───────────────┘
                │
                ▼
     ┌──────────────────────────────────┐
     │  AI Response                     │
     │  [Thinking]: "Found the issue"   │
     │  [Tool]: <replace_in_file>       │
     │           [fix content]          │
     │          </replace_in_file>       │
     └──────────┬───────────────────────┘
                │
                ▼
     ┌──────────────────────────┐
     │  Tool Execution          │
     │  • Apply changes         │
     │  • Auto-format           │
     └──────────┬───────────────┘
                │
                ▼
     ┌──────────────────────────────────┐
     │  AI Response                     │
     │  [Tool]: <attempt_completion>    │
     │           <result>               │
     │             Fixed auth bug       │
     │           </result>               │
     │          </attempt_completion>    │
     └──────────┬───────────────────────┘
                │
                ▼
     ┌──────────────────────────┐
     │  Task Complete           │
     │  Present result to user  │
     └──────────────────────────┘
```

**Ключевые промпты:**

- **AGENT_ROLE**: Идентичность агента
- **OBJECTIVE**: Итеративный подход (Step 1-5)
- **CAPABILITIES**: Описание доступных инструментов
- **EDITING_FILES**: Выбор между write_to_file и replace_in_file
- **RULES**: Стиль ответов (прямой, без вопросов)

**Данные между итерациями:**

- Контекст беседы накапливается
- Каждый tool result добавляется в messages
- System prompt остается неизменным
- Tool availability может меняться

---

## Pipeline 3: Plan Mode → Act Mode

Пайплайн для режима планирования с переходом к выполнению.

```
┌────────────────────────────────────────────┐
│  User Message: "Build a REST API for blog"│
└────────────────┬───────────────────────────┘
                 │
                 ▼
     ┌──────────────────────────────┐
     │  Detect Mode: PLAN MODE     │
     │  • New task                 │
     │  • No YOLO mode             │
     └──────────┬──────────────────┘
                │
                ▼
     ┌──────────────────────────────────────┐
     │  System Prompt Assembly             │
     │  with ACT_VS_PLAN component:        │
     │                                      │
     │  "You have two operational modes:   │
     │   PLAN MODE: Provides access to     │
     │   plan_mode_respond tool..."        │
     └──────────┬──────────────────────────┘
                │
                ▼
     ┌──────────────────────────────────────┐
     │  Tool Selection: PLAN MODE          │
     │  Available tools:                   │
     │  • read_file                        │
     │  • search_files                     │
     │  • list_files                       │
     │  • plan_mode_respond ✓              │
     │                                      │
     │  NOT available:                     │
     │  • write_to_file                    │
     │  • replace_in_file                  │
     │  • execute_command                  │
     │  • attempt_completion               │
     └──────────┬──────────────────────────┘
                │
                ▼
     ┌──────────────────────────────────────┐
     │  AI explores context                │
     │  [Tool]: <list_files>               │
     │          <path>./src</path>         │
     │         </list_files>                │
     └──────────┬──────────────────────────┘
                │
                ▼
     ┌──────────────────────────────────────┐
     │  Tool Result:                       │
     │  [Directory structure returned]     │
     └──────────┬──────────────────────────┘
                │
                ▼
     ┌──────────────────────────────────────┐
     │  AI presents plan                   │
     │  [Tool]: <plan_mode_respond>        │
     │  <plan>                             │
     │  I'll build REST API with:          │
     │  1. Express.js server               │
     │  2. MongoDB for storage             │
     │  3. CRUD endpoints for posts        │
     │  4. Authentication middleware       │
     │  5. Input validation                │
     │                                      │
     │  Shall I proceed?                   │
     │  </plan>                            │
     │  </plan_mode_respond>                │
     └──────────┬──────────────────────────┘
                │
                ▼
     ┌──────────────────────────────────────┐
     │  User approves: "Yes, go ahead"     │
     └──────────┬──────────────────────────┘
                │
                ▼
     ┌──────────────────────────────────────┐
     │  Switch to ACT MODE                 │
     │  Re-assemble system prompt with:    │
     │  • Same base components             │
     │  • Different tool set               │
     └──────────┬──────────────────────────┘
                │
                ▼
     ┌──────────────────────────────────────┐
     │  Tool Selection: ACT MODE           │
     │  Available tools:                   │
     │  • ALL tools enabled                │
     │  • plan_mode_respond removed        │
     │  • attempt_completion added         │
     └──────────┬──────────────────────────┘
                │
                ▼
     ┌──────────────────────────────────────┐
     │  AI creates task progress           │
     │  <update_task_todo>                 │
     │  - [ ] Setup Express server         │
     │  - [ ] Configure MongoDB            │
     │  - [ ] Create Post model            │
     │  - [ ] Implement CRUD routes        │
     │  - [ ] Add authentication           │
     │  - [ ] Write tests                  │
     │  </update_task_todo>                 │
     └──────────┬──────────────────────────┘
                │
                ▼
     ┌──────────────────────────────────────┐
     │  AI executes plan                   │
     │  • Creates files                    │
     │  • Installs dependencies            │
     │  • Implements features              │
     │  • Updates checklist                │
     │  • Signals completion               │
     └──────────────────────────────────────┘
```

**Ключевые промпты:**

- **ACT_VS_PLAN_MODE**: Объяснение двух режимов
- **TASK_PROGRESS**: Инструкции по чеклисту

**Переходные данные:**

- **Plan → Act**: План из plan_mode_respond используется как контекст
- **Context preserved**: История исследования файлов сохраняется
- **Tool set change**: Полный набор инструментов в Act mode

---

## Pipeline 4: File Editing Workflow

Пайплайн редактирования файлов с учетом автоформатирования.

```
┌──────────────────────────────────────────┐
│  User: "Add error handling to api.js"   │
└────────────────┬─────────────────────────┘
                 │
                 ▼
     ┌──────────────────────────────┐
     │  EDITING_FILES prompt:      │
     │  "You have two tools:       │
     │   - write_to_file           │
     │   - replace_in_file         │
     │                             │
     │  DEFAULT to replace_in_file"│
     └──────────┬──────────────────┘
                │
                ▼
     ┌──────────────────────────────────┐
     │  AI: Read current file           │
     │  <read_file>                     │
     │    <path>api.js</path>           │
     │  </read_file>                     │
     └──────────┬──────────────────────┘
                │ ╔═══════════════════════╗
                │ ║ File Content V1:      ║
                │ ║ function fetchData() {║
                │ ║   fetch('/api')       ║
                │ ║ }                     ║
                │ ╚═══════════════════════╝
                ▼
     ┌──────────────────────────────────┐
     │  AI: Apply targeted edit         │
     │  <replace_in_file>               │
     │  <path>api.js</path>             │
     │  <diff>                          │
     │  <<<<<<< SEARCH                  │
     │  function fetchData() {          │
     │    fetch('/api')                 │
     │  }                               │
     │  =======                         │
     │  function fetchData() {          │
     │    try {                         │
     │      fetch('/api')               │
     │    } catch(e) {                  │
     │      console.error(e)            │
     │    }                             │
     │  }                               │
     │  >>>>>>> REPLACE                 │
     │  </diff>                         │
     │  </replace_in_file>               │
     └──────────┬──────────────────────┘
                │
                ▼
     ┌──────────────────────────────────┐
     │  File System: Apply edit         │
     │  • Match SEARCH block            │
     │  • Replace with REPLACE block    │
     │  • Trigger auto-format           │
     └──────────┬──────────────────────┘
                │
                ▼
     ┌──────────────────────────────────────┐
     │  Auto-formatter runs:                │
     │  • Adjusts indentation (2→4 spaces)  │
     │  • Converts quotes (' → ")           │
     │  • Adds semicolons                   │
     │  • Organizes imports                 │
     └──────────┬──────────────────────────┘
                │ ╔═══════════════════════════╗
                │ ║ File Content V2 (final):  ║
                │ ║ function fetchData() {    ║
                │ ║     try {                 ║
                │ ║         fetch("/api");    ║
                │ ║     } catch(e) {          ║
                │ ║         console.error(e); ║
                │ ║     }                     ║
                │ ║ }                         ║
                │ ╚═══════════════════════════╝
                ▼
     ┌──────────────────────────────────────┐
     │  Response to AI:                     │
     │  "File edited successfully.          │
     │   Note: Auto-formatting applied:     │
     │   - Indentation changed 2→4 spaces   │
     │   - Single quotes → double quotes    │
     │   - Semicolons added                 │
     │                                       │
     │  Use this final state as reference   │
     │  for subsequent edits."              │
     └──────────┬──────────────────────────┘
                │
                ▼
     ┌──────────────────────────────────────┐
     │  AI: Next edit (uses V2 format)     │
     │  <replace_in_file>                  │
     │  <diff>                             │
     │  <<<<<<< SEARCH                     │
     │  function fetchData() {             │
     │      try {                          │
     │          fetch("/api");             │
     │  =======                            │
     │  async function fetchData() {       │
     │      try {                          │
     │          const res = await fetch("/api");│
     │  >>>>>>> REPLACE                    │
     │  </diff>                            │
     │  </replace_in_file>                  │
     └──────────────────────────────────────┘
```

**Ключевые промпты:**

- **EDITING_FILES**: Выбор инструмента и workflow
  - "Default to replace_in_file"
  - "Use final formatted state as reference"
  - "Multiple edits in one call"

**Важные данные:**

- **V1 → V2**: Содержимое файла меняется из-за автоформатирования
- **Response includes diff**: AI информируется о точных изменениях
- **Context for next edit**: AI должен использовать V2, а не V1

---

## Pipeline 5: Context Management & Continuation

Пайплайн управления контекстом при приближении к лимиту.

```
┌──────────────────────────────────────────┐
│  Conversation reaches ~90% context limit│
└────────────────┬─────────────────────────┘
                 │
                 ▼
     ┌──────────────────────────────────────┐
     │  Trigger: Context Management        │
     │  • Token count high                 │
     │  • Need to continue work            │
     └──────────┬──────────────────────────┘
                │
                ▼
     ┌──────────────────────────────────────┐
     │  Call summarizeTask() prompt        │
     │                                      │
     │  "Create comprehensive summary:     │
     │   1. Primary Request and Intent     │
     │   2. Key Technical Concepts         │
     │   3. Files and Code Sections        │
     │   4. Problem Solving                │
     │   5. Pending Tasks                  │
     │   6. Task Evolution                 │
     │   7. Current Work                   │
     │   8. Next Step                      │
     │   9. Required Files"                │
     └──────────┬──────────────────────────┘
                │
                ▼
     ┌──────────────────────────────────────┐
     │  AI generates summary               │
     │  {                                   │
     │    context: "Working on auth...",   │
     │    technical_concepts: [            │
     │      "JWT tokens",                  │
     │      "bcrypt hashing"               │
     │    ],                               │
     │    relevant_files: [                │
     │      "src/auth/login.ts",           │
     │      "src/middleware/auth.ts"       │
     │    ],                               │
     │    problem_solving: "Fixed token...",│
     │    pending_tasks: [...],            │
     │    task_progress: "- [x] Setup..."  │
     │  }                                   │
     └──────────┬──────────────────────────┘
                │
                ▼
     ┌──────────────────────────────────────┐
     │  Store Summary                      │
     │  • Save to conversation history     │
     │  • Mark old messages as condensed   │
     └──────────┬──────────────────────────┘
                │
                ▼
     ┌──────────────────────────────────────┐
     │  Create New Task/Continue           │
     │  Option A: new_task tool            │
     │  Option B: Auto continuation        │
     └──────────┬──────────────────────────┘
                │
                ▼
     ┌──────────────────────────────────────┐
     │  New Conversation Context           │
     │  messages = [                       │
     │    {                                 │
     │      role: "system",                │
     │      content: continuationPrompt()   │
     │    },                               │
     │    {                                 │
     │      role: "user",                  │
     │      content: summary               │
     │    },                               │
     │    {                                 │
     │      role: "user",                  │
     │      content: "Continue work"       │
     │    }                                 │
     │  ]                                   │
     └──────────┬──────────────────────────┘
                │
                ▼
     ┌──────────────────────────────────────┐
     │  continuationPrompt():              │
     │  "Continue from where work          │
     │   was suspended.                    │
     │   - Don't ask clarifications        │
     │   - Focus on recent message         │
     │   - Ignore special commands"        │
     └──────────┬──────────────────────────┘
                │
                ▼
     ┌──────────────────────────────────────┐
     │  AI resumes work                    │
     │  • Has full context from summary    │
     │  • Knows exactly what to do next    │
     │  • Can access required files        │
     └──────────────────────────────────────┘
```

**Ключевые промпты:**

- **summarizeTask**: 9-section summary format
- **continuationPrompt**: Instructions for resuming
- **contextTruncationNotice**: Notification to user

**Данные сохраняются:**

- Technical concepts
- File paths
- Code patterns
- Architectural decisions
- Current progress (checklist)
- Next step

**Данные теряются:**

- Точные tool calls
- Промежуточные рассуждения
- Некоторые user messages (кроме ключевых цитат)

---

## Pipeline 6: CLI Subagent Invocation

Пайплайн для делегирования исследовательских задач подагентам.

```
┌──────────────────────────────────────────────┐
│  User: "How does authentication work here?" │
└────────────────┬─────────────────────────────┘
                 │
                 ▼
     ┌──────────────────────────────────────────┐
     │  AI Decision: Use subagent?             │
     │  • Will explore 10+ files               │
     │  • Pure research task                   │
     │  • CLI_SUBAGENTS prompt active          │
     └──────────┬──────────────────────────────┘
                │
                ▼
     ┌──────────────────────────────────────────┐
     │  CLI_SUBAGENTS prompt:                  │
     │  "Use Cline CLI for focused research   │
     │                                          │
     │  WHEN TO USE:                           │
     │  - Exploring 10+ files                  │
     │  - Users explicitly request it          │
     │                                          │
     │  PROHIBITED:                            │
     │  - Don't use for editing                │
     │  - Don't use for commands               │
     │                                          │
     │  FORMAT: cline 'your prompt here'"      │
     └──────────┬──────────────────────────────┘
                │
                ▼
     ┌──────────────────────────────────────────┐
     │  AI: Invoke subagent                    │
     │  <execute_command>                      │
     │  <command>                              │
     │  cline "Analyze authentication flow.    │
     │  Find all files involved in login,      │
     │  session management, and token          │
     │  validation. Provide brief technical    │
     │  summary."                              │
     │  </command>                             │
     │  </execute_command>                      │
     └──────────┬──────────────────────────────┘
                │
                ▼
     ┌──────────────────────────────────────────┐
     │  CLI Spawns Subagent                    │
     │  • New Cline instance                   │
     │  • Fresh context                        │
     │  • Same API key                         │
     │  • Isolated exploration                 │
     └──────────┬──────────────────────────────┘
                │
                ▼
     ┌──────────────────────────────────────────┐
     │  Subagent explores:                     │
     │  1. <search_files> for "auth"           │
     │  2. <read_file> for key files           │
     │  3. <list_code_definition_names>        │
     │  4. Analyzes patterns                   │
     └──────────┬──────────────────────────────┘
                │
                ▼
     ┌──────────────────────────────────────────┐
     │  Subagent returns summary:              │
     │                                          │
     │  "Authentication System Analysis:       │
     │                                          │
     │  Core Files:                            │
     │  - src/auth/authenticate.ts: Main logic│
     │  - src/middleware/auth.ts: JWT verify  │
     │  - src/models/User.ts: User schema     │
     │                                          │
     │  Flow:                                  │
     │  1. Login → generates JWT (HS256)      │
     │  2. Token stored in httpOnly cookie    │
     │  3. Middleware validates on requests   │
     │  4. Refresh logic in /auth/refresh     │
     │                                          │
     │  Technologies: JWT, bcrypt, passport"   │
     └──────────┬──────────────────────────────┘
                │
                ▼
     ┌──────────────────────────────────────────┐
     │  Main agent receives result             │
     │  • Compact summary                      │
     │  • No context overflow                  │
     │  • Can now answer user                  │
     └──────────┬──────────────────────────────┘
                │
                ▼
     ┌──────────────────────────────────────────┐
     │  AI responds to user:                   │
     │  "The authentication system uses JWT    │
     │  tokens with HS256 signing. Here's      │
     │  how it works: [detailed explanation    │
     │  based on subagent findings]"           │
     └──────────────────────────────────────────┘
```

**Ключевые промпты:**

- **CLI_SUBAGENTS**: When/how to use subagents
- **Subagent system prompt**: Same as main agent
- **OBJECTIVE**: Subagent follows same task methodology

**Изоляция:**

- Subagent не имеет доступа к контексту родителя
- Subagent не может редактировать файлы
- Subagent не может выполнять команды (кроме исследовательских)
- Результат возвращается как текст

**Предотвращение рекурсии:**

- CLI_SUBAGENTS компонент не включается если `context.isSubagent === true`
- Subagent не может создать еще один subagent

---

## Pipeline 7: MCP Tool Usage

Пайплайн использования инструментов Model Context Protocol.

```
┌──────────────────────────────────────────┐
│  User: "Check the weather in New York"  │
└────────────────┬─────────────────────────┘
                 │
                 ▼
     ┌──────────────────────────────────────┐
     │  MCP Hub: Load connected servers    │
     │  {                                   │
     │    "weather-server": {              │
     │      command: "node",               │
     │      args: ["weather-mcp"],         │
     │      tools: [{                      │
     │        name: "get_weather",         │
     │        inputSchema: {...}           │
     │      }]                             │
     │    }                                 │
     │  }                                   │
     └──────────┬──────────────────────────┘
                │
                ▼
     ┌──────────────────────────────────────┐
     │  MCP component in system prompt:    │
     │  "The Model Context Protocol        │
     │   enables communication with        │
     │   MCP servers.                      │
     │                                      │
     │   Connected: weather-server         │
     │   Tools:                            │
     │   - get_weather(location: string)   │
     │     Returns current weather"        │
     └──────────┬──────────────────────────┘
                │
                ▼
     ┌──────────────────────────────────────┐
     │  Convert MCP tools to Cline format  │
     │  mcpToolToClineToolSpec():          │
     │  {                                   │
     │    id: "weather-server:get_weather",│
     │    name: "use_mcp_tool",            │
     │    description: "Get weather...",   │
     │    parameters: [{                   │
     │      name: "server",                │
     │      value: "weather-server"        │
     │    }, {                             │
     │      name: "tool_name",             │
     │      value: "get_weather"           │
     │    }, {                             │
     │      name: "arguments",             │
     │      type: "object",                │
     │      properties: {...}              │
     │    }]                               │
     │  }                                   │
     └──────────┬──────────────────────────┘
                │
                ▼
     ┌──────────────────────────────────────┐
     │  AI: Call MCP tool                  │
     │  <use_mcp_tool>                     │
     │  <server>weather-server</server>    │
     │  <tool_name>get_weather</tool_name> │
     │  <arguments>                        │
     │  {                                   │
     │    "location": "New York"           │
     │  }                                   │
     │  </arguments>                       │
     │  </use_mcp_tool>                     │
     └──────────┬──────────────────────────┘
                │
                ▼
     ┌──────────────────────────────────────┐
     │  MCP Hub: Route to server           │
     │  1. Find server by ID               │
     │  2. Validate tool exists            │
     │  3. Validate arguments schema       │
     └──────────┬──────────────────────────┘
                │
                ▼
     ┌──────────────────────────────────────┐
     │  Execute on MCP server              │
     │  • Send JSON-RPC request            │
     │  • Wait for response                │
     └──────────┬──────────────────────────┘
                │
                ▼
     ┌──────────────────────────────────────┐
     │  MCP Server response:               │
     │  {                                   │
     │    "temperature": 72,               │
     │    "condition": "Partly Cloudy",    │
     │    "humidity": 65,                  │
     │    "wind_speed": 8                  │
     │  }                                   │
     └──────────┬──────────────────────────┘
                │
                ▼
     ┌──────────────────────────────────────┐
     │  Format response for AI             │
     │  "MCP tool 'get_weather' result:    │
     │   Temperature: 72°F                 │
     │   Condition: Partly Cloudy          │
     │   Humidity: 65%                     │
     │   Wind: 8 mph"                      │
     └──────────┬──────────────────────────┘
                │
                ▼
     ┌──────────────────────────────────────┐
     │  AI responds to user:               │
     │  "The current weather in New York:  │
     │   • Temperature: 72°F               │
     │   • Partly Cloudy                   │
     │   • Humidity: 65%                   │
     │   • Wind speed: 8 mph"              │
     └──────────────────────────────────────┘
```

**Ключевые промпты:**

- **MCP component**: Динамически генерируемая документация серверов
- **loadMcpDocumentation**: Guide по созданию MCP серверов

**Данные:**

- **Server config** → **MCP Hub**: Connection details
- **Hub** → **System Prompt**: Available tools
- **AI** → **Hub**: Tool call with arguments
- **Hub** → **Server**: JSON-RPC request
- **Server** → **Hub**: Result
- **Hub** → **AI**: Formatted response

**MCP Resources** (отдельный поток):

```
AI → <access_mcp_resource uri="file://path/to/doc">
Hub → Fetch resource from server
Hub → Return content
AI → Process content
```

---

## Pipeline 8: New Task Creation

Пайплайн создания новой задачи с сохранением контекста.

```
┌──────────────────────────────────────────┐
│  User: "/newtask Refactor to TypeScript"│
└────────────────┬─────────────────────────┘
                 │
                 ▼
     ┌──────────────────────────────────────┐
     │  Detect special command             │
     │  • /newtask recognized              │
     │  • Trigger newTaskToolResponse()    │
     └──────────┬──────────────────────────┘
                │
                ▼
     ┌──────────────────────────────────────┐
     │  Inject explicit_instructions       │
     │  <explicit_instructions              │
     │   type="new_task">                  │
     │  The user asked you to create       │
     │  new task with preloaded context.   │
     │                                      │
     │  You are ONLY allowed to respond    │
     │  by calling the new_task tool.      │
     │                                      │
     │  Parameters:                        │
     │  - context: Detailed summary        │
     │  - technical_concepts: Key tech     │
     │  - relevant_files: Files list       │
     │  - problem_solving: What solved     │
     │  - pending_tasks: What remains      │
     │  - task_progress: Current checklist │
     │  </explicit_instructions>            │
     └──────────┬──────────────────────────┘
                │
                ▼
     ┌──────────────────────────────────────┐
     │  AI: Generate context summary       │
     │  (similar to summarizeTask)         │
     │  • Analyzes conversation history    │
     │  • Identifies key work done         │
     │  • Captures technical details       │
     │  • Notes pending items              │
     └──────────┬──────────────────────────┘
                │
                ▼
     ┌──────────────────────────────────────┐
     │  AI: Call new_task tool             │
     │  <new_task>                         │
     │  <context>                          │
     │  Working on authentication system.   │
     │  Implemented JWT login, middleware. │
     │  User now wants to refactor to TS.  │
     │  </context>                         │
     │                                      │
     │  <technical_concepts>               │
     │  - JWT with HS256                   │
     │  - bcrypt password hashing          │
     │  - Express middleware pattern       │
     │  </technical_concepts>              │
     │                                      │
     │  <relevant_files>                   │
     │  - src/auth/login.js                │
     │  - src/auth/middleware.js           │
     │  - src/models/User.js               │
     │  </relevant_files>                  │
     │                                      │
     │  <problem_solving>                  │
     │  Fixed token expiration bug by...   │
     │  </problem_solving>                 │
     │                                      │
     │  <pending_tasks>                    │
     │  Now need to refactor to TypeScript │
     │  </pending_tasks>                   │
     │                                      │
     │  <task_progress>                    │
     │  - [x] Implement JWT login          │
     │  - [x] Add auth middleware          │
     │  - [ ] Refactor to TypeScript       │
     │  </task_progress>                   │
     │  </new_task>                         │
     └──────────┬──────────────────────────┘
                │
                ▼
     ┌──────────────────────────────────────┐
     │  System: Create new task            │
     │  • Generate task ID                 │
     │  • Store context data               │
     │  • Clear conversation               │
     └──────────┬──────────────────────────┘
                │
                ▼
     ┌──────────────────────────────────────┐
     │  New Task Initialized               │
     │  messages = [                       │
     │    {                                 │
     │      role: "system",                │
     │      content: systemPrompt          │
     │    },                               │
     │    {                                 │
     │      role: "user",                  │
     │      content: "Context: ..." +      │
     │               "Task: Refactor TS"   │
     │    }                                 │
     │  ]                                   │
     └──────────┬──────────────────────────┘
                │
                ▼
     ┌──────────────────────────────────────┐
     │  AI starts fresh with context       │
     │  • Knows full background            │
     │  • Understands current state        │
     │  • Can proceed with TypeScript      │
     │  • Has relevant file paths          │
     │  • Knows what's already done        │
     └──────────────────────────────────────┘
```

**Ключевые промпты:**

- **new_task command prompt**: Explicit instructions
- **Parameters definition**: XML-formatted structure

**Преимущества:**

- Clean slate (fresh token count)
- Preserved context (не нужно переспрашивать)
- Organized task boundaries
- Easy task switching

**Данные передаются:**

- Full context summary
- Technical concepts
- File paths
- Current progress
- Problem-solving history

---

## Data Flow Patterns

### Pattern 1: Prompt Component → Placeholder → Final Prompt

```
Component Function            Placeholder Object         Template Rendering
==================            ===================        ==================

agent_role.ts                 {
  ↓                             AGENT_ROLE:              "You are Cline,
return "You are Cline..."  →    "You are Cline..."   →   a highly skilled..."
                              }
                                                         ╔════════════════╗
capabilities.ts               {                          ║ Final Prompt:  ║
  ↓                             CAPABILITIES:            ║                ║
return "You can execute..."  →  "You can execute..."  → ║ You are Cline, ║
                              }                          ║ ...            ║
                                                         ║ You can exec...║
objective.ts                  {                          ║ ...            ║
  ↓                             OBJECTIVE:               ║ You accomplish ║
return "You accomplish..."   →  "You accomplish..."   → ║ a given task...║
                              }                          ╚════════════════╝
```

### Pattern 2: Context Accumulation

```
Iteration 1:
  User: "Fix bug"
  messages = [
    {user: "Fix bug"}
  ]

Iteration 2:
  AI: <read_file>
  Result: [file content]
  messages = [
    {user: "Fix bug"},
    {assistant: tool_call},
    {tool: result}
  ]

Iteration 3:
  AI: <replace_in_file>
  Result: "Success"
  messages = [
    {user: "Fix bug"},
    {assistant: tool_call_1},
    {tool: result_1},
    {assistant: tool_call_2},
    {tool: result_2}
  ]

Context grows with each tool use
→ System prompt stays the same
→ Tools list stays the same
→ Only messages array grows
```

### Pattern 3: Tool Result → Next Prompt

```
Tool Execution                   Format Response              Next AI Input
==============                   ===============              =============

read_file("auth.ts")             "File content:               messages.push(
  ↓                              [formatted file]             {tool: formatted}
Returns raw content          →   Line numbers added      →   )
                                 Truncation notices
                                 Error handling                AI sees full
                                                              conversation
replace_in_file(...)             "File edited.                including this
  ↓                              Auto-format applied:         formatted result
Returns success + diff       →   - Indent changed        →
                                 - Quotes converted"           Can reference
                                                              changes made
execute_command("npm test")      "Command output:
  ↓                              [test results]
Returns stdout/stderr        →   Exit code: 0            →
                                 Duration: 2.3s"
```

### Pattern 4: Mode Transitions

```
State: PLAN MODE                    State: ACT MODE
==================                  ================

System Prompt:                      System Prompt:
  ACT_VS_PLAN (plan section)          ACT_VS_PLAN (act section)

Tools:                              Tools:
  ✓ read_file                         ✓ ALL tools
  ✓ search_files                      ✓ write_to_file
  ✓ list_files                        ✓ replace_in_file
  ✓ plan_mode_respond                 ✓ execute_command
  ✗ write_to_file                     ✓ attempt_completion
  ✗ replace_in_file                   ✗ plan_mode_respond
  ✗ attempt_completion

AI behavior:                        AI behavior:
  - Explores context                  - Executes plan
  - Asks questions                    - Creates checklist
  - Presents plan                     - Makes changes
  - Waits for approval                - Signals completion

             │                                   ▲
             │  User approves plan               │
             └───────────────────────────────────┘
                   Transition happens:
                   - Re-build system prompt
                   - Switch tool set
                   - Preserve conversation
```

### Pattern 5: Variant Selection Logic

```
Input Context:
  provider: "anthropic"
  modelId: "claude-3-5-sonnet-20241022"
  modelFamily: "next-gen"

         │
         ▼
PromptRegistry.get(context)
         │
         ▼
    Match Logic:
    =============
    1. Check NEXT_GEN matcher → ✓ Match!
    2. Return next-gen variant

    (If no match)
    3. Fallback to GENERIC

         │
         ▼
    Variant Selected:
      id: "next-gen-v1"
      components: [13 sections]
      tools: [17 tools]

         │
         ▼
    Build with this variant:
      - Use next-gen templates
      - Include all components
      - Add update_task_todo tool
```

---

## Заключение

Система пайплайнов Cline демонстрирует сложную архитектуру с:

1. **Модульная сборка промптов**
   - Component-based assembly
   - Template resolution
   - Dynamic placeholder injection

2. **Адаптивное выполнение**
   - Mode switching (Plan ↔ Act)
   - Context-aware tool filtering
   - Model-specific optimizations

3. **Управление состоянием**
   - Context accumulation
   - Conversation condensation
   - Task continuation

4. **Расширяемость**
   - MCP integration
   - CLI subagents
   - Custom tools

5. **Обработка данных**
   - Structured data flow
   - Format preservation
   - Error recovery

Каждый пайплайн тщательно спроектирован для эффективной передачи данных между этапами, минимизации потерь контекста и максимизации возможностей AI агента.

---

*Документация создана: 2025-11-13*
*Источник: github.com/cline/cline (main branch)*

