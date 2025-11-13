# Cline AI Prompts Documentation

> Полная документация всех промптов, используемых в AI-агенте Cline

**Дата создания:** 2025-11-13
**Источник:** [github.com/cline/cline](https://github.com/cline/cline)
**Версия:** main branch

---

## Оглавление

1. [Введение](#введение)
2. [Архитектура системы промптов](#архитектура)
3. [Identity & Role Prompts](#identity--role-prompts)
4. [Capabilities Prompts](#capabilities-prompts)
5. [Workflow Prompts](#workflow-prompts)
6. [File Operations Prompts](#file-operations-prompts)
7. [Tool Use Prompts](#tool-use-prompts)
8. [Rules & Guidelines Prompts](#rules--guidelines-prompts)
9. [Context Management Prompts](#context-management-prompts)
10. [Special Commands Prompts](#special-commands-prompts)
11. [MCP Integration Prompts](#mcp-integration-prompts)
12. [CLI Subagents Prompts](#cli-subagents-prompts)
13. [Tools Specifications](#tools-specifications)
14. [Prompt Variants](#prompt-variants)

---

## Введение

Cline - это автономный AI-ассистент для программирования, работающий внутри VS Code. Система промптов Cline построена на модульной архитектуре, где каждый промпт имеет конкретное назначение и может быть адаптирован под различные AI модели.

### Основные принципы

- **Модульность**: Промпты собираются из переиспользуемых компонентов
- **Адаптивность**: Различные варианты для разных моделей AI
- **Контекстность**: Промпты адаптируются под текущую ситуацию
- **Расширяемость**: Легко добавлять новые компоненты и инструменты

### Статистика

- **Компонентов**: ~14
- **Вариантов**: 9+
- **Инструментов**: 17+
- **Размер базового промпта**: ~11,747 tokens

---

## Архитектура

### Структура файлов

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
    │   └── ...                   # Другие варианты
    ├── spec.ts                    # Спецификации инструментов
    ├── types.ts                   # Типы TypeScript
    └── index.ts                   # Главный экспорт
```

### Процесс сборки промпта

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
```

---

## Identity & Role Prompts

### 1. AGENT_ROLE - Основная роль агента

**Файл:** `src/core/prompts/system-prompt/components/agent_role.ts`

**Назначение:** Определяет базовую идентичность и роль AI-ассистента Cline. Этот промпт устанавливает основу для всех последующих взаимодействий, определяя, кем является агент и какими навыками он обладает.

**Когда используется:**
- В начале каждого системного промпта
- При инициализации новой сессии
- Может быть переопределен в специфических вариантах для разных моделей

**Особенности:**
- Обрабатывается через TemplateEngine для подстановки динамических значений
- Может быть кастомизирован через variant overrides
- Является обязательным компонентом (required) для всех вариантов промптов

**Промпт:**

```
You are Cline, a highly skilled software engineer with extensive knowledge in many programming languages, frameworks, design patterns, and best practices.
```

**Контекст применения:**

Этот промпт создает профессиональную идентичность агента как:
- Опытного инженера-программиста
- Эксперта в множестве языков программирования
- Специалиста по фреймворкам и паттернам проектирования
- Знатока лучших практик разработки

**Связанные компоненты:**
- CAPABILITIES - расширяет роль описанием конкретных возможностей
- OBJECTIVE - определяет подход к выполнению задач
- RULES - устанавливает операционные ограничения

---

## Capabilities Prompts

### 2. CAPABILITIES - Возможности агента

**Файл:** `src/core/prompts/system-prompt/components/capabilities.ts`

**Назначение:** Описывает все доступные агенту возможности и инструменты. Этот промпт информирует AI о том, что он может делать и какие инструменты доступны для выполнения задач.

**Когда используется:**
- В составе каждого системного промпта
- Перед началом выполнения задач, чтобы агент знал свои возможности
- Может быть расширен дополнительными возможностями через MCP серверы

**Промпт:**

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

**Контекст применения:**

Этот промпт создает понимание у агента о доступных возможностях:

1. **Файловые операции:**
   - Чтение и запись файлов
   - Рекурсивный просмотр директорий
   - Поиск по регулярным выражениям
   - Просмотр определений кода

2. **Выполнение команд:**
   - CLI команды в терминале VS Code
   - Поддержка интерактивных команд
   - Длительные операции

3. **Браузерная автоматизация:**
   - Навигация по веб-страницам
   - Взаимодействие с элементами
   - Создание скриншотов
   - Анализ console logs

4. **Расширяемость:**
   - Интеграция с MCP серверами для дополнительных возможностей

**Особенности:**
- Динамически включает секции в зависимости от доступных возможностей
- Browser support включается только если browser_action tool доступен
- MCP capabilities добавляются динамически из подключенных серверов

**Связанные компоненты:**
- TOOL_USE - детальное описание использования инструментов
- MCP - интеграция с Model Context Protocol серверами
- RULES - ограничения на использование возможностей

---

## Workflow Prompts

### 3. OBJECTIVE - Цели и подход к задачам

**Файл:** `src/core/prompts/system-prompt/components/objective.ts`

**Назначение:** Определяет методологию выполнения задач агентом - итеративный подход с разбиением на шаги.

**Промпт:**

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

**Особенности:**
- Включает логику "yolo mode" для автоматического вывода параметров
- Акцент на итеративном подходе без лишних диалогов
- Определяет 5-шаговый процесс выполнения задач

---

### 4. ACT_VS_PLAN_MODE - Режимы работы

**Файл:** `src/core/prompts/system-prompt/components/act_vs_plan_mode.ts`

**Назначение:** Объясняет два режима работы агента - планирование и выполнение.

**Промпт:**

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

**Когда используется:**
- **Plan Mode**: При старте новой задачи для обсуждения плана
- **Act Mode**: После утверждения плана или в режиме YOLO

**Особенности:**
- Поддержка двух режимов работы
- Условная логика для YOLO mode
- Кастомизация через template overrides

---

### 5. TASK_PROGRESS - Отслеживание прогресса

**Файл:** `src/core/prompts/system-prompt/components/task_progress.ts`

**Назначение:** Инструкции по отслеживанию прогресса многоэтапных задач через чеклисты.

**Промпт (стандартный вариант):**

```
Track progress on multi-step tasks using task_progress parameter with markdown checklists:
- [ ] for incomplete items
- [x] for completed items
```

**Промпт (Native Next-Gen вариант):**

```
The task_progress must be included as a separate parameter in the tool, it should not be
included inside other content or argument blocks.
```

**Ключевые рекомендации:**
- Создавать comprehensive todo списки при переходе в action mode
- Обновлять чеклисты без уведомления пользователя
- Фокусироваться на значимых вехах, а не на технических деталях
- Переписывать чеклисты при изменении scope
- Отмечать пункты как выполненные немедленно

**Варианты:**
- **Стандартный**: Для большинства моделей
- **Native Next-Gen**: Для продвинутых моделей с нативными tool calls
- **Native GPT-5**: Специальный вариант для GPT-5

---

## File Operations Prompts

### 6. EDITING_FILES - Редактирование файлов

**Файл:** `src/core/prompts/system-prompt/components/editing_files.ts`

**Назначение:** Руководство по правильному редактированию файлов, выбору между write_to_file и replace_in_file.

**Промпт:**

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

**Ключевые принципы:**
- **replace_in_file** - основной инструмент для редактирования
- **write_to_file** - только для новых файлов или полной замены
- Учет автоформатирования кода
- Множественные правки в одном вызове

---

## Tool Use Prompts

### 7. TOOL_USE - Использование инструментов

**Файл:** `src/core/prompts/system-prompt/components/tool_use/`

#### 7.1 Guidelines - Руководства

**Файл:** `guidelines.ts`

**Промпт:**

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

**Файл:** `formatting.ts`

**Промпт:**

```
Tool calls use XML-style tag formatting with parameters enclosed in nested tags.

Example:
<read_file>
<path>/path/to/file.txt</path>
</read_file>

[When focus chain is enabled, additional section for task progress checklist formatting]
```

#### 7.3 Examples - Примеры использования

**Файл:** `examples.ts`

**Содержит примеры:**
1. **Command execution** - `<execute_command>` синтаксис для запуска npm скриптов
2. **File creation** - `<write_to_file>` с JSON конфигурацией
3. **Task creation** - `<new_task>` структура с секциями контекста
4. **File editing** - `<replace_in_file>` с diff-style синтаксисом
5. **MCP tool usage** - два примера формата `<use_mcp_tool>`

---

## Rules & Guidelines Prompts

### 8. RULES - Операционные правила

**Файл:** `src/core/prompts/system-prompt/components/rules.ts`

**Назначение:** Устанавливает операционные ограничения и правила поведения агента.

**Промпт:**

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

## Context Management Prompts

### 9. Context Management - Управление контекстом

**Файл:** `src/core/prompts/contextManagement.ts`

#### 9.1 summarizeTask - Сводка задачи

**Назначение:** Создание детальной сводки беседы при приближении к лимиту контекста.

**Промпт:**

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

#### 9.2 continuationPrompt - Продолжение

**Назначение:** Возобновление беседы из предыдущей сводки.

**Промпт:**

```
Continue from where work was suspended.

INSTRUCTIONS:
- Continue work based on summary
- Ignore special commands (/newtask, /compact, /smol) unless explicitly repeated by user
- Do not ask clarification questions about the summary
- Focus on the most recent user message
```

---

## Special Commands Prompts

### 10. Special Commands - Специальные команды

**Файл:** `src/core/prompts/commands.ts`

#### 10.1 new_task - Новая задача

**Функция:** `newTaskToolResponse()`

**Промпт:**

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

#### 10.2 condense - Сжатие контекста

**Функция:** `condenseToolResponse()`

**Назначение:** Создание сводки беседы для сжатия контекста.

#### 10.3 new_rule - Новое правило

**Функция:** `newRuleToolResponse()`

**Назначение:** Создание Cline rule файлов в .clinerules/ директории.

#### 10.4 report_bug - Отчет об ошибке

**Функция:** `reportBugToolResponse()`

**Назначение:** Подача GitHub issues с описанием проблемы.

#### 10.5 subagent - Подагент

**Функция:** `subagentToolResponse()`

**Назначение:** Вызов Cline CLI подагентов для исследовательских задач.

---

## MCP Integration Prompts

### 11. MCP - Model Context Protocol

**Файл:** `src/core/prompts/system-prompt/components/mcp.ts`

**Назначение:** Интеграция с MCP серверами для расширения возможностей агента.

**Промпт:**

```
The Model Context Protocol (MCP) enables communication between the system and locally
running MCP servers that provide additional tools and resources to extend your capabilities.

[Динамически генерируемая документация для подключенных MCP серверов, включающая:]
- Server name and execution command
- Available tools with input schemas
- Resource templates with URI patterns
- Direct resources with descriptions
```

**Структура документации:**
- Каждый сервер представлен заголовком с командой
- Инструменты с JSON схемами входных данных
- Шаблоны ресурсов и прямые ресурсы отдельно
- Последовательное форматирование для парсинга

**MCP Server Creation Guide:**

**Файл:** `src/core/prompts/loadMcpDocumentation.ts`

```
MCP servers operate in non-interactive environments and cannot:
- Initiate OAuth flows
- Prompt for user input

All credentials must be provided upfront through environment variables.

Technical Implementation:
1. Bootstrap project: create-typescript-server
2. Install dependencies
3. Structure server with resource and tool handlers

Note: "Tools are more flexible and can handle dynamic parameters" vs resources

Configuration:
- Servers register in MCP settings file
- New servers default to: disabled=false, autoApprove=[]
```

---

## CLI Subagents Prompts

### 12. CLI_SUBAGENTS - CLI подагенты

**Файл:** `src/core/prompts/system-prompt/components/cli_subagents.ts`

**Назначение:** Инструкции по использованию CLI подагентов для исследовательских задач.

**Промпт:**

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

**Условия включения:**
- CLI установлен
- Подагенты включены
- Текущий контекст не является CLI подагентом (предотвращение рекурсии)

---

## Tools Specifications

### 13. Available Tools - Доступные инструменты

**Файл:** `src/core/prompts/system-prompt/spec.ts` и `registry/ClineToolSet.ts`

#### Список основных инструментов

1. **execute_command** / **bash**
   - Выполнение CLI команд в терминале VS Code
   - Поддержка интерактивных и long-running команд

2. **read_file**
   - Чтение содержимого файлов
   - Поддержка различных форматов

3. **write_to_file**
   - Создание новых файлов
   - Полная замена содержимого существующих файлов

4. **replace_in_file**
   - Точечное редактирование файлов
   - Diff-style SEARCH/REPLACE блоки

5. **list_files**
   - Рекурсивный просмотр директорий
   - Получение структуры проекта

6. **search_files**
   - Поиск по регулярным выражениям
   - Контекстные результаты поиска

7. **list_code_definition_names**
   - Обзор определений кода
   - Анализ структуры файлов

8. **browser_action** (условно)
   - Автоматизация браузера через Puppeteer
   - Навигация, взаимодействие, скриншоты

9. **fetch**
   - Загрузка данных из URL
   - HTTP запросы

10. **use_mcp_tool**
    - Использование инструментов MCP серверов
    - Динамические параметры

11. **access_mcp_resource**
    - Доступ к ресурсам MCP серверов
    - URI-based доступ

12. **ask_followup_question**
    - Задать уточняющий вопрос пользователю
    - Получение дополнительной информации

13. **attempt_completion**
    - Завершение задачи
    - Предоставление результата

14. **new_task**
    - Создание новой задачи с контекстом
    - Сохранение текущего прогресса

15. **plan_mode_respond**
    - Ответ в режиме планирования
    - Представление плана пользователю

16. **ask_docs**
    - Обращение к документации
    - Поиск справочной информации

17. **update_task_todo** (Next-Gen)
    - Обновление todo списка задачи
    - Отслеживание прогресса

#### Tool Specification Interface

```typescript
interface ClineToolSpec {
  variant: string                          // Вариант промпта
  id: string                               // Уникальный ID
  name: string                             // Имя инструмента
  description: string                      // Описание назначения
  instructions?: string                    // Дополнительные инструкции
  includeIf?: (context) => boolean        // Условие включения
  parameters: ClineToolSpecParameter[]     // Параметры инструмента
}

interface ClineToolSpecParameter {
  name: string                             // Имя параметра
  type: "string" | "boolean" | "integer" | "array" | "object"
  description: string                      // Описание параметра
  required: boolean                        // Обязательный ли
  includeIf?: (context) => boolean        // Условие включения
  dependsOn?: string                       // Зависимость от другого параметра
}
```

#### Конвертация для разных провайдеров

- **OpenAI**: `toolSpecFunctionDefinition()` - ChatCompletionTool формат
- **Anthropic**: `toolSpecInputSchema()` - Tool definitions
- **Google Gemini**: `toolSpecFunctionDeclarations()` - Function declarations
- **MCP Tools**: `mcpToolToClineToolSpec()` - Конвертация MCP в Cline формат

#### Tool Set Management

Класс `ClineToolSet` управляет инструментами:

```typescript
class ClineToolSet {
  // Регистрация инструмента
  register(variantId, tool): void

  // Получение инструментов с fallback
  getTools(variantId): ClineToolSpec[]

  // Поиск инструмента по имени
  getToolByName(variantId, toolName): ClineToolSpec | undefined

  // Фильтрация по контексту
  getEnabledTools(variantId, context): ClineToolSpec[]

  // Конвертация для провайдеров
  getNativeTools(variant, context): NativeToolDefinition[]
}
```

**Иерархия fallback:**
1. Точный вариант
2. GENERIC вариант
3. Любой доступный вариант

---

## Prompt Variants

### 14. Варианты промптов для разных моделей

**Файл:** `src/core/prompts/system-prompt/variants/`

#### 14.1 GENERIC - Базовый вариант

**Файл:** `variants/generic/config.ts`

**Характеристики:**
- **Model Family**: GENERIC
- **Version**: 1
- **Назначение**: Fallback конфигурация для любых моделей

**Matcher логика:**
```typescript
// Применяется к моделям, которые:
- Не имеют provider или model ID информации, ИЛИ
- Не являются локальными моделями с compact prompts, ИЛИ
- Не являются next-generation моделями, ИЛИ
- Не являются GLM-family моделями
```

**Секции промпта (13):**
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

**Инструменты (16):**
bash, read_file, write_to_file, replace_in_file, list_files, search_files,
list_code_definition_names, browser_action, fetch, use_mcp_tool, access_mcp_resource,
ask_followup_question, attempt_completion, new_task, plan_mode_respond, ask_docs

---

#### 14.2 NEXT_GEN - Продвинутые модели

**Файл:** `variants/next-gen/config.ts`

**Характеристики:**
- **Model Family**: NEXT_GEN
- **Version**: 1
- **Tags**: "next-gen", "advanced", "production"
- **Назначение**: Оптимизация для frontier моделей с enhanced agentic capabilities

**Matcher логика:**
```typescript
// Применяется к:
- Next-gen model family без native tool calls
- Исключает compact prompts на локальных моделях
- Исключает next-gen providers с определенными условиями
- Избегает GPT-5 non-chat вариантов
```

**Секции промпта (13):**
Те же, что у GENERIC

**Инструменты (17):**
Все из GENERIC + `update_task_todo` - обновление todo списка

**Особенности:**
- Sophisticated multi-tool orchestration
- Suited for complex autonomous reasoning tasks
- Compile-time validation с strict mode

---

#### 14.3 XS - Компактный вариант

**Файл:** `variants/xs/config.ts`

**Характеристики:**
- **Model Family**: XS
- **Version**: 1
- **Tags**: "local", "xs", "compact"
- **Назначение**: Для моделей с малым контекстом

**Matcher логика:**
```typescript
// Применяется к:
- Custom prompt set = "compact"
- AND local model provider
```

**Секции промпта (9) - сокращенный набор:**
1. Agent role
2. Rules
3. Act vs plan
4. CLI subagents
5. Capabilities
6. Editing files
7. Objective
8. System info
9. User instructions

**Инструменты (10) - минимальный набор:**
bash, read_file, write_to_file, replace_in_file, search_files, list_files,
ask_followup_question, attempt_completion, new_task, plan_mode_respond

**Особенности:**
- Compact footprint для малого контекста
- Compile-time validation
- Component overrides применяются post-build

---

#### 14.4 Другие варианты

**Доступные варианты:**
- **gpt-5/** - Специфичный для GPT-5
- **native-gpt-5/** - Native tool calls для GPT-5
- **native-gpt-5-1/** - Версия 1 native GPT-5
- **native-next-gen/** - Native tool calls для next-gen
- **glm/** - Для GLM моделей
- **hermes/** - Для Hermes моделей

#### Variant Registry

**Файл:** `registry/PromptRegistry.ts`

Singleton управляющий system prompts:

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

**Variant Matching стратегии:**
1. Model family detection через matcher functions
2. Fallback к generic variant если нет совпадения
3. Version-specific retrieval через `modelId@version` keys
4. Tag и label-based lookup

---

## Заключение

### Ключевые особенности системы промптов Cline:

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

### Workflow процесс:

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

### Статистика:

- **Компоненты**: ~14 файлов
- **Варианты**: 9+ директорий
- **Инструменты**: 17+ определений
- **Утилиты**: ~10+ helper functions
- **Общий размер базового промпта**: ~11,747 tokens

---

## Дополнительные ресурсы

- **Репозиторий**: [github.com/cline/cline](https://github.com/cline/cline)
- **Документация**: [docs.cline.bot](https://docs.cline.bot)
- **Промпты**: `/src/core/prompts/`
- **System Prompt**: `/src/core/prompts/system-prompt/`
- **Варианты**: `/src/core/prompts/system-prompt/variants/`
- **Инструменты**: `/src/core/prompts/system-prompt/registry/tools/`

---

*Анализ выполнен: 2025-11-13*
*Версия репозитория: main branch*
*Документация создана на основе исходного кода Cline*

