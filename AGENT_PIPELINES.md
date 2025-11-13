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

