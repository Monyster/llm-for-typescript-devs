# KB: Reasoning Models — Повний довідник

> 📚 Фундамент: [Course — Reasoning Models](../../course/reasoning-models.md)

## Коли це потрібно

- Обираєте reasoning-стратегію для AI-продукту
- Порівнюєте провайдерів по reasoning capabilities
- Оптимізуєте вартість reasoning tokens
- Налаштовуєте interleaved thinking з tools
- Потрібна таблиця "коли яку модель і який рівень reasoning"

---

## Quick Reference

```typescript
// Anthropic — adaptive (рекомендовано для Opus 4.6)
providerOptions: { anthropic: { thinking: { type: 'adaptive' } } }

// Anthropic — explicit budget (Sonnet 4.5, інші Claude 4)
providerOptions: { anthropic: { thinking: { type: 'enabled', budget_tokens: 4096 } } }

// OpenAI — reasoning effort
providerOptions: { openai: { reasoningEffort: 'medium' } }

// Google — thinking budget
providerOptions: { google: { thinkingConfig: { thinkingBudget: 4096 } } }
```

---

## Порівняння провайдерів

### Anthropic: Extended Thinking

**Підтримувані моделі:** Claude Opus 4.6, Opus 4.5, Opus 4.1, Opus 4, Sonnet 4.5, Sonnet 4, Haiku 4.5, Sonnet 3.7

**Три режими:**

| Режим | Параметр | Коли |
|-------|---------|------|
| Вимкнено | `thinking: { type: 'disabled' }` | За замовчуванням, прості задачі |
| Adaptive | `thinking: { type: 'adaptive' }` | Рекомендований для Opus 4.6 — модель сама вирішує |
| Explicit budget | `thinking: { type: 'enabled', budget_tokens: N }` | Контрольований бюджет, мінімум 1024 |

**Interleaved thinking** — модель думає між tool calls:
- Claude Opus 4.6: автоматично з adaptive thinking
- Claude 4 моделі: потрібен header `interleaved-thinking-2025-05-14`
- `budget_tokens` може перевищувати `max_tokens` (бюджет розподіляється по всіх thinking-блоках в одному assistant turn)

**Thinking block clearing** (beta): Claude автоматично очищає старі thinking-блоки з попередніх turns. Header: `context-management-2025-06-27`. Підтримується на Claude Sonnet 4/4.5, Haiku 4.5, Opus 4/4.1/4.5.

**Summarization:** Claude 4 моделі повертають резюме thinking (не повний текст). Summarization обробляється іншою моделлю. Claude Sonnet 3.7 повертає повний thinking output.

**Redacted thinking:** Іноді safety-системи шифрують thinking-блок. Повертається як `redacted_thinking` з полем `data`. Зашифровані блоки автоматично дешифруються при передачі назад в API.

```typescript
// Повний приклад Anthropic з AI SDK
import { generateText } from 'ai';
import { anthropic } from '@ai-sdk/anthropic';

const { text, reasoning, usage } = await generateText({
  model: anthropic('claude-sonnet-4-5-20250929'),
  prompt: 'Проаналізуй race condition у цьому коді...',
  maxTokens: 16000,
  providerOptions: {
    anthropic: {
      thinking: {
        type: 'enabled',
        budget_tokens: 8000,
      },
    },
  },
});

// usage.reasoningTokens — кількість reasoning токенів
console.log(`Reasoning tokens: ${usage?.reasoningTokens}`);
console.log(`Total tokens: ${usage?.totalTokens}`);
```

**Prompt caching + thinking:**
- Extended thinking задачі часто тривають > 5 хвилин
- Рекомендують 1-годинний cache duration
- Thinking-блоки кешуються і рахуються як input tokens при читанні з кешу

### OpenAI: Reasoning Models

**Моделі:** o1, o3, o3-mini, o4-mini, GPT-5 (через Responses API)

| Модель | reasoning_effort | Примітка |
|--------|-----------------|----------|
| o1 | low / medium / high | Перша reasoning модель |
| o3 | low / medium / high / xhigh | Найпотужніший reasoning |
| o3-mini | low / medium / high | Баланс ціна/якість |
| o4-mini | low / medium / high | Актуальна дешева reasoning |
| GPT-5 | low / medium / high | Через Responses API |

```typescript
// o4-mini
import { openai } from '@ai-sdk/openai';

const { text, reasoning } = await generateText({
  model: openai('o4-mini'),
  prompt: 'Розв\'яжи...',
  providerOptions: {
    openai: { reasoningEffort: 'high' },
  },
});

// GPT-5 через Responses API
const { text } = await generateText({
  model: openai.responses('gpt-5'),
  prompt: 'Спроектуй...',
  providerOptions: {
    openai: { reasoningEffort: 'medium' },
  },
});
```

**Особливості OpenAI:**
- System messages автоматично конвертуються в developer messages для o-серії
- Reasoning content не показується напряму — тільки summary
- `reasoningEffort` впливає на кількість reasoning tokens і час відповіді

