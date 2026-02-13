# Модуль 13: Production — Кешування, роутинг, observability

## 🎯 Що ви отримаєте з цього модуля

Після проходження ви будете:
- Реалізовувати prompt caching для зниження вартості на 50-90%
- Налаштовувати fallback та circuit breaker для надійності
- Підключати observability (Langfuse, LangSmith) для моніторингу
- Використовувати AI Gateway як єдину точку входу

**Які задачі це дозволяє вирішувати:** Перетворити прототип на production-систему що працює 24/7, витримує навантаження, автоматично відновлюється при збоях, і не з'їдає бюджет.

---

## 13.1 Prompt Caching — економія 50-90%

Prompt caching дозволяє повторно використовувати результати обробки великих system prompts та контекстів.

### Як це працює

```
Без кешування:
  Запит 1: [System 5000 tok] + [User 100 tok] → обробка 5100 токенів
  Запит 2: [System 5000 tok] + [User 80 tok]  → обробка 5080 токенів
  Разом: ~10,180 токенів оплачено

З кешуванням:
  Запит 1: [System 5000 tok] + [User 100 tok] → обробка 5100 (кеш створено)
  Запит 2: [System cached] + [User 80 tok]    → обробка ~80 токенів
  Разом: ~5,180 токенів оплачено (економія 50%)
```

### OpenAI: Автоматичне кешування

OpenAI кешує автоматично для промптів >1,024 токенів:

```typescript
// OpenAI кешує system prompt автоматично
// Знижка: 50% на кешовані токени, TTL 24 години
const { text } = await generateText({
  model: openai('gpt-4o-mini'),
  system: longSystemPrompt,   // >1024 токенів — буде закешовано
  prompt: userMessage,
});
// Перший запит: повна ціна
// Наступні: system prompt зі знижкою 50%
```

### Anthropic: Explicit кешування

```typescript
import { anthropic } from '@ai-sdk/anthropic';

const { text } = await generateText({
  model: anthropic('claude-sonnet-4-5-20250929'),
  messages: [
    {
      role: 'system',
      content: longSystemPrompt,
      // Явно вказуємо що кешувати
      providerMetadata: {
        anthropic: { cacheControl: { type: 'ephemeral' } },
      },
    },
    { role: 'user', content: userMessage },
  ],
});
// Знижка: 90% на кешовані токени, TTL 5 хвилин
```

### Google Gemini: Context Caching API

Gemini має окремий API для створення кешу:

```typescript
// Google Gemini — створюємо кеш як окремий ресурс
// Крок 1: Створити кеш
const cacheResponse = await fetch(
  `https://generativelanguage.googleapis.com/v1beta/cachedContents?key=${GOOGLE_KEY}`,
  {
    method: 'POST',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify({
      model: 'models/gemini-2.5-flash',
      contents: [{ role: 'user', parts: [{ text: hugeContext }] }],
      ttl: '3600s',  // 1 година (ви контролюєте TTL)
    }),
  }
);
const cache = await cacheResponse.json();

// Крок 2: Використовувати кеш у запитах
const result = await fetch(
  `https://generativelanguage.googleapis.com/v1beta/models/gemini-2.5-flash:generateContent?key=${GOOGLE_KEY}`,
  {
    method: 'POST',
    body: JSON.stringify({
      cachedContent: cache.name,  // Посилання на створений кеш
      contents: [{ role: 'user', parts: [{ text: 'Питання по контексту' }] }],
    }),
  }
);
// Знижка: 75% на кешовані токени
```

### Порівняння кешування між провайдерами

| Аспект | OpenAI | Anthropic | Google Gemini |
|--------|--------|-----------|---------------|
| Тип | Автоматичне | Explicit (`cacheControl`) | Окремий API (`cachedContents`) |
| Мінімум для кешу | 1,024 токенів | 1,024 токенів (Sonnet) | 32,768 токенів |
| TTL | 24 години (автоматичний) | 5 хвилин (можна продовжити) | Ви задаєте (до 1 дня) |
| Знижка | 50% | **90%** (найбільша) | 75% |
| Контроль | Немає (автоматичний) | Повний (ви обираєте що кешувати) | Повний |
| Оплата за запис | Ні | Так (25% надбавка за перший запит) | Так |

### Реальна економія

| Сценарій | Без кешу | З кешем | Економія |
|---------|---------|--------|----------|
| RAG з великим контекстом | $0.50/запит | $0.05/запит | 90% |
| Чатбот з довгим system prompt | $0.10/запит | $0.03/запит | 70% |
| Аналіз PDF (100 стор.) | $3.00/запит | $0.15/запит | 95% |

---

## 13.2 Fallback та Circuit Breaker

### Fallback між провайдерами

```typescript
import { generateText } from 'ai';
import { openai } from '@ai-sdk/openai';
import { anthropic } from '@ai-sdk/anthropic';
import { google } from '@ai-sdk/google';

interface ModelConfig {
  name: string;
  model: () => any;
  maxRetries: number;
}

const fallbackChain: ModelConfig[] = [
  { name: 'Claude Sonnet', model: () => anthropic('claude-sonnet-4-5-20250929'), maxRetries: 2 },
  { name: 'GPT-4o-mini', model: () => openai('gpt-4o-mini'), maxRetries: 2 },
  { name: 'Gemini Flash', model: () => google('gemini-2.5-flash-preview-04-17'), maxRetries: 1 },
];

