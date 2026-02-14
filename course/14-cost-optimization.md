# Модуль 14: Оптимізація вартості — Як зменшити витрати на 80%

## 🎯 Що ви отримаєте з цього модуля

Після проходження ви будете:
- Застосовувати 5 стратегій оптимізації вартості LLM
- Реалізовувати model routing (дешева модель для простих задач)
- Використовувати batch API для економії 50%
- Будувати калькулятор вартості для свого продукту

**Які задачі це дозволяє вирішувати:** Перетворити AI-продукт з "занадто дорого для production" на "вигідний бізнес". Зменшити рахунок за LLM API з $10,000/міс до $2,000/міс без втрати якості.

---

## 💰 Бізнес-цінність

**Проблема клієнта:** "Наш AI-продукт коштує $10K/місяць на API — це занадто."
**Рішення з цього модуля:** П'ять рівнів оптимізації: routing, caching, prompts, batching, дешевші моделі.
**Як продати:** "Аудит вартості AI-рішення" — проект на $3–10K. Типова економія: 60–80% = десятки тисяч на рік. ROI за перший місяць.

## 14.1 П'ять рівнів оптимізації

```
Рівень 1: Prompt caching     → 30-50% економії (Модуль 13)
Рівень 2: Model routing      → до 85% економії
Рівень 3: Batch processing   → 50% знижка
Рівень 4: Token optimization → 15-40% економії
Рівень 5: Model distillation → 40-200x довгострокова економія
```

---

## 14.2 Model Routing — правильна модель для кожної задачі

Не кожен запит потребує GPT-5 або Claude Opus. 80% запитів може обробити дешева модель:

```typescript
import { generateText, generateObject } from 'ai';
import { openai } from '@ai-sdk/openai';
import { anthropic } from '@ai-sdk/anthropic';
import { z } from 'zod';

// Крок 1: Класифікуємо складність запиту (дешевою моделлю)
async function routeRequest(message: string) {
  const { object } = await generateObject({
    model: openai('gpt-4o-mini'),  // $0.15/1M — дуже дешево
    schema: z.object({
      complexity: z.enum(['simple', 'moderate', 'complex']),
      reasoning: z.string(),
    }),
    temperature: 0,
    prompt: `Оціни складність запиту:
"${message}"

simple: проста відповідь, FAQ, привітання
moderate: генерація тексту, аналіз, пояснення
complex: складна логіка, код, мульти-крокове міркування`,
  });

  return object.complexity;
}

// Крок 2: Обираємо модель на основі складності
function getModelForComplexity(complexity: string) {
  switch (complexity) {
    case 'simple':
      return openai('gpt-4o-mini');        // $0.15/1M — FAQ, привітання
    case 'moderate':
      return openai('gpt-5-mini');         // $0.25/1M — генерація, аналіз
    case 'complex':
      return anthropic('claude-sonnet-4-5-20250929'); // $3/1M — складні задачі
  }
}

// Крок 3: Генеруємо відповідь правильною моделлю
async function smartGenerate(message: string) {
  const complexity = await routeRequest(message);
  const model = getModelForComplexity(complexity);

  console.log(`[Router] Складність: ${complexity} → ${model.modelId}`);

  const { text } = await generateText({ model, prompt: message });
  return text;
}
```

### Економія від routing

| Розподіл запитів | Без routing (все Claude Sonnet) | З routing | Економія |
|-----------------|-------------------------------|-----------|----------|
| 60% simple | $3.00/1M × 0.6 = $1.80 | $0.15/1M × 0.6 = $0.09 | — |
| 30% moderate | $3.00/1M × 0.3 = $0.90 | $0.25/1M × 0.3 = $0.08 | — |
| 10% complex | $3.00/1M × 0.1 = $0.30 | $3.00/1M × 0.1 = $0.30 | — |
| **Разом** | **$3.00/1M** | **$0.47/1M** | **84%** |

---

## 14.3 Batch API — 50% знижка