### Google: Thinking Mode

**Моделі:** Gemini 2.5 Flash, Gemini 2.5 Pro

```typescript
import { google } from '@ai-sdk/google';

const { text, reasoning } = await generateText({
  model: google('gemini-2.5-flash'),
  prompt: 'Порівняй...',
  providerOptions: {
    google: {
      thinkingConfig: {
        thinkingBudget: 4096,
      },
    },
  },
});
```

### DeepSeek: R1

**Моделі:** DeepSeek-R1, DeepSeek-V3

DeepSeek-R1 завжди думає. Reasoning контент доступний через `reasoning_content` поле або `<think>` теги.

```typescript
// AI SDK автоматично витягує reasoning
import { createOpenAICompatible } from '@ai-sdk/openai-compatible';

const deepseek = createOpenAICompatible({
  baseURL: 'https://api.deepseek.com/v1',
  name: 'deepseek',
  apiKey: process.env.DEEPSEEK_API_KEY,
});

const { text, reasoning } = await generateText({
  model: deepseek('deepseek-reasoner'),
  prompt: '...',
});
```

Для моделей що повертають reasoning в `<think>` тегах (self-hosted), AI SDK має `extractReasoningMiddleware`:

```typescript
import { extractReasoningMiddleware, wrapLanguageModel } from 'ai';

const wrappedModel = wrapLanguageModel({
  model: yourModel,
  middleware: extractReasoningMiddleware({
    tagName: 'think',
  }),
});
```

---

## Таблиця: Reasoning effort vs задача

| Задача | Рекомендований рівень | Чому |
|--------|----------------------|------|
| FAQ / привітання | **off** | Марнування грошей |
| Категоризація тікетів | **off** або **low** | Модель і так справляється |
| Переклад тексту | **off** | Не потребує логіки |
| Витяг даних (structured output) | **off** або **low** | Зазвичай непотрібно |
| Підсумок документу | **low** | Трохи reasoning допоможе з довгими текстами |
| Порівняльний аналіз | **medium** | Потрібна логіка порівняння |
| Code review | **medium—high** | Пошук помилок потребує аналізу |
| Дебаг складного бага | **high** | Потрібен step-by-step аналіз |
| Математичні розрахунки | **high** | Reasoning різко покращує точність |
| Архітектурне планування | **high** | Потрібен глибокий аналіз trade-offs |
| Аналіз юридичних документів | **high** | Деталі критичні |
| Мульти-крокове планування агента | **high** (planning) + **off** (execution) | Розділити етапи |

---

## Pricing: Reasoning tokens (лютий 2026)

| Модель | Input $/M | Output $/M | Reasoning = Output | Примітка |
|--------|----------|-----------|-------------------|----------|
| Claude Opus 4.6 | $15 | $75 | $75/M | Adaptive — модель контролює обсяг |
| Claude Sonnet 4.5 | $3 | $15 | $15/M | Budget контрольований |
| Claude Haiku 4.5 | $0.80 | $4 | $4/M | Найдешевший reasoning Anthropic |
| o3 | $2 | $8 | $8/M | Потужний reasoning |
| o4-mini | $1.10 | $4.40 | $4.40/M | Баланс |
| GPT-5 | ~$2-3 | ~$8-10 | ~$8-10/M | Через Responses API |
| Gemini 2.5 Flash | $0.15 | $0.60 | $0.60/M | Найдешевший |
| DeepSeek-R1 | $0.55 | $2.19 | Включено | Завжди думає |

*Ціни орієнтовні, перевіряйте на сайтах провайдерів.*

**Формула оцінки вартості reasoning:**

```
Вартість = (input_tokens × input_price) + ((output_tokens + reasoning_tokens) × output_price)
```

Reasoning tokens зазвичай = 2x–10x від output tokens залежно від складності і budget.

---

## Production-патерни

### 1. Adaptive Router: Автоматичний вибір рівня

```typescript
interface RoutingConfig {
  simple: { model: string; reasoning: boolean };
  moderate: { model: string; reasoning: 'low' | 'medium' };
  complex: { model: string; reasoning: 'high' | 'adaptive' };
}

const ROUTING: RoutingConfig = {
  simple: { model: 'gpt-4o-mini', reasoning: false },
  moderate: { model: 'claude-sonnet-4-5', reasoning: 'low' },
  complex: { model: 'claude-opus-4-6', reasoning: 'adaptive' },
};

// Класифікатор на базі keyword heuristics (безкоштовно)
function quickClassify(query: string): 'simple' | 'moderate' | 'complex' {
  const complexKeywords = ['debug', 'analyze', 'architect', 'prove', 'compare', 'review'];
  const simpleKeywords = ['what is', 'how to', 'translate', 'list', 'hello'];
  
  const lower = query.toLowerCase();
  if (complexKeywords.some(k => lower.includes(k))) return 'complex';
  if (simpleKeywords.some(k => lower.includes(k))) return 'simple';
  return 'moderate';
}
```

