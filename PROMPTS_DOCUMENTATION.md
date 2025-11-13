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