Для задач що не потребують відповіді в реальному часі:

```typescript
import OpenAI from 'openai';

const client = new OpenAI();

// Створюємо batch з запитів
const batch = await client.batches.create({
  input_file_id: await uploadBatchFile([
    { custom_id: 'req-1', body: { model: 'gpt-4o-mini', messages: [{ role: 'user', content: 'Класифікуй: ...' }] } },
    { custom_id: 'req-2', body: { model: 'gpt-4o-mini', messages: [{ role: 'user', content: 'Класифікуй: ...' }] } },
    // ... тисячі запитів
  ]),
  endpoint: '/v1/chat/completions',
  completion_window: '24h',  // Результат протягом 24 годин
});

// Ціна: 50% від звичайної
// GPT-4o-mini batch: $0.075/1M замість $0.15/1M
```

**Ідеально для:** щоденна класифікація тікетів, обробка email, генерація описів товарів, аналіз відгуків.

---

## 🆕 14.3.1 Reasoning Budget — нова стаття витрат

Reasoning tokens — це output-токени які модель витрачає на "обдумування". Вони можуть бути в 2–10x більші за саму відповідь:

```
Запит з reasoning (budget 8000):
  Input: 2000 токенів → $0.006 (при $3/M)
  Output: 500 токенів → $0.0075
  Reasoning: ~6000 токенів → $0.09   ← ОСНОВНА СТАТТЯ!
  Разом: ~$0.104

Той самий запит БЕЗ reasoning:
  Разом: ~$0.014 (у 7 разів дешевше)
```

**Оптимізація reasoning бюджету:**

```typescript
// ❌ Reasoning для ВСІХ запитів — дорого
providerOptions: { anthropic: { thinking: { type: 'adaptive' } } }

// ✅ Reasoning тільки для складних задач
const complexity = classifyComplexity(query); // дешевий виклик
if (complexity === 'complex') {
  // Reasoning з мінімальним бюджетом що працює
  providerOptions: { anthropic: { thinking: { type: 'enabled', budget_tokens: 2048 } } }
}
// Для простих — без reasoning взагалі
```

Anthropic рекомендує: починайте з мінімуму (1024 tokens) і збільшуйте інкрементально. Детальніше: [Reasoning Models](reasoning-models.md).

---

## 14.4 Token Optimization

### Стиснення промптів

```typescript
// ❌ Повний промпт — ~200 токенів
const verbose = `You are a helpful customer support assistant for our e-commerce platform.
Your job is to help customers with their questions about orders, returns, and products.
Please be polite, professional, and concise in your responses.
Always check the order status before making any promises.`;

// ✅ Стиснений промпт — ~80 токенів (та ж якість)
const concise = `E-commerce support agent. Be concise. Check order status before promises.`;
```

### Обмеження довжини відповіді

```typescript
// Для класифікації — не потрібна довга відповідь
const { text } = await generateText({
  model: openai('gpt-4o-mini'),
  maxTokens: 10,  // Достатньо для одного слова-категорії
  prompt: 'Класифікуй: "Де моє замовлення?" → billing/technical/order/other',
});
```

### Зменшення контексту

```typescript
// Замість всієї історії — summary + останні 5 повідомлень
function optimizeContext(messages: CoreMessage[]): CoreMessage[] {
  if (messages.length <= 10) return messages;

  const system = messages[0];
  const summary = summarize(messages.slice(1, -5));
  const recent = messages.slice(-5);

  return [
    system,
    { role: 'assistant', content: `[Контекст: ${summary}]` },
    ...recent,
  ];
}
```

### TOON — Token-Oriented Object Notation

Коли ви передаєте структуровані дані в промпт (товари, замовлення, логи, результати RAG), JSON з'їдає токени на дужки, лапки та повторення ключів. **TOON** — це компактний формат який кодує ту ж JSON-структуру, але використовує на **30-60% менше токенів**.

