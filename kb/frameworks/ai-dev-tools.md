# Модуль 38: Claude Code та Cursor — AI-assisted розробка LLM-застосунків

## 🎯 Що ви отримаєте з цього модуля

Після проходження ви будете:
- Ефективно використовувати Claude Code для розробки AI-застосунків
- Налаштовувати Cursor для максимальної продуктивності
- Знати коли AI-assisted розробка допомагає, а коли шкодить
- Використовувати MCP-сервери для розширення можливостей IDE

**Які задачі це дозволяє вирішувати:** Розробляти LLM-застосунки у 2-5x швидше. Використовувати AI для написання AI-коду — meta-рівень де знання цього курсу дають найбільшу перевагу.

---

## 38.1 Claude Code — Agentic Coding в терміналі

### Що це

Claude Code — CLI-інструмент від Anthropic. Працює в терміналі, має доступ до файлової системи, bash, і може виконувати команди.

```bash
# Встановлення
npm install -g @anthropic-ai/claude-code

# Запуск у проекті
cd my-ai-project
claude
```

### Коли Claude Code найефективніший для AI-розробки

**Створення нових функцій:** "Додай RAG pipeline що індексує Markdown файли з папки docs/ і шукає по ним через pgvector."

**Рефакторинг:** "Перепиши всі прямі виклики OpenAI API на використання AI SDK з фолбеком між провайдерами."

**Debugging:** "Агент зациклюється на tool calls і не дає фінальну відповідь. Подивись логи і знайди проблему."

**Тестування:** "Створи eval suite з 10 тест-кейсів для промпту класифікації тікетів."

### CLAUDE.md — контекст проекту

Створіть файл `CLAUDE.md` у корені проекту щоб Claude Code розумів контекст:

```markdown
# Project: AI Customer Support Platform

## Stack
- TypeScript 5.x, Node.js 22
- Vercel AI SDK 6
- OpenAI GPT-4o-mini (основна модель)
- Anthropic Claude Sonnet (для складних запитів)
- PostgreSQL + pgvector для RAG
- Redis для rate limiting та кешу

## Architecture
- /src/agents/ — AI агенти
- /src/tools/ — Tool definitions для agents
- /src/rag/ — RAG pipeline (indexing, retrieval)
- /src/prompts/ — System prompts (версіоновані)
- /eval/ — Evaluation test cases

## Conventions
- Всі промпти українською
- Zod для всіх schema
- Обробка помилок: повертай { error } з tools, не throw
- temperature: 0 для класифікації, 0.7 для генерації
```

---

## 38.2 Cursor — AI-first IDE

### Налаштування для AI-розробки

```json
// .cursor/settings.json
{
  "cursor.ai.model": "claude-sonnet-4-5",
  "cursor.ai.contextLength": "long",
  "cursor.rules": [
    "Use Vercel AI SDK 6 for all LLM interactions",
    "Use Zod for all schemas",
    "Handle errors in tools by returning { error } objects",
    "Write TypeScript, not JavaScript"
  ]
}
```

### .cursorrules — правила для AI

```
# .cursorrules

You are working on an AI/LLM application built with TypeScript and Vercel AI SDK 6.

## Key Patterns
- Use `generateText` for simple completions
- Use `generateObject` with Zod schemas for structured output
- Use `streamText` for streaming responses
- Use `tool()` with description, parameters (Zod), and execute function
- Always set `maxSteps` when using tools (default: 5)

## Error Handling
- Tools should never throw errors
- Return `{ error: string }` from tool execute functions
- Use try-catch around all LLM calls
- Implement fallback chain: Claude → GPT → Gemini

## Prompts
- System prompts in Ukrainian
- Keep prompts concise but specific
- Use temperature: 0 for classification, 0.3 for extraction, 0.7 for generation
```

---

## 38.3 MCP-сервери для IDE

Підключіть MCP-сервери щоб AI в IDE мав доступ до ваших даних:

```json
// .cursor/mcp.json або claude_desktop_config.json
{
  "mcpServers": {
    "postgres": {
      "command": "npx",
      "args": ["-y", "@modelcontextprotocol/server-postgres", "postgresql://localhost:5432/mydb"]
    },
    "github": {
      "command": "npx",
      "args": ["-y", "@modelcontextprotocol/server-github"],
      "env": { "GITHUB_TOKEN": "..." }
    }
  }
}
```

Тепер можна сказати: "Подивись структуру таблиці customers і створи tool для пошуку клієнтів."

---

## 38.4 Коли AI-assisted розробка шкодить

**AI добре працює для:** boilerplate коду, типові CRUD операції, написання тестів по існуючому коду, рефакторинг з чіткими правилами, генерація documentation.

**AI погано працює для:** архітектурні рішення (може запропонувати over-engineering), security-critical код (може пропустити вразливість), оптимізація performance (не розуміє runtime), нетривіальна бізнес-логіка (не знає контекст).

**Правило:** AI пише код, ви робите code review. Ніколи не мерджте AI-generated код без перевірки.

---

## 38.5 Productivity Tips

**Будьте конкретними:** "Додай generateObject з Zod schema для класифікації тікетів на 5 категорій" краще ніж "Зроби класифікацію."

**Давайте контекст:** Відкрийте релевантні файли перед запитом. AI бачить відкриті файли.

**Ітеруйте:** Починайте з простого → тестуйте → ускладнюйте.

**Використовуйте eval:** Після генерації промпту — одразу створіть тест-кейси.

---

## Перевір себе

1. Що таке CLAUDE.md і як він допомагає?
2. Як .cursorrules покращує якість AI-generated коду?
3. Які MCP-сервери корисні для AI-розробки?
4. Коли AI-assisted розробка шкодить якості коду?
5. Напишіть CLAUDE.md для свого AI-проекту

---

**Назад:** [← Модуль 37 — Testing patterns](37-testing-patterns.md) | **Далі:** [Модуль 39 — Prompt management platforms →](39-prompt-management.md)
