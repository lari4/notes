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

