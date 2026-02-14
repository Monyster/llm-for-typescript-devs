# Reasoning Models — коли думати довше

> ⏱ ~2 години | 🟡 Middle | Потрібен: [Модуль 02 — Chat API](02-chat-api.md), [Модуль 03 — AI SDK](03-ai-sdk.md)
>
> 📖 Deep dive: [KB: Reasoning Models Reference](../kb/advanced/reasoning-models-reference.md)

## 💰 Бізнес-цінність

**Проблема клієнта:** AI-агент справляється з простими питаннями, але помиляється на складних — аналіз контрактів, дебаг коду, планування.  
**Рішення з цього модуля:** Увімкнути reasoning для складних задач, вимкнути для простих — контролювати якість і вартість.  
**Результат:** Reasoning-моделі показують на 20-40% вищу якість на складних задачах (math, code, analysis) при контрольованому збільшенні вартості.  
**Як продати:** "AI-система з адаптивним reasoning — автоматично визначає складність і витрачає бюджет розумно" — аргумент на $3-5K додаткової вартості проекту.

---

## Що таке reasoning models

Звичайна LLM генерує відповідь токен за токеном, "зліва направо". Reasoning model додає етап **внутрішнього міркування** (thinking) перед відповіддю — щось подібне до того, як людина спочатку обмірковує задачу на чернетці, а потім дає відповідь.

```
Звичайна модель:
  Запит → [генерація] → Відповідь

Reasoning модель:
  Запит → [thinking...thinking...thinking] → Відповідь
              ↑ окремі "reasoning tokens"
              ↑ користувач їх не бачить (або бачить резюме)
              ↑ але вони коштують грошей
```

Reasoning tokens — це реальні токени, які модель генерує. Вони входять у вартість output-токенів, але зазвичай не показуються кінцевому користувачу.

### Хто це підтримує (2026)

| Провайдер | Реалізація | Параметр |
|-----------|-----------|---------|
| **Anthropic** | Extended Thinking (Claude Sonnet 4.5, Opus 4.6) | `thinking: { type: 'adaptive' }` або `{ type: 'enabled', budget_tokens: N }` |
| **OpenAI** | Reasoning (o1, o3, o4-mini, GPT-5) | `reasoningEffort: 'low' | 'medium' | 'high'` |
| **Google** | Thinking (Gemini 2.5 Flash/Pro) | `thinkingConfig: { thinkingBudget: N }` |
| **DeepSeek** | Thinking (DeepSeek-R1) | Автоматично, `<think>` теги |
| **xAI** | Reasoning (Grok 4) | Автоматично в reasoning-моделях |

---

## AI SDK: єдиний інтерфейс

AI SDK абстрагує різницю між провайдерами через `providerOptions`:

### Anthropic: Extended Thinking

```typescript
import { generateText } from 'ai';
import { anthropic } from '@ai-sdk/anthropic';

// Adaptive thinking — Claude сам вирішує скільки думати
// Рекомендований режим для Claude Opus 4.6
const { text, reasoning } = await generateText({
  model: anthropic('claude-opus-4-6'),
  prompt: 'Проаналізуй цей контракт на ризики...',
  providerOptions: {
    anthropic: {
      thinking: { type: 'adaptive' },
    },
  },
});

console.log('Reasoning:', reasoning); // Що модель "думала"
console.log('Answer:', text);         // Фінальна відповідь
```

```typescript
// Explicit budget — контролюємо скільки токенів на thinking
// Для Claude Sonnet 4.5 та інших Claude 4 моделей
const { text, reasoning } = await generateText({
  model: anthropic('claude-sonnet-4-5-20250929'),
  prompt: 'Знайди помилку в цьому коді...',
  providerOptions: {
    anthropic: {
      thinking: { type: 'enabled', budget_tokens: 8000 },
    },
  },
  maxTokens: 16000, // Має бути > budget_tokens
});
```

### OpenAI: Reasoning Effort

```typescript
import { openai } from '@ai-sdk/openai';

const { text, reasoning } = await generateText({
  model: openai('o4-mini'),
  prompt: 'Розв\'яжи цю математичну задачу...',
  providerOptions: {
    openai: {
      reasoningEffort: 'high', // 'low' | 'medium' | 'high'
    },
  },
});
```

```typescript
// GPT-5 з reasoning через Responses API
const { text, reasoning } = await generateText({
  model: openai.responses('gpt-5'),
  prompt: 'Спроектуй архітектуру мікросервісів для...',
  providerOptions: {
    openai: {
      reasoningEffort: 'medium',
    },
  },
});
```

### Google: Thinking Budget

```typescript
import { google } from '@ai-sdk/google';

const { text, reasoning } = await generateText({
  model: google('gemini-2.5-flash'),
  prompt: 'Порівняй ці два алгоритми...',
  providerOptions: {
    google: {
      thinkingConfig: { thinkingBudget: 4096 },
    },
  },
});
```

### DeepSeek: Автоматичний reasoning