async function generateWithFallback(params: Omit<Parameters<typeof generateText>[0], 'model'>) {
  for (const config of fallbackChain) {
    try {
      const result = await generateText({
        ...params,
        model: config.model(),
        maxRetries: config.maxRetries,
      });
      return { ...result, provider: config.name };
    } catch (error) {
      console.warn(`[${config.name}] Failed: ${error.message}`);
    }
  }
  throw new Error('All providers failed');
}
```

### Circuit Breaker

Якщо провайдер повторно відмовляє — тимчасово відключаємо його:

```typescript
class CircuitBreaker {
  private failures = 0;
  private lastFailure = 0;
  private state: 'closed' | 'open' | 'half-open' = 'closed';

  constructor(
    private threshold = 5,          // Кількість помилок для відключення
    private resetTimeout = 60_000,  // Час відключення (1 хв)
  ) {}

  async execute<T>(fn: () => Promise<T>): Promise<T> {
    if (this.state === 'open') {
      if (Date.now() - this.lastFailure > this.resetTimeout) {
        this.state = 'half-open'; // Спробуємо ще раз
      } else {
        throw new Error('Circuit breaker is open');
      }
    }

    try {
      const result = await fn();
      this.failures = 0;
      this.state = 'closed';
      return result;
    } catch (error) {
      this.failures++;
      this.lastFailure = Date.now();
      if (this.failures >= this.threshold) {
        this.state = 'open';
      }
      throw error;
    }
  }
}

// Використання
const openaiBreaker = new CircuitBreaker(5, 60_000);
const result = await openaiBreaker.execute(() =>
  generateText({ model: openai('gpt-4o-mini'), prompt: '...' })
);
```

---

## 13.3 Observability — що відбувається всередині

Без observability ваша LLM-система — чорна скринька. Потрібно бачити: скільки коштує, як довго працює, де помилки.

### Langfuse (open-source, рекомендовано)

```typescript
import { Langfuse } from 'langfuse';

const langfuse = new Langfuse({
  publicKey: process.env.LANGFUSE_PUBLIC_KEY,
  secretKey: process.env.LANGFUSE_SECRET_KEY,
});

// Трейсинг кожного запиту
const trace = langfuse.trace({ name: 'customer-support-query', userId });

const generation = trace.generation({
  name: 'classify-ticket',
  model: 'gpt-4o-mini',
  input: userMessage,
});

const { text } = await generateText({
  model: openai('gpt-4o-mini'),
  prompt: userMessage,
});

generation.end({ output: text });

// Тепер у Langfuse dashboard ви бачите:
// - Латентність кожного запиту
// - Вартість по моделях
// - Рейт помилок
// - Повний трейс агентів (кожен крок)
```

### Що моніторити

| Метрика | Навіщо | Алерт при |
|---------|--------|-----------|
| **Латентність (P50, P99)** | UX та SLA | P99 > 10с |
| **Вартість за запит** | Бюджет | >$0.10/запит |
| **Error rate** | Надійність | >5% помилок |
| **Token usage** | Оптимізація | Різке зростання |
| **Finish reason** | Якість | Часті "length" (обрізані відповіді) |

---

## 13.4 AI Gateway

AI Gateway — це проксі між вашим кодом і провайдерами LLM:

```
Ваш код → AI Gateway → OpenAI / Anthropic / Google
                ↓
         Кешування, Rate Limiting,
         Логування, Fallback, Load Balancing
```

### Portkey (рекомендовано)

```typescript
import { createOpenAI } from '@ai-sdk/openai';

// Portkey як проксі — одна зміна baseURL
const model = createOpenAI({
  apiKey: process.env.OPENAI_API_KEY,
  baseURL: 'https://api.portkey.ai/v1',
  headers: {
    'x-portkey-api-key': process.env.PORTKEY_API_KEY,
    'x-portkey-config': JSON.stringify({
      strategy: { mode: 'fallback' },      // Автоматичний fallback
      cache: { mode: 'semantic', ttl: 3600 }, // Семантичне кешування
      retry: { attempts: 3, on_status_codes: [429, 500] },
    }),
  },
});
```

### Альтернативи

| Gateway | Тип | Особливість |
|---------|-----|-------------|
| **Portkey** | SaaS | Найповніший, 1600+ LLMs |
| **LiteLLM** | Self-hosted | Open-source, повний контроль |
| **Cloudflare AI Gateway** | SaaS | Безкоштовні core features |
| **Vercel AI Gateway** | SaaS | Нативна інтеграція з AI SDK 6 |

---

## 13.5 Production Checklist

Перед запуском в production перевірте:

**Надійність:** Fallback між провайдерами, retry з exponential backoff, circuit breaker, timeout для кожного запиту, graceful degradation при недоступності LLM.

**Вартість:** Prompt caching увімкнено, бюджет на день/місяць встановлено, алерти при перевищенні бюджету, моніторинг вартості по функціях.

**Observability:** Трейсинг кожного LLM-запиту, метрики латентності та помилок, логування input/output для дебагу, дашборд для команди.

**Масштабування:** Rate limiting для кожного користувача, черга для batch-запитів, горизонтальне масштабування (stateless архітектура).

---

## Перевір себе

1. Як prompt caching економить 90% для RAG-застосунку?
2. Чим explicit (Anthropic) кешування відрізняється від automatic (OpenAI)?
3. Реалізуйте fallback chain: Claude → GPT → Gemini
4. Що таке circuit breaker і навіщо він потрібен?
5. Які 5 метрик LLM-системи обов'язково моніторити?

---

**Назад:** [← Модуль 12 — Agent Frameworks](12-agent-frameworks.md) | **Далі:** [Модуль 14 — Оптимізація вартості →](14-cost-optimization.md)