```
// JSON — 22,250 токенів для 60 записів аналітики
{
  "metrics": [
    { "date": "2025-01-01", "views": 5715, "clicks": 211, "conversions": 28, "revenue": 7976.46 },
    { "date": "2025-01-02", "views": 7103, "clicks": 393, "conversions": 28, "revenue": 8360.53 },
    { "date": "2025-01-03", "views": 7248, "clicks": 378, "conversions": 24, "revenue": 3212.57 }
  ]
}

// TOON — 9,120 токенів для тих самих даних (−59%!)
metrics[3]{date,views,clicks,conversions,revenue}:
  2025-01-01,5715,211,28,7976.46
  2025-01-02,7103,393,28,8360.53
  2025-01-03,7248,378,24,3212.57
```

**Як це працює:** TOON оголошує поля один раз у header (`{date,views,...}`), вказує кількість записів (`[3]`), і далі записує лише значення рядок за рядком — як CSV, але з підтримкою вкладеності.

#### Встановлення та використання

```typescript
import { encode, decode } from '@toon-format/toon';

// JSON → TOON (перед відправкою в LLM)
const data = {
  users: [
    { id: 1, name: 'Alice', role: 'admin', lastLogin: '2025-01-15' },
    { id: 2, name: 'Bob', role: 'user', lastLogin: '2025-01-14' },
  ],
};

const toonString = encode(data);
// users[2]{id,name,role,lastLogin}:
//   1,Alice,admin,2025-01-15
//   2,Bob,user,2025-01-14

// Використання в промпті
const { text } = await generateText({
  model: openai('gpt-4o-mini'),
  prompt: `Data is in TOON format (arrays show length and fields).

\`\`\`toon
${toonString}
\`\`\`

Task: Which users have role "admin"?`,
});

// TOON → JSON (якщо модель повертає TOON)
const parsed = decode(modelOutput, { strict: true }); // strict: ловить помилки
```

#### Streaming великих датасетів

```typescript
import { encodeLines } from '@toon-format/toon';

// Для великих масивів — потоковий encode (не тримає все в пам'яті)
const largeData = await fetchThousandsOfRecords();
let toonOutput = '';
for (const line of encodeLines(largeData, { delimiter: '\t' })) {
  toonOutput += line + '\n';
}
// Tab-роздільник ще ефективніший за кому (менше токенів)
```

#### Коли TOON економить найбільше

| Тип даних | Економія vs JSON | Приклад |
|-----------|-----------------|---------|
| Однорідні масиви об'єктів | **40-60%** | Список товарів, юзерів, логів |
| Плоскі таблиці | **35-50%** | Аналітика, метрики, CSV-подібні дані |
| Напів-однорідні дані | **15-30%** | Мікс простих і вкладених об'єктів |
| Глибоко вкладені об'єкти | **0-10%** | Конфігурації, дерева — тут JSON compact краще |

#### Benchmarks: TOON vs JSON

За результатами бенчмарків на 4 моделях (Claude Haiku, Gemini Flash, GPT-5-nano, Grok 4) і 209 питаннях:

| Формат | Accuracy | Токени | Ефективність (acc/1K tok) |
|--------|----------|--------|--------------------------|
| **TOON** | **73.9%** | **2,744** | **26.9** |
| JSON compact | 70.7% | 3,081 | 22.9 |
| YAML | 69.0% | 3,719 | 18.6 |
| JSON | 69.7% | 4,545 | 15.3 |
| XML | 67.1% | 5,167 | 13.0 |

TOON не тільки менший за JSON, але й дає **вищу accuracy** — явні `[N]` довжини та `{fields}` headers допомагають моделі краще відстежувати структуру.

#### Коли НЕ використовувати TOON

TOON не срібна куля. Використовуйте JSON коли:
- Дані глибоко вкладені з мінімумом табличних масивів
- Структура неоднорідна (кожен об'єкт має різні поля)
- Ваш pipeline вже оптимізований під JSON (не варто мігрувати заради 10% економії)
- Для чисто табличних даних CSV може бути ще компактнішим (але без вкладеності)

Документація та playground: **https://toonformat.dev**

---

## 14.5 Калькулятор вартості

```typescript
interface CostEstimate {
  perRequest: number;
  daily: number;
  monthly: number;
  breakdown: Record<string, number>;
}