```typescript
import { createOpenAICompatible } from '@ai-sdk/openai-compatible';

const deepseek = createOpenAICompatible({
  baseURL: 'https://api.deepseek.com/v1',
  name: 'deepseek',
  apiKey: process.env.DEEPSEEK_API_KEY,
});

// DeepSeek-R1 завжди думає — reasoning витягується автоматично
const { text, reasoning } = await generateText({
  model: deepseek('deepseek-reasoner'),
  prompt: 'Доведи що число √2 ірраціональне',
});
```

---

## Коли вмикати reasoning

Reasoning — не безкоштовний. Кожен reasoning-токен коштує як output-токен (або дорожче). Потрібно свідомо вирішувати коли це окупається.

### ✅ Вмикай reasoning (high/enabled)

- **Складний аналіз:** контракти, фінансові документи, медичні виписки
- **Дебаг коду:** пошук складних помилок, аналіз race conditions
- **Математика:** розрахунки, статистика, оптимізація
- **Планування:** архітектурні рішення, проектування агентів
- **Code review:** аналіз pull request, пошук вразливостей
- **Логічні задачі:** де потрібен крок-за-кроком аналіз

### ❌ НЕ вмикай reasoning (off/low)

- **Проста генерація тексту:** відповіді на FAQ, переклад, рерайт
- **Класифікація:** sentiment analysis, категоризація тікетів
- **Витяг даних:** structured output з документів
- **Невеликі чат-відповіді:** привітання, прості питання
- **Потокова обробка:** тисячі однотипних запитів (занадто дорого)

### Правило: "Чи потрібна тут людині чернетка?"

Якщо задача настільки складна що людина-експерт спочатку записала б свої думки на папері — вмикайте reasoning. Якщо людина відповіла б одразу — не вмикайте.

---

## Адаптивний reasoning: змінюємо рівень на льоту

Найпотужніший патерн — автоматично визначати складність і підбирати рівень reasoning:

```typescript
import { generateText } from 'ai';
import { anthropic } from '@ai-sdk/anthropic';
import { openai } from '@ai-sdk/openai';

type Complexity = 'simple' | 'moderate' | 'complex';

// Крок 1: Дешева модель класифікує складність
async function classifyComplexity(query: string): Promise<Complexity> {
  const { text } = await generateText({
    model: openai('gpt-4o-mini'),
    system: `Класифікуй складність запиту. Відповідай ТІЛЬКИ одним словом:
- simple: проста відповідь, факт, FAQ
- moderate: потребує аналізу, порівняння
- complex: складна логіка, дебаг, планування, математика`,
    prompt: query,
  });

  const complexity = text.trim().toLowerCase() as Complexity;
  return ['simple', 'moderate', 'complex'].includes(complexity)
    ? complexity
    : 'moderate';
}

// Крок 2: Обираємо модель і reasoning рівень
async function smartGenerate(query: string) {
  const complexity = await classifyComplexity(query);

  const config = {
    simple: {
      // Дешева модель, без reasoning
      model: openai('gpt-4o-mini'),
      providerOptions: {},
    },
    moderate: {
      // Середня модель, легкий reasoning
      model: anthropic('claude-sonnet-4-5-20250929'),
      providerOptions: {
        anthropic: {
          thinking: { type: 'enabled', budget_tokens: 2048 },
        },
      },
    },
    complex: {
      // Потужна модель, повний reasoning
      model: anthropic('claude-opus-4-6'),
      providerOptions: {
        anthropic: {
          thinking: { type: 'adaptive' },
        },
      },
    },
  }[complexity];

  const { text, reasoning, usage } = await generateText({
    ...config,
    prompt: query,
    maxTokens: 16000,
  });

  console.log(`[${complexity}] Tokens: ${usage?.totalTokens}`);
  return { text, reasoning, complexity };
}
```

**Результат:** Прості запити → $0.001, складні → $0.05. Замість того щоб платити $0.05 за КОЖЕН запит.

---

## Reasoning + Tools (Interleaved Thinking)

Ключовий патерн 2025–2026: модель **думає між tool calls**. Кожен результат інструменту запускає новий цикл reasoning.

```
Запит: "Знайди помилку в production логах і запропонуй фікс"

[thinking] Мені потрібно спочатку подивитись логи...
→ tool_call: fetchLogs({ service: 'api', level: 'error', limit: 50 })
← tool_result: [50 помилок, більшість "Connection timeout to DB"]

[thinking] Бачу багато timeout-ів. Це може бути проблема з пулом з'єднань.
Перевірю конфігурацію...
→ tool_call: readConfig({ service: 'api', file: 'database.ts' })
← tool_result: { pool: { max: 5, idleTimeout: 10000 } }

[thinking] max: 5 замало для production з високим навантаженням.
idleTimeout: 10000 — теж занадто маленький. Рекомендую збільшити до max: 20.
Перевірю ще метрики навантаження...
→ tool_call: fetchMetrics({ service: 'api', metric: 'active_connections' })
← tool_result: { current: 47, peak: 52, avg: 38 }

Відповідь: "Проблема: пул з'єднань до БД обмежений 5, 
а навантаження потребує 38-52. Рішення: збільшити pool.max до 60..."
```

