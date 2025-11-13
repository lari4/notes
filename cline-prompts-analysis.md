# Анализ промптов репозитория Cline

Дата анализа: 2025-11-13
Репозиторий: https://github.com/cline/cline

## Оглавление

1. [Общая архитектура системы промптов](#архитектура)
2. [Основные компоненты промптов](#компоненты)
3. [Инструменты (Tools)](#инструменты)
4. [Варианты промптов (Variants)](#варианты)
5. [Специальные промпты для команд](#команды)
6. [Управление контекстом](#контекст)
7. [Форматирование ответов](#ответы)

---

## Архитектура

### Расположение файлов

Система промптов находится в директории: `src/core/prompts/`

Структура:
```
src/core/prompts/
├── commands.ts                    # Промпты для специальных команд
├── contextManagement.ts           # Управление контекстом беседы
├── loadMcpDocumentation.ts        # Документация MCP серверов
├── responses.ts                   # Форматирование ответов
└── system-prompt/                 # Модульная система промптов
    ├── components/                # Переиспользуемые компоненты
    │   ├── act_vs_plan_mode.ts   # Режимы действия vs планирования
    │   ├── agent_role.ts         # Определение роли агента
    │   ├── capabilities.ts       # Описание возможностей
    │   ├── cli_subagents.ts      # CLI подагенты
    │   ├── editing_files.ts      # Редактирование файлов
    │   ├── feedback.ts           # Обратная связь
    │   ├── mcp.ts                # MCP интеграция
    │   ├── objective.ts          # Цели и задачи
    │   ├── rules.ts              # Операционные правила
    │   ├── system_info.ts        # Системная информация
    │   ├── task_progress.ts      # Отслеживание прогресса
    │   ├── user_instructions.ts  # Пользовательские инструкции
    │   └── tool_use/             # Использование инструментов
    │       ├── examples.ts       # Примеры использования
    │       ├── formatting.ts     # Форматирование
    │       ├── guidelines.ts     # Руководства
    │       └── tools.ts          # Определения инструментов
    ├── registry/                  # Реестр промптов и инструментов
    │   ├── ClineToolSet.ts       # Набор инструментов
    │   ├── PromptBuilder.ts      # Сборщик промптов
    │   └── PromptRegistry.ts     # Реестр вариантов
    ├── templates/                 # Шаблоны
    │   ├── TemplateEngine.ts     # Движок шаблонов
    │   └── placeholders.ts       # Определения плейсхолдеров
    ├── variants/                  # Варианты для разных моделей
    │   ├── generic/              # Базовый вариант
    │   ├── next-gen/             # Для продвинутых моделей
    │   ├── xs/                   # Компактный вариант
    │   ├── gpt-5/                # GPT-5 специфичный
    │   └── ...                   # Другие варианты
    ├── spec.ts                    # Спецификации инструментов
    ├── types.ts                   # Типы TypeScript
    └── index.ts                   # Главный экспорт
```

### Принципы работы

1. **Модульность**: Промпты собираются из переиспользуемых компонентов
2. **Вариативность**: Разные конфигурации для разных моделей AI
3. **Шаблонизация**: Плейсхолдеры заменяются динамическими значениями
4. **Контекстная адаптация**: Промпты адаптируются под контекст выполнения

---

## Компоненты

### 1. AGENT_ROLE - Роль агента

**Файл**: `src/core/prompts/system-prompt/components/agent_role.ts`

**Назначение**: Определяет базовую роль и идентичность AI ассистента

**Промпт**:
```
You are Cline, a highly skilled software engineer with extensive knowledge in many programming languages, frameworks, design patterns, and best practices.
```

**Особенности**:
- Может быть переопределен в вариантах
- Обрабатывается через TemplateEngine
- Устанавливает основную идентичность агента

---

### 2. CAPABILITIES - Возможности

**Файл**: `src/core/prompts/system-prompt/components/capabilities.ts`

**Назначение**: Описывает доступные агенту возможности и инструменты

**Ключевые части промпта**:

#### Основные возможности
```
You can execute CLI commands on the user's computer, list files, view source code definitions,
perform regex searches, and read and edit files on the user's computer.

When you start a task, you receive a recursive file structure overview showing all directories
and files in the project.
```

#### Инструменты для работы с файлами
```
- list_files: Recursively explore directories when passing 'true' for recursive parameter
- search_files: Perform regex queries across files with contextual results
- list_code_definition_names: Provides an overview of source code definitions for all files
  at the top level of specified directories
```

#### Выполнение команд
```
- execute_command: Runs CLI commands with support for interactive and long-running commands
  in the VSCode terminal
```

#### Браузерные возможности (условно)
```
When browser support is enabled:
- browser_action: Interact with websites through Puppeteer for web development tasks including
  navigation, element interaction, screenshots, and console log analysis
```

#### MCP серверы
```
You can leverage MCP servers that may provide different capabilities for enhanced task completion.
```

---

### 3. OBJECTIVE - Цели и задачи

**Файл**: `src/core/prompts/system-prompt/components/objective.ts`

**Назначение**: Определяет подход к выполнению задач

**Промпт**:
```
You accomplish a given task iteratively, breaking it down into clear steps and working through
them methodically.

1. Task Analysis - Decompose user requests into prioritized, logical goals
2. Sequential Execution - Work through goals one at a time, utilizing available tools as needed
3. Thoughtful Tool Selection - Before invoking any tool, analyze file structures and determine
   the most relevant tool for the task, verifying all required parameters are present or
   reasonably inferable
4. Completion Protocol - Use attempt_completion tool to deliver results and optionally provide
   CLI commands
5. Iteration - Accept user feedback for refinement without prolonging unnecessary dialogue

IMPORTANT: Do NOT continue in pointless back and forth conversations. Don't end your responses
with questions or offers for further assistance.
```

**Особенности**:
- Включает логику "yolo mode" для автоматического вывода параметров
- Акцент на итеративном подходе
- Избегание ненужных диалогов

---

### 4. EDITING_FILES - Редактирование файлов

**Файл**: `src/core/prompts/system-prompt/components/editing_files.ts`

**Назначение**: Руководство по редактированию файлов

**Промпт**:
```
You have access to two file-editing tools:

write_to_file: Used for creating new files or completely replacing existing content.
Most appropriate for initial scaffolding, extensive reorganizations, or boilerplate generation.

replace_in_file: Designed for targeted, localized edits to specific sections without
rewriting entire files.

RECOMMENDATION: Default to replace_in_file for most changes. It's the safer, more precise
option that minimizes potential issues.

IMPORTANT WORKFLOW:
- Assess change scope before selecting a tool
- When multiple edits are needed in one file, use a single replace_in_file call with
  multiple SEARCH/REPLACE blocks rather than making successive calls
- After editing, auto-formatting may modify the file (adjusting indentation, quotes,
  imports, semicolons, etc.)
- Use the final formatted state as reference for subsequent edits

AUTO-FORMATTING CONSIDERATIONS:
Editors may automatically reformat code through:
- Quote conversion
- Import organization
- Brace style standardization
- Semicolon adjustments
- Indentation normalization

These final results should guide any follow-up modifications.
```

---

### 5. ACT_VS_PLAN_MODE - Режимы работы

**Файл**: `src/core/prompts/system-prompt/components/act_vs_plan_mode.ts`

**Назначение**: Объясняет два режима работы агента

**Промпт**:
```
You have two operational modes:

ACT MODE: Grants full tool access except for plan_mode_respond. You use available tools
to complete tasks and signal completion via the attempt_completion tool.

PLAN MODE: Provides access to the plan_mode_respond tool specifically. This mode enables
collaborative planning where you:
- Gather context using read_file or search_files
- Ask clarifying questions (unless YOLO mode is enabled)
- Present structured plans for user review
- Discuss and refine approaches interactively

You transition to ACT MODE only after user approval of the plan.
```

**Особенности**:
- Поддержка двух режимов работы
- Условная логика для YOLO mode
- Кастомизация через template overrides

---

### 6. RULES - Операционные правила

**Файл**: `src/core/prompts/system-prompt/components/rules.ts`

**Назначение**: Устанавливает операционные ограничения и правила

**Ключевые правила**:

#### Работа с директориями
```
- The agent operates from a fixed working directory and cannot change directories
- Must use full paths when referencing files outside the current directory
- Cannot use ~ or $HOME for home directory references
```

#### Использование инструментов
```
- Browser actions: Should be used for generic tasks (weather, news) when MCP server
  tools aren't available
- File operations: Use search_files for pattern matching, read_file for examination,
  and replace_in_file for modifications
- Project creation: New projects should be organized in dedicated directories with
  logical structure
- Command execution: Must consider system compatibility and prepend cd commands when necessary
```

#### Стиль ответов
```
The agent should:
- Avoid opening conversational phrases ("Great," "Certainly," "Sure")
- Be direct and technical rather than conversational
- Never end responses with questions
- Wait for user confirmation after each tool use before proceeding
- Use attempt_completion tool to present final results
```

#### Обработка информации
```
- Leverage vision capabilities when images are provided
- Use environment context without assuming explicit user requests
- Don't ask unnecessary questions; use available tools to gather information
- Avoid back-and-forth conversation; focus on task completion
```

---

### 7. TOOL_USE - Использование инструментов

**Файл**: `src/core/prompts/system-prompt/components/tool_use/`

#### 7.1 Guidelines - Руководства

**Файл**: `guidelines.ts`

**Промпт**:
```
TOOL USE GUIDELINES:

1. Assess existing information and identify gaps before using tools
2. Select appropriate tools based on task requirements
3. Use tools iteratively with one per message
4. Follow XML formatting specifications for tool calls
5. Wait for user feedback after each tool use
6. Never assume tool success without confirmation
```

#### 7.2 Formatting - Форматирование

**Файл**: `formatting.ts`

**Промпт**:
```
Tool calls use XML-style tag formatting with parameters enclosed in nested tags.

Example:
<read_file>
<path>/path/to/file.txt</path>
</read_file>

[When focus chain is enabled, additional section for task progress checklist formatting]
```

#### 7.3 Examples - Примеры

**Файл**: `examples.ts`

**Содержит примеры**:
1. Command execution - `<execute_command>` синтаксис для запуска npm скриптов
2. File creation - `<write_to_file>` с JSON конфигурацией
3. Task creation - `<new_task>` структура с секциями контекста
4. File editing - `<replace_in_file>` с diff-style синтаксисом
5. MCP tool usage - два примера формата `<use_mcp_tool>`

**Особенности**:
- Условно включает элементы "focus chain" для отслеживания прогресса задач
- Использует плейсхолдеры, разрешаемые через TemplateEngine

---

### 8. MCP - Model Context Protocol

**Файл**: `src/core/prompts/system-prompt/components/mcp.ts`

**Назначение**: Интеграция MCP серверов

**Промпт**:
```
The Model Context Protocol (MCP) enables communication between the system and locally
running MCP servers that provide additional tools and resources to extend your capabilities.

[Динамически генерируемая документация для подключенных MCP серверов, включающая:]
- Server name and execution command
- Available tools with input schemas
- Resource templates with URI patterns
- Direct resources with descriptions
```

**Структура документации**:
- Каждый сервер представлен заголовком с командой
- Инструменты с JSON схемами входных данных
- Шаблоны ресурсов и прямые ресурсы отдельно
- Последовательное форматирование для парсинга

---

### 9. TASK_PROGRESS - Отслеживание прогресса

**Файл**: `src/core/prompts/system-prompt/components/task_progress.ts`

**Назначение**: Отслеживание прогресса многоэтапных задач

**Варианты промптов**:

#### Стандартный вариант
```
Track progress on multi-step tasks using task_progress parameter with markdown checklists:
- [ ] for incomplete items
- [x] for completed items
```

#### Native Next-Gen вариант
```
The task_progress must be included as a separate parameter in the tool, it should not be
included inside other content or argument blocks.
```

#### Native GPT-5 вариант
```
The task_progress MUST be included as a separate parameter in the tool, it should NOT be
included inside other content or argument blocks.
```

**Ключевые рекомендации**:
- Создавать comprehensive todo списки при переходе в action mode
- Обновлять чеклисты без уведомления пользователя
- Фокусироваться на значимых вехах, а не на технических деталях
- Переписывать чеклисты при изменении scope
- Отмечать пункты как выполненные немедленно

---

### 10. CLI_SUBAGENTS - CLI подагенты

**Файл**: `src/core/prompts/system-prompt/components/cli_subagents.ts`

**Назначение**: Инструкции по использованию CLI подагентов

**Промпт**:
```
Use Cline CLI tool to create AI subagents for focused research and exploration tasks,
preventing context overflow.

WHEN TO USE:
- When exploring 10 or more files
- When users explicitly request it

PROHIBITED USES:
Do not use subagents for editing code or executing commands - limit to reading and research.

COMMAND FORMAT:
cline "your prompt here"

EXAMPLE APPLICATIONS:
- Pattern identification (locating React components with specific hooks)
- Architecture analysis (tracing authentication flows)
- API cataloging
- Directory summaries
- Cross-application feature mapping

STRATEGIC RECOMMENDATIONS:
- Request brief, technically dense summaries
- Explore large/complex files through subagents first before direct reading
```

**Условия включения**:
- CLI установлен
- Подагенты включены
- Текущий контекст не является CLI подагентом (предотвращение рекурсии)

---

### 11. SYSTEM_INFO - Системная информация

**Файл**: `src/core/prompts/system-prompt/components/system_info.ts`

**Назначение**: Предоставляет информацию о системном окружении

**Шаблон**:
```
Operating System: {{os}}
IDE: {{ide}}
Default Shell: {{shell}}
Home Directory: {{homeDir}}

[Для single-root workspace:]
Working Directory: {{workingDir}}

[Для multi-root workspace:]
Workspaces:
- {{workspace1}} ({{vcs}})
- {{workspace2}} ({{vcs}})
...
```

**Функции**:
- `getSystemEnv()` - получает детали окружения (OS, shell, home dir)
- `getSystemInfo()` - обрабатывает данные и различает single vs multi-root workspaces
- Поддержка тестового режима через переменные окружения

---

### 12. USER_INSTRUCTIONS - Пользовательские инструкции

**Файл**: `src/core/prompts/system-prompt/components/user_instructions.ts`

**Назначение**: Агрегирует пользовательские инструкции из разных источников

**Промпт**:
```
The following additional instructions are provided by the user, and should be followed
to the best of your ability without interfering with the TOOL USE guidelines.

[Здесь вставляются инструкции из различных файлов правил в порядке приоритета]
```

**Порядок приоритета источников**:
1. Preferred language settings
2. Global Cline rules
3. Local Cline rules
4. Cursor rules (file and directory)
5. Windsurf rules
6. Agents rules
7. Ignore file instructions

**Особенности**:
- Инструкции объединяются с двойными переводами строк
- Использует TemplateEngine для рендеринга
- Layered подход: от широких к специфичным конфигурациям

---

### 13. FEEDBACK - Обратная связь

**Файл**: `src/core/prompts/system-prompt/components/feedback.ts`

**Назначение**: Направляет пользователей к документации и поддержке

**Промпт**:
```
For help or feedback, report the issue using the /reportbug slash command in the chat.

When users ask capability questions about Cline, direct to documentation at:
https://docs.cline.bot

TOPIC SECTIONS:
- getting-started: Installation and basics
- model-selection: Configuration options for various AI models
- features: Core functionalities like auto-approval and workflows
- task-management: Context handling
- prompt-engineering: Skill improvement
- cline-tools: References and slash commands
- mcp: Integration protocols
- enterprise: Business deployments
- more-info: Telemetry and reference materials
```

**Особенности**:
- Включается только когда focus chain settings включены
- Поддерживает component overrides

---

## Инструменты

### Архитектура инструментов

**Файл**: `src/core/prompts/system-prompt/registry/ClineToolSet.ts`

#### Класс ClineToolSet

**Основные методы**:

```typescript
// Регистрация инструмента
register(variantId, tool): void

// Получение инструментов для варианта с fallback к GENERIC
getTools(variantId): ClineToolSpec[]

// Поиск конкретного инструмента
getToolByName(variantId, toolName): ClineToolSpec | undefined

// Поиск с fallback: exact variant → GENERIC → all variants
getToolByNameWithFallback(variantId, toolName): ClineToolSpec | undefined

// Фильтрация по контексту
getEnabledTools(variantId, context): ClineToolSpec[]

// Конвертация для разных провайдеров
getNativeConverter(providerId): Function
getNativeTools(variant, context): NativeToolDefinition[]
```

**Иерархия fallback**:
1. Точный вариант
2. GENERIC вариант
3. Любой доступный вариант

**Паттерны дизайна**:
- Дедупликация по ID конфигурации
- Контекстная фильтрация (context-aware)
- Поддержка MCP инструментов

### Конвертация MCP инструментов

**Функция**: `mcpToolToClineToolSpec`

**Процесс**:
1. Извлечение input schemas
2. Маппинг properties в parameters
3. Сохранение JSON Schema полей: `enum`, `format`, `minimum`, `maximum`
4. Генерация идентификаторов (server UID + tool name)
5. Создание описательных имен

### Спецификация инструментов

**Файл**: `src/core/prompts/system-prompt/spec.ts`

**Интерфейс ClineToolSpec**:
```typescript
{
  variant: string
  id: string
  name: string
  description: string
  instructions?: string
  includeIf?: (context) => boolean
  parameters: ClineToolSpecParameter[]
}
```

**ClineToolSpecParameter**:
```typescript
{
  name: string
  type: "string" | "boolean" | "integer" | "array" | "object"
  description: string
  required: boolean
  includeIf?: (context) => boolean
  dependsOn?: string
  // JSON Schema fields для MCP
}
```

### Функции конвертации

#### 1. toolSpecFunctionDefinition()
**Назначение**: Генерирует OpenAI ChatCompletionTool формат

**Особенности**:
- Strict parameter validation
- additionalProperties отключены

#### 2. toolSpecInputSchema()
**Назначение**: Создает Anthropic Tool definitions

**Особенности**:
- Эквивалентная структура параметров

#### 3. toolSpecFunctionDeclarations()
**Назначение**: Генерирует Google Gemini function declarations

**Особенности**:
- Provider-specific type mapping (integer → NUMBER)

#### 4. Cross-provider conversions
- `openAIToolToAnthropic()`: OpenAI → Anthropic
- `toOpenAIResponsesAPITool()`: Универсальная конвертация

### Список основных инструментов

По данным из конфигураций вариантов:

1. **execute_command** / **bash** - Выполнение CLI команд
2. **read_file** - Чтение файлов
3. **write_to_file** - Создание новых файлов
4. **replace_in_file** - Редактирование файлов
5. **list_files** - Листинг директорий
6. **search_files** - Поиск по регулярным выражениям
7. **list_code_definition_names** - Обзор определений кода
8. **browser_action** - Взаимодействие с браузером
9. **fetch** - Загрузка данных из URL
10. **use_mcp_tool** - Использование MCP инструментов
11. **access_mcp_resource** - Доступ к MCP ресурсам
12. **ask_followup_question** - Задать уточняющий вопрос
13. **attempt_completion** - Завершение задачи
14. **new_task** - Создание новой задачи
15. **plan_mode_respond** - Ответ в режиме планирования
16. **ask_docs** - Обращение к документации
17. **update_task_todo** - Обновление todo списка

---

## Варианты

### Система вариантов

**Назначение**: Оптимизация промптов для разных моделей AI

**Основные варианты**:

### 1. GENERIC - Базовый вариант

**Файл**: `src/core/prompts/system-prompt/variants/generic/config.ts`

**Характеристики**:
- **Model Family**: GENERIC
- **Version**: 1
- **Назначение**: Fallback конфигурация для любых моделей

**Matcher логика**:
```typescript
// Применяется к моделям, которые:
- Не имеют provider или model ID информации, ИЛИ
- Не являются локальными моделями с compact prompts, ИЛИ
- Не являются next-generation моделями, ИЛИ
- Не являются GLM-family моделями
```

**Секции промпта** (13):
1. Agent role
2. Tool use guidance
3. Task progress
4. MCP integration
5. File editing
6. Plan vs execution
7. CLI subagents
8. Capabilities
9. Rules
10. System info
11. Objectives
12. User instructions
13. Feedback

**Инструменты** (16):
bash, read_file, write_to_file, replace_in_file, list_files, search_files,
list_code_definition_names, browser_action, fetch, use_mcp_tool, access_mcp_resource,
ask_followup_question, attempt_completion, new_task, plan_mode_respond, ask_docs

**Шаблон**:
```typescript
{{AGENT_ROLE}}
=====
{{TOOL_USE}}
=====
{{TASK_PROGRESS}}
=====
{{MCP}}
=====
{{EDITING_FILES}}
=====
{{ACT_VS_PLAN}}
=====
{{CLI_SUBAGENTS}}
=====
{{CAPABILITIES}}
=====
{{FEEDBACK}}
=====
{{RULES}}
=====
{{SYSTEM_INFO}}
=====
{{OBJECTIVE}}
=====
{{USER_INSTRUCTIONS}}
```

---

### 2. NEXT_GEN - Продвинутые модели

**Файл**: `src/core/prompts/system-prompt/variants/next-gen/config.ts`

**Характеристики**:
- **Model Family**: NEXT_GEN
- **Version**: 1
- **Tags**: "next-gen", "advanced", "production"
- **Назначение**: Оптимизация для frontier моделей с enhanced agentic capabilities

**Matcher логика**:
```typescript
// Применяется к:
- Next-gen model family без native tool calls
- Исключает compact prompts на локальных моделях
- Исключает next-gen providers с определенными условиями
- Избегает GPT-5 non-chat вариантов
```

**Секции промпта** (13):
Те же, что у GENERIC

**Инструменты** (17):
Все из GENERIC + `update_task_todo` - обновление todo списка

**Особенности**:
- Sophisticated multi-tool orchestration
- Suited for complex autonomous reasoning tasks
- Compile-time validation с strict mode

**Шаблон**:
Использует baseTemplate с теми же секциями

**Специальный rules_template**:
```typescript
// Включает детальные операционные правила:
- Directory constraints
- Command execution
- File operations
- Project creation
- Communication style (direct, technical, non-conversational)
- Tool usage protocol
- User interaction (с учетом "yolo mode")
```

---

### 3. XS - Компактный вариант

**Файл**: `src/core/prompts/system-prompt/variants/xs/config.ts`

**Характеристики**:
- **Model Family**: XS
- **Version**: 1
- **Tags**: "local", "xs", "compact"
- **Назначение**: Для моделей с малым контекстом

**Matcher логика**:
```typescript
// Применяется к:
- Custom prompt set = "compact"
- AND local model provider
```

**Секции промпта** (9) - сокращенный набор:
1. Agent role
2. Rules
3. Act vs plan
4. CLI subagents
5. Capabilities
6. Editing files
7. Objective
8. System info
9. User instructions

**Инструменты** (10) - минимальный набор:
bash, read_file, write_to_file, replace_in_file, search_files, list_files,
ask_followup_question, attempt_completion, new_task, plan_mode_respond

**Особенности**:
- Compact footprint для малого контекста
- Compile-time validation
- Component overrides применяются post-build

---

### 4. Другие варианты

Согласно структуре директорий, существуют также:

- **gpt-5/** - Специфичный для GPT-5
- **native-gpt-5/** - Native tool calls для GPT-5
- **native-gpt-5-1/** - Версия 1 native GPT-5
- **native-next-gen/** - Native tool calls для next-gen
- **glm/** - Для GLM моделей
- **hermes/** - Для Hermes моделей

---

## Команды

### Специальные промпты для команд

**Файл**: `src/core/prompts/commands.ts`

### 1. new_task - Новая задача

**Функция**: `newTaskToolResponse()`

**Назначение**: Генерация контекста для новой задачи

**Промпт**:
```xml
<explicit_instructions type="new_task">
The user has explicitly asked you to help them create a new task with preloaded context,
which you will generate. The user may have provided instructions or additional information
for you to consider when summarizing existing work and creating the context for the new task.

Irrespective of whether additional information or instructions are given, you are ONLY
allowed to respond to this message by calling the new_task tool.

[XML-formatted instructions defining parameters:]
- context: Detailed summary covering current work
- technical_concepts: Key technologies and patterns
- relevant_files: Files examined or modified
- problem_solving: Solved issues and approaches
- pending_tasks: Remaining work

[Optional when focus chain enabled:]
- task_progress: Current task checklist
</explicit_instructions>
```

---

### 2. condense - Сжатие контекста

**Функция**: `condenseToolResponse()`

**Назначение**: Создание сводки беседы для сжатия контекста

**Промпт**:
```xml
<explicit_instructions type="condense">
[Similar structure to new_task]

Create conversation summary to compact context windows while preserving:
- Technical details
- Code patterns
- Architectural decisions

[Includes optional task_progress parameter]
</explicit_instructions>
```

---

### 3. new_rule - Новое правило

**Функция**: `newRuleToolResponse()`

**Назначение**: Создание Cline rule файлов

**Промпт**:
```xml
<explicit_instructions type="new_rule">
Create Cline rule files (markdown format) within .clinerules/ directory.

Files should document:
- Coding conventions
- Communication style
- Development workflows
- Project-specific preferences
</explicit_instructions>
```

---

### 4. report_bug - Отчет об ошибке

**Функция**: `reportBugToolResponse()`

**Назначение**: Подача GitHub issues

**Промпт**:
```xml
<explicit_instructions type="report_bug">
Submit GitHub issue requiring:
- title: Issue title
- what_happened: Description
- steps_to_reproduce: Reproduction steps
- api_request_output: API output (optional)
- additional_context: Extra context (optional)
</explicit_instructions>
```

---

### 5. subagent - Подагент

**Функция**: `subagentToolResponse()`

**Назначение**: Вызов Cline CLI подагентов

**Промпт**:
```
Transform user request into executable command:
cline "<prompt>"
```

---

### 6. deep_planning - Глубокое планирование

**Функция**: `deepPlanningToolResponse()`

**Назначение**: Генерация deep-planning промптов

**Особенности**:
- Принимает optional focus chain settings
- Учитывает API provider информацию
- Model-family-aware variant selection

---

## Контекст

### Управление контекстом

**Файл**: `src/core/prompts/contextManagement.ts`

### 1. summarizeTask - Сводка задачи

**Функция**: `summarizeTask()`

**Назначение**: Создание детальной сводки беседы при приближении к лимиту контекста

**Промпт**:
```
When context limits are approached, create comprehensive summary covering:

REQUIRED SECTIONS:
1. Primary Request and Intent - User's explicit requests
2. Key Technical Concepts - Technologies, frameworks, patterns discussed
3. Files and Code Sections - Specific files examined/modified with details
4. Problem Solving - Issues solved, attempts made, lessons learned
5. Pending Tasks - Remaining work, blockers
6. Task Evolution - How task evolved from original to current with user quotes
7. Current Work - What you were doing immediately before summary
8. Next Step - Following action aligned with user requests
9. Required Files - File paths necessary to continue

EMPHASIS:
Capture technical details, code patterns, and architectural decisions essential
for continuing development without context loss.

[Optional when focus chain enabled:]
- task_progress: Current checklist
```

**Особенности**:
- CWD handling в POSIX формате
- Multi-root workspace support с `@workspace:path` нотацией
- Структурированные требования к 9 секциям
- Focus chain integration

---

### 2. continuationPrompt - Продолжение

**Функция**: `continuationPrompt()`

**Назначение**: Возобновление беседы из предыдущей сводки

**Промпт**:
```
Continue from where work was suspended.

INSTRUCTIONS:
- Continue work based on summary
- Ignore special commands (/newtask, /compact, /smol) unless explicitly repeated by user
- Do not ask clarification questions about the summary
- Focus on the most recent user message
```

**Особенности**:
- Вставляет summary из предыдущей сессии
- Приоритет последнему сообщению пользователя
- Специальные команды требуют повторного выполнения

---

## Ответы

### Форматирование ответов

**Файл**: `src/core/prompts/responses.ts`

**Объект**: `formatResponse`

### Основные методы:

#### 1. Уведомления

```typescript
duplicateFileReadNotice()
// "This file read has been removed to save space in the context window.
// Refer to the latest file read for the most up to date version."

contextTruncationNotice()
// "Some previous conversation history has been removed to maintain optimal
// context window length..."

processFirstUserMessageForTruncation()
// "[Continue assisting the user!]"
```

#### 2. Обработка инструментов

```typescript
toolDenied()
// "The user denied this operation."

toolError(error?: string)
// "The tool execution failed with the following error:
// <error>${error}</error>"

clineIgnoreError(path: string)
// "Access to ${path} is blocked by the .clineignore file settings..."
```

#### 3. Ошибки использования

```typescript
noToolsUsed(usingNativeToolCalls: boolean)
// "[ERROR] You did not use a tool in your previous response!..."

tooManyMistakes(feedback?: string)
// "You seem to be having trouble proceeding. The user has provided..."

missingToolParameterError(paramName: string)
// "Missing value for required parameter '${paramName}'..."
```

#### 4. Редактирование файлов

```typescript
// Детальная обратная связь при сохранении файлов:
- Предупреждения об изменениях auto-formatting
- Инструкции использовать финальное содержимое как reference
```

#### 5. Интеграция правил

```typescript
// Форматирование инструкций из:
- .clinerules
- .clineignore
- .cursorrules
- AGENTS.md
```

#### 6. Обработка изображений

```typescript
formatImagesIntoBlocks()
// Конвертирует base64-encoded изображения в формат Anthropic
```

#### 7. Другие утилиты

```typescript
formatFilesList()
// Сортирует и отображает файлы, помечает игнорируемые lock символом

createPrettyPatch()
// Генерирует читаемые diffs между версиями файлов

taskResumption()
// Переориентирует агента при возобновлении прерванной работы

planModeInstructions()
// Руководство для planning mode vs act mode
```

---

## Шаблоны и плейсхолдеры

### Template Engine

**Файл**: `src/core/prompts/system-prompt/templates/TemplateEngine.ts`

**Основные методы**:

```typescript
class TemplateEngine {
  // Заменяет {{PLACEHOLDER}} паттерны значениями из контекста
  resolve(template, context): string

  // Проверяет наличие всех требуемых плейсхолдеров
  validate(template, requiredPlaceholders): string[]

  // Извлекает все уникальные плейсхолдеры из шаблона
  extractPlaceholders(template): string[]

  // Получает вложенные значения через dot-notation
  private getNestedValue(obj, path): any

  // Экранирование/деэкранирование плейсхолдеров
  escape(template): string
  unescape(template): string
}
```

**Возможности**:
- Поддержка вложенных свойств через dot notation (`user.name`)
- Обработка function-type templates
- Partial resolution (нерешенные плейсхолдеры остаются)

---

### Плейсхолдеры

**Файл**: `src/core/prompts/system-prompt/templates/placeholders.ts`

#### Стандартные плейсхолдеры:

**Системная информация**:
- `{{OS}}` - Операционная система
- `{{SHELL}}` - Shell по умолчанию
- `{{HOME_DIR}}` - Домашняя директория
- `{{WORKING_DIR}}` - Рабочая директория
- `{{CWD}}` - Current working directory

**MCP конфигурация**:
- `{{MCP_SERVERS_LIST}}` - Список MCP серверов

**Контекстные переменные**:
- `{{SUPPORTS_BROWSER}}` - Поддержка браузера
- `{{MODEL_FAMILY}}` - Семейство модели
- `{{CURRENT_DATE}}` - Текущая дата

**Секции системного промпта**:
- `{{AGENT_ROLE}}`
- `{{TOOL_USE}}`
- `{{TOOLS}}`
- `{{MCP}}`
- `{{EDITING_FILES}}`
- `{{ACT_VS_PLAN}}`
- `{{CLI_SUBAGENTS}}`
- `{{TODO}}`
- `{{CAPABILITIES}}`
- `{{RULES}}`
- `{{SYSTEM_INFO}}`
- `{{OBJECTIVE}}`
- `{{USER_INSTRUCTIONS}}`
- `{{FEEDBACK}}`
- `{{TASK_PROGRESS}}`

#### Обязательные vs опциональные:

**Обязательные** (required for basic prompt):
- `AGENT_ROLE`
- `SYSTEM_INFO`

**Опциональные** (improve functionality):
- `FEEDBACK`
- `USER_INSTRUCTIONS`
- `TODO`

---

## Сборка промптов

### PromptBuilder

**Файл**: `src/core/prompts/system-prompt/registry/PromptBuilder.ts`

**Процесс сборки**:

```
1. Component Building
   ↓
   Выполняет зарегистрированные component функции последовательно
   Перехватывает результаты если содержат контент
   Логирует warnings при failures без остановки

2. Placeholder Preparation
   ↓
   Объединяет:
   - Variant-defined placeholders (lowest priority)
   - Standard system placeholders (CWD, browser, model family, date)
   - Component sections
   - Runtime placeholders (highest priority, override others)

3. Template Resolution
   ↓
   Подставляет плейсхолдеры в base template

4. Post-Processing
   ↓
   Нормализует whitespace и форматирование
   Удаляет множественные пустые строки
   Очищает malformed section headers
   Exceptions для diff-like content (SEARCH, REPLACE markers)
```

**Порядок компонентов**:
Определяется `variant.componentOrder`

**Интеграция инструментов**:

```typescript
static getToolsPrompts(variantId, toolIds, context)
// Процесс:
- Resolve tool IDs through ClineToolSet с fallback
- Фильтрация по контекстным требованиям
- Сохранение запрошенного порядка
- Генерация formatted tool descriptions с parameters и примерами
```

**Metadata**:
Build metadata включает:
- Variant identity
- Component usage
- Resolved placeholder names

---

## Registry System

### PromptRegistry

**Файл**: `src/core/prompts/system-prompt/registry/PromptRegistry.ts`

**Класс**: Singleton управляющий system prompts и их вариантами

**Основные возможности**:

```typescript
class PromptRegistry {
  // Получает matched prompt с native tools
  get(context): { prompt, nativeTools }

  // Доступ к специфичной версии
  getVersion(modelId, version): PromptVariant

  // Поиск по semantic labels
  getByTag(tag): PromptVariant[]

  // Список зарегистрированных моделей
  getAvailableModels(): string[]

  // Добавляет переиспользуемый компонент
  registerComponent(name, component): void
}
```

**Variant Matching стратегии**:
1. Model family detection через matcher functions
2. Fallback к generic variant если нет совпадения
3. Version-specific retrieval через `modelId@version` keys
4. Tag и label-based lookup

**Загрузка**:
- Варианты загружаются из variants directory configuration
- Safety fallback: создает minimal generic variant если загрузка fails
- Health check при инициализации

**Error Handling**:
- Detailed error reporting
- Available variants listing
- Counts и loaded state при lookup failures
- Предотвращение silent failures

---

## MCP Документация

**Файл**: `src/core/prompts/loadMcpDocumentation.ts`

**Функция**: `loadMcpDocumentation()`

**Назначение**: Comprehensive руководство по созданию MCP серверов

**Ключевые секции**:

### Creating MCP Servers

```
MCP servers operate in non-interactive environments and cannot:
- Initiate OAuth flows
- Prompt for user input

All credentials must be provided upfront through environment variables.
```

### Technical Implementation

**Пример**: Weather server на TypeScript

```
Steps:
1. Bootstrap project: create-typescript-server
2. Install dependencies: axios
3. Structure server:
   - Resource handlers
   - Tool handlers

Note: "Tools are more flexible and can handle dynamic parameters" vs resources
```

### Configuration

```
Servers register in MCP settings file with:
- Command paths
- Arguments
- Environment variables

New servers should default to:
- disabled=false
- autoApprove=[]
```

### Important Notes

```
- Build output → build/ or dist/ directories
- Systems automatically expose connected server tools/resources
- MCP servers aren't always necessary for requests
- Users should explicitly request MCP server creation
```

---

## Validation и Типы

### Типы

**Файл**: `src/core/prompts/system-prompt/types.ts`

**Основные интерфейсы**:

#### PromptVariant
```typescript
interface PromptVariant {
  readonly id: string
  readonly version: number
  readonly modelFamily: ModelFamily
  readonly matcher: (providerInfo) => boolean
  // ... другие свойства
}
```

#### SystemPromptContext
```typescript
interface SystemPromptContext {
  readonly providerInfo: ProviderInfo
  readonly ide: IDEType
  readonly supportsBrowser: boolean
  readonly mcpHub: McpHubInstance
  readonly ruleFileInstructions: RuleFileInstructions
  // ... другие свойства
}
```

#### ConfigOverride
```typescript
interface ConfigOverride {
  components?: {
    [key: string]: {
      template?: Function
      enabled?: boolean
      order?: number
    }
  }
  tools?: {
    [key: string]: {
      enabled?: boolean
      order?: number
    }
  }
}
```

#### ValidationResult
```typescript
interface ValidationResult {
  valid: boolean
  errors: ValidationError[]
  warnings: ValidationWarning[]
}
```

#### VariantBuilder
```typescript
interface VariantBuilder {
  description(text: string): this
  version(v: number): this
  tags(...tags: string[]): this
  template(base: string): this
  build(): PromptVariant
}
```

---

## Заключение

Система промптов Cline представляет собой **сложную модульную архитектуру** с:

### Ключевые особенности:

1. **Модульность**
   - Переиспользуемые компоненты
   - Композиция через builder pattern
   - Dependency injection через context

2. **Адаптивность**
   - Варианты для разных моделей
   - Контекстная фильтрация
   - Graceful degradation

3. **Расширяемость**
   - Plugin система для компонентов
   - MCP integration
   - Custom tool definitions

4. **Надежность**
   - Compile-time validation
   - Runtime error handling
   - Fallback mechanisms

5. **Оптимизация**
   - Model-specific prompts
   - Compact варианты для малого контекста
   - Efficient template resolution

### Workflow:

```
User Request
    ↓
Context Analysis → Variant Selection
    ↓
Component Assembly → Placeholder Resolution
    ↓
Template Rendering → Post-Processing
    ↓
Final System Prompt → AI Model
    ↓
Tool Execution → Response Formatting
    ↓
User Output
```

### Файловая статистика:

- **Компоненты**: ~14 файлов
- **Варианты**: ~9 директорий
- **Инструменты**: ~17+ определений
- **Утилиты**: ~10+ helper functions
- **Общий размер**: ~11,747 tokens (базовый промпт)

### Основные промпты по категориям:

**Identity & Role**:
- Agent Role (Cline as skilled software engineer)

**Capabilities**:
- Tool descriptions
- File operations
- Command execution
- Browser automation
- MCP integration

**Workflow**:
- Objective (iterative task completion)
- Act vs Plan modes
- Task progress tracking
- CLI subagents

**Guidelines**:
- Tool use guidelines
- File editing best practices
- Operational rules
- Communication style

**Context Management**:
- Task summarization
- Continuation prompts
- Multi-root workspace support

**User Customization**:
- Rule file integration
- Custom instructions
- Feedback mechanisms

Эта система демонстрирует **enterprise-grade подход** к prompt engineering с акцентом на:
- Maintainability (поддерживаемость)
- Scalability (масштабируемость)
- Reliability (надежность)
- Performance (производительность)

---

## Дополнительные ресурсы

- **Репозиторий**: https://github.com/cline/cline
- **Документация**: https://docs.cline.bot
- **Промпты**: `/src/core/prompts/`
- **System Prompt**: `/src/core/prompts/system-prompt/`
- **Варианты**: `/src/core/prompts/system-prompt/variants/`
- **Инструменты**: `/src/core/prompts/system-prompt/registry/tools/`

---

*Анализ выполнен: 2025-11-13*
*Версия репозитория: main branch*