function estimateProductCost(config: {
  dailyRequests: number;
  avgInputTokens: number;
  avgOutputTokens: number;
  modelPricing: { input: number; output: number }; // per 1M tokens
  cachingRate?: number;   // 0-1, частка кешованих запитів
  routingRate?: number;   // 0-1, частка запитів на дешевшу модель
  cheapModelPricing?: { input: number; output: number };
}): CostEstimate {
  const { dailyRequests, avgInputTokens, avgOutputTokens, modelPricing } = config;
  const cachingRate = config.cachingRate ?? 0;
  const routingRate = config.routingRate ?? 0;

  // Базова вартість
  const baseCostPerReq =
    (avgInputTokens / 1_000_000) * modelPricing.input +
    (avgOutputTokens / 1_000_000) * modelPricing.output;

  // Зі кешуванням: кешовані токени коштують 10% (Anthropic) або 50% (OpenAI)
  const cacheSavings = baseCostPerReq * cachingRate * 0.5;

  // З routing: частина запитів на дешевшу модель
  let routingSavings = 0;
  if (config.cheapModelPricing) {
    const cheapCost =
      (avgInputTokens / 1_000_000) * config.cheapModelPricing.input +
      (avgOutputTokens / 1_000_000) * config.cheapModelPricing.output;
    routingSavings = (baseCostPerReq - cheapCost) * routingRate;
  }

  const optimizedCost = baseCostPerReq - cacheSavings - routingSavings;

  return {
    perRequest: optimizedCost,
    daily: optimizedCost * dailyRequests,
    monthly: optimizedCost * dailyRequests * 30,
    breakdown: {
      basePerRequest: baseCostPerReq,
      cacheSavings,
      routingSavings,
      finalPerRequest: optimizedCost,
    },
  };
}

// Приклад: чатбот підтримки
const estimate = estimateProductCost({
  dailyRequests: 10_000,
  avgInputTokens: 2000,
  avgOutputTokens: 500,
  modelPricing: { input: 3, output: 15 },      // Claude Sonnet
  cachingRate: 0.7,                              // 70% кешовано
  routingRate: 0.6,                              // 60% на дешеву модель
  cheapModelPricing: { input: 0.15, output: 0.6 }, // GPT-4o-mini
});

console.log(`Місячна вартість: $${estimate.monthly.toFixed(2)}`);
```

---

## 14.6 Pricing Models для AI-продуктів

Як заробляти якщо AI коштує грошей:

| Модель | Як працює | Приклад |
|--------|----------|---------|
| **Per-seat** | Підписка за користувача | $20/міс за юзера (ваш кост ~$3) |
| **Usage-based** | Плата за використання | $0.05 за запит (ваш кост ~$0.01) |
| **Outcome-based** | Плата за результат | $0.99 за вирішений тікет |
| **Tiered** | Рівні з лімітами | Free (100 запитів), Pro (10K), Enterprise |

**Здорова маржа AI-продукту:** 50-60% gross margin (якщо менше — оптимізуйте).

---

## Перевір себе

1. Назвіть 5 рівнів оптимізації вартості LLM
2. Порахуйте: 10,000 запитів/день, 1000 input + 300 output токенів, GPT-4o-mini. Скільки на місяць?
3. Як model routing знижує вартість на 84%?
4. Коли використовувати Batch API? Які обмеження?
5. Яку pricing model ви б обрали для AI-чатбота підтримки?

---

**Назад:** [← Модуль 13 — Production](13-production.md) | **Далі:** [Модуль 15 — Безпека →](15-security.md)