В AI SDK це працює з `maxSteps`:

```typescript
const { text, reasoning, steps } = await generateText({
  model: anthropic('claude-sonnet-4-5-20250929'),
  prompt: 'Знайди помилку в production логах і запропонуй фікс',
  tools: { fetchLogs, readConfig, fetchMetrics },
  maxSteps: 10,
  providerOptions: {
    anthropic: {
      thinking: { type: 'enabled', budget_tokens: 10000 },
    },
  },
});

// reasoning містить ВСІ thinking-блоки, включно з тими між tool calls
console.log('Reasoning steps:', reasoning);
```

Для Claude Opus 4.6 interleaved thinking увімкнений автоматично з `adaptive`. Для Claude 4 моделей потрібен beta-header `interleaved-thinking-2025-05-14`.

---

## Вплив на вартість

Reasoning tokens — це output tokens. Вони коштують стільки ж або дорожче.

```
Приклад: аналіз документу

Без reasoning:
  Input: 2000 токенів ($0.006 при $3/M)
  Output: 500 токенів ($0.0075 при $15/M)
  Разом: ~$0.014

З reasoning (budget 8000):
  Input: 2000 токенів ($0.006)
  Output: 500 + ~6000 reasoning ($0.0975 при $15/M)
  Разом: ~$0.104

Різниця: ~7x дорожче
```

Тому reasoning вмикають **вибірково**, а не для всіх запитів.

### Budget tokens — контроль витрат

```typescript
// Мінімальний budget (1024) — для легкого reasoning
thinking: { type: 'enabled', budget_tokens: 1024 }

// Середній budget — для більшості задач
thinking: { type: 'enabled', budget_tokens: 4096 }

// Великий budget — для дуже складних задач
thinking: { type: 'enabled', budget_tokens: 16000 }

// Adaptive — модель сама вирішує (Claude Opus 4.6)
thinking: { type: 'adaptive' }
```

Anthropic рекомендує: починайте з мінімуму (1024) і збільшуйте інкрементально. `budget_tokens` — це target, не strict limit — фактичне використання може відрізнятись.

---

## Reasoning в agent loop

Найефективніший патерн — різний рівень reasoning для різних етапів агента:

```typescript
import { generateText, tool } from 'ai';
import { anthropic } from '@ai-sdk/anthropic';
import { openai } from '@ai-sdk/openai';

// Планування — потрібен глибокий reasoning
async function planTask(taskDescription: string) {
  const { text } = await generateText({
    model: anthropic('claude-opus-4-6'),
    system: 'Ти — планувальник. Розбий задачу на конкретні кроки.',
    prompt: taskDescription,
    providerOptions: {
      anthropic: { thinking: { type: 'adaptive' } },
    },
  });
  return text;
}

// Виконання окремого кроку — reasoning не потрібен
async function executeStep(step: string, tools: Record<string, any>) {
  const { text, toolCalls } = await generateText({
    model: openai('gpt-4o-mini'), // Дешева, швидка модель
    system: 'Виконай цей крок. Використай інструменти при потребі.',
    prompt: step,
    tools,
    maxSteps: 3,
    // Без reasoning — крок вже конкретний
  });
  return { text, toolCalls };
}

// Фінальний аналіз — потрібен reasoning для синтезу
async function synthesizeResults(results: string[]) {
  const { text } = await generateText({
    model: anthropic('claude-sonnet-4-5-20250929'),
    system: 'Проаналізуй результати і дай фінальну відповідь.',
    prompt: `Результати кроків:\n${results.join('\n---\n')}`,
    providerOptions: {
      anthropic: {
        thinking: { type: 'enabled', budget_tokens: 4096 },
      },
    },
  });
  return text;
}
```

---

## Чеклист: Reasoning для вашого проекту

1. **Де reasoning потрібен?** Виділіть задачі де якість критична: аналіз, планування, дебаг
2. **Де reasoning зайвий?** FAQ, класифікація, витяг даних — тут reasoning = гроші на вітер
3. **Adaptive routing?** Чи варто автоматично класифікувати складність запитів?
4. **Budget control?** Починайте з 1024, збільшуйте інкрементально
5. **Agent loop?** Reasoning на етапі планування, без reasoning на етапі виконання

---

## Що далі

Ви тепер знаєте як і коли використовувати reasoning. Для повного довідника по всіх провайдерах, порівняльних таблиць і production-патернів — [KB: Reasoning Models Reference](../kb/advanced/reasoning-models-reference.md).

Наступний крок в курсі: як обрати правильний фреймворк для вашого AI-агента — [Модуль 12: Agent Frameworks](12-agent-frameworks.md).

---

## Джерела

- [Anthropic — Extended Thinking docs](https://platform.claude.com/docs/en/build-with-claude/extended-thinking)
- [Anthropic — Claude 4 announcement](https://www.anthropic.com/news/claude-4)
- [Vercel — AI SDK Reasoning support](https://vercel.com/blog/ai-sdk-4-2)
- [AI SDK — OpenAI o1 guide](https://sdk.vercel.ai/docs/guides/o1)