### 2. Reasoning Budget Monitor

```typescript
// Логувати reasoning tokens для аналізу вартості
async function generateWithTracking(params: any) {
  const start = Date.now();
  const result = await generateText(params);
  const elapsed = Date.now() - start;

  // Логуємо для аналізу
  await logger.log({
    model: params.model.modelId,
    inputTokens: result.usage?.inputTokens,
    outputTokens: result.usage?.outputTokens,
    reasoningTokens: result.usage?.reasoningTokens,
    totalCost: calculateCost(result.usage, params.model.modelId),
    latencyMs: elapsed,
    reasoning: !!result.reasoning,
  });

  return result;
}
```

### 3. Fallback: reasoning → simple при timeout

```typescript
async function generateWithFallback(query: string) {
  try {
    // Спробувати з reasoning, timeout 30 секунд
    return await Promise.race([
      generateText({
        model: anthropic('claude-sonnet-4-5-20250929'),
        prompt: query,
        providerOptions: {
          anthropic: { thinking: { type: 'enabled', budget_tokens: 4096 } },
        },
      }),
      new Promise((_, reject) =>
        setTimeout(() => reject(new Error('Reasoning timeout')), 30000)
      ),
    ]);
  } catch (error) {
    // Fallback: без reasoning
    console.warn('Reasoning timeout, falling back to non-reasoning');
    return await generateText({
      model: anthropic('claude-sonnet-4-5-20250929'),
      prompt: query,
      // Без providerOptions.thinking — reasoning вимкнено
    });
  }
}
```

### 4. Streaming reasoning для UI

```typescript
import { streamText } from 'ai';

const result = streamText({
  model: anthropic('claude-sonnet-4-5-20250929'),
  prompt: 'Проаналізуй...',
  providerOptions: {
    anthropic: {
      thinking: { type: 'enabled', budget_tokens: 8000 },
    },
  },
});

// Стрімити reasoning + text для UI
for await (const part of result.fullStream) {
  if (part.type === 'reasoning') {
    // Показати "модель думає..." з анімацією
    updateUI({ thinking: true, reasoningText: part.textDelta });
  }
  if (part.type === 'text-delta') {
    // Показати фінальну відповідь
    updateUI({ thinking: false, responseText: part.textDelta });
  }
}
```

---

## Поширені помилки

### 1. Reasoning для всіх запитів
```typescript
// ❌ reasoning завжди увімкнений
const result = await generateText({
  model: anthropic('claude-opus-4-6'),
  providerOptions: { anthropic: { thinking: { type: 'adaptive' } } },
  prompt: 'Привіт, як справи?', // reasoning на "привіт" = гроші на вітер
});
```

### 2. budget_tokens > max_tokens
```typescript
// ❌ budget_tokens має бути < max_tokens (для non-interleaved)
thinking: { type: 'enabled', budget_tokens: 20000 }
// з max_tokens: 16000 → помилка
```

### 3. Ігнорування reasoning в мультитурн
```typescript
// ❌ Не передавати thinking-блоки назад в API
// Anthropic вимагає включати їх для збереження потоку reasoning

// ✅ AI SDK робить це автоматично через messages
```

### 4. Очікування що reasoning = правильна відповідь
```
Reasoning покращує якість, але НЕ гарантує правильність.
Модель може "впевнено помилятись" навіть з reasoning.
Завжди верифікуйте критичні результати.
```

---

## Пов'язане

- **Курс:** [Chat API](../../course/02-chat-api.md) — базовий API де reasoning є параметром
- **Курс:** [AI SDK](../../course/03-ai-sdk.md) — unified interface для reasoning
- **Курс:** [Агенти](../../course/08-agents.md) — reasoning в agent loop
- **Курс:** [Оптимізація вартості](../../course/14-cost-optimization.md) — reasoning tokens як стаття витрат
- **Курс:** [Context Engineering](../../course/context-engineering.md) — reasoning як частина контексту
- **KB:** [Multi-agent](../agents/multi-agent.md) — reasoning для координатора, без reasoning для виконавців

## Джерела

- [Anthropic — Extended Thinking docs](https://platform.claude.com/docs/en/build-with-claude/extended-thinking)
- [Anthropic — Claude 4 announcement](https://www.anthropic.com/news/claude-4)
- [AWS — Extended thinking in Bedrock](https://docs.aws.amazon.com/bedrock/latest/userguide/claude-messages-extended-thinking.html)
- [Vercel — AI SDK 4.2: Reasoning](https://vercel.com/blog/ai-sdk-4-2)
- [AI SDK — OpenAI o1 guide](https://sdk.vercel.ai/docs/guides/o1)
- [AI SDK — OpenAI o3-mini guide](https://sdk.vercel.ai/docs/guides/o3)
