# Модуль 05: Structured Output — Отримання даних у потрібному форматі

## 🎯 Що ви отримаєте з цього модуля

Після проходження ви будете:
- Отримувати типізовані TypeScript об'єкти від LLM замість вільного тексту
- Використовувати Zod-схеми для валідації відповідей
- Знати різницю між native structured output і "JSON mode"
- Будувати pipeline обробки даних з гарантією формату

**Які задачі це дозволяє вирішувати:** Витягування контактів з email, парсинг рахунків, класифікація з метаданими, генерація каталогів товарів, аналіз резюме — будь-яка задача де потрібні дані, а не текст.

---

## 5.1 Проблема вільного тексту

Коли ви просите LLM "відповісти у JSON", результат ненадійний:

```typescript
// ❌ Ненадійний підхід — просити JSON текстом
const { text } = await generateText({
  model: openai('gpt-4o-mini'),
  prompt: 'Витягни ім\'я та email з тексту. Відповідай JSON. Текст: "Привіт, я Олег, пишіть мені на oleg@test.com"',
});

// Може повернути:
// '{"name": "Олег", "email": "oleg@test.com"}'  ← OK
// 'Ось JSON:\n```json\n{"name": "Олег"}\n```'    ← Markdown обгортка
// '{"name": "Олег", "mail": "oleg@test.com"}'    ← Інша назва поля
// 'Ім\'я: Олег, Email: oleg@test.com'            ← Взагалі не JSON

JSON.parse(text);  // 💥 Може впасти!
```

## 5.2 Рішення: generateObject + Zod

`generateObject` **гарантує** що результат відповідає вашій Zod-схемі:

```typescript
import { generateObject } from 'ai';
import { openai } from '@ai-sdk/openai';
import { z } from 'zod';

// Визначаємо ТОЧНУ структуру яку хочемо
const ContactSchema = z.object({
  name: z.string().describe('Повне ім\'я людини'),
  email: z.string().email().describe('Email адреса'),
  company: z.string().nullable().describe('Назва компанії, null якщо не вказано'),
  role: z.string().nullable().describe('Посада, null якщо не вказано'),
});

const { object } = await generateObject({
  model: openai('gpt-4o-mini'),
  schema: ContactSchema,
  prompt: 'Привіт, я Олег Шевченко, CTO у TechStartup, пишіть на oleg@techstartup.io',
});

// object має ТИП ContactSchema — TypeScript знає структуру!
console.log(object.name);     // "Олег Шевченко"
console.log(object.email);    // "oleg@techstartup.io"
console.log(object.company);  // "TechStartup"
console.log(object.role);     // "CTO"
```

### Як це працює під капотом

1. AI SDK конвертує Zod-схему в JSON Schema
2. Відправляє її разом із запитом до провайдера
3. Провайдер (OpenAI/Anthropic/Google) **гарантує** що відповідь відповідає схемі
4. AI SDK валідує та парсить результат
5. Ви отримуєте типізований об'єкт

### Різниця між провайдерами

Код з AI SDK **ідентичний** — міняється лише один рядок `model:`:

```typescript
import { generateObject } from 'ai';
import { openai } from '@ai-sdk/openai';
import { anthropic } from '@ai-sdk/anthropic';
import { google } from '@ai-sdk/google';

// OpenAI — native JSON Schema mode (найстабільніший structured output)
const result1 = await generateObject({
  model: openai('gpt-4o-mini'),
  schema: ContactSchema,
  prompt: text,
});

// Anthropic — використовує tool_use під капотом для structured output
// Claude краще працює з складними вкладеними об'єктами та .describe()
const result2 = await generateObject({
  model: anthropic('claude-sonnet-4-5-20250929'),
  schema: ContactSchema,
  prompt: text,
});

// Google Gemini — має власний responseSchema mode
// Найдешевший варіант, Gemini Flash дає хороші результати для простих схем
const result3 = await generateObject({
  model: google('gemini-2.5-flash-preview-04-17'),
  schema: ContactSchema,
  prompt: text,
});
```

**Але під капотом кожен провайдер реалізує structured output по-різному:**

| Провайдер | Механізм | Обмеження |
|-----------|---------|-----------|
| **OpenAI** | Native `response_format: { type: "json_schema" }` | Не підтримує `z.union()`, max 5 рівнів вкладеності |
| **Anthropic** | Використовує `tool_use` з одним tool | Немає strict mode, AI SDK додає валідацію |
| **Google** | `responseSchema` в Gemini API | Обмежена підтримка `nullable`, не всі Zod типи |

Якщо ви хочете **без AI SDK** (щоб зрозуміти що відбувається під капотом):

```typescript
// OpenAI — напряму через API
const openaiResponse = await fetch('https://api.openai.com/v1/chat/completions', {
  method: 'POST',
  headers: { 'Authorization': `Bearer ${OPENAI_API_KEY}`, 'Content-Type': 'application/json' },
  body: JSON.stringify({
    model: 'gpt-4o-mini',
    messages: [{ role: 'user', content: text }],
    response_format: {
      type: 'json_schema',
      json_schema: {
        name: 'contact',
        strict: true,  // OpenAI гарантує 100% відповідність схемі
        schema: zodToJsonSchema(ContactSchema),
      },
    },
  }),
});

// Anthropic — використовує tool_use як хак для structured output
const anthropicResponse = await fetch('https://api.anthropic.com/v1/messages', {
  method: 'POST',
  headers: { 'x-api-key': ANTHROPIC_API_KEY, 'Content-Type': 'application/json', 'anthropic-version': '2023-06-01' },
  body: JSON.stringify({
    model: 'claude-sonnet-4-5-20250929',
    max_tokens: 1024,
    messages: [{ role: 'user', content: text }],
    tools: [{
      name: 'extract_contact',
      description: 'Extract contact info',
      input_schema: zodToJsonSchema(ContactSchema),
    }],
    tool_choice: { type: 'tool', name: 'extract_contact' },  // Примусово викликати tool
  }),
});
// Результат буде в response.content[0].input (тип tool_use)

// Google Gemini — responseSchema
const geminiResponse = await fetch(
  `https://generativelanguage.googleapis.com/v1beta/models/gemini-2.5-flash:generateContent?key=${GOOGLE_API_KEY}`,
  {
    method: 'POST',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify({
      contents: [{ parts: [{ text }] }],
      generationConfig: {
        responseMimeType: 'application/json',
        responseSchema: zodToJsonSchema(ContactSchema),
      },
    }),
  }
);
```

**Висновок:** AI SDK абстрагує ці відмінності — ви пишете один код, а він адаптує під кожен провайдер. Але розуміння механізмів допомагає при дебагу.

---

## 5.3 Практичні приклади

### Класифікація з метаданими

```typescript
const TicketAnalysis = z.object({
  category: z.enum(['billing', 'technical', 'feature_request', 'complaint', 'other']),
  priority: z.enum(['low', 'medium', 'high', 'critical']),
  sentiment: z.enum(['positive', 'neutral', 'negative', 'angry']),
  language: z.string().describe('Мова повідомлення (ISO 639-1 код)'),
  requiresHuman: z.boolean().describe('Чи потрібна участь живого оператора'),
  suggestedResponse: z.string().describe('Запропонована відповідь клієнту'),
});

const { object: ticket } = await generateObject({
  model: openai('gpt-4o-mini'),
  schema: TicketAnalysis,
  prompt: `Проаналізуй тікет підтримки:
"Вже третій день не працює оплата! Я втрачаю клієнтів! Зробіть щось нарешті!!!"`,
});

// ticket.category === 'technical'
// ticket.priority === 'critical'
// ticket.sentiment === 'angry'
// ticket.requiresHuman === true
```

### Витягування даних з документа

```typescript
const InvoiceSchema = z.object({
  invoiceNumber: z.string(),
  date: z.string().describe('Дата у форматі YYYY-MM-DD'),
  seller: z.object({
    name: z.string(),
    taxId: z.string().nullable(),
  }),
  buyer: z.object({
    name: z.string(),
    taxId: z.string().nullable(),
  }),
  items: z.array(z.object({
    description: z.string(),
    quantity: z.number(),
    unitPrice: z.number(),
    total: z.number(),
  })),
  totalAmount: z.number(),
  currency: z.string(),
});

const { object: invoice } = await generateObject({
  model: openai('gpt-4o-mini'),
  schema: InvoiceSchema,
  prompt: `Витягни дані з рахунку:
${invoiceText}`,
});

// Тепер можна зберегти в базу, згенерувати звіт, тощо
await db.invoices.create(invoice);
```

### Генерація масиву об'єктів

```typescript
const { object: products } = await generateObject({
  model: openai('gpt-4o-mini'),
  output: 'array',  // Генеруємо масив, а не один об'єкт
  schema: z.object({
    name: z.string(),
    description: z.string().max(150),
    priceRange: z.enum(['budget', 'mid', 'premium']),
    targetAudience: z.string(),
  }),
  prompt: 'Згенеруй 5 ідей для AI SaaS-продуктів для малого бізнесу',
});

// products — масив з 5 типізованих об'єктів
```

### Enum — отримання одного значення

```typescript
const { object: category } = await generateObject({
  model: openai('gpt-4o-mini'),
  output: 'enum',
  enum: ['spam', 'promotion', 'personal', 'work', 'newsletter'],
  prompt: `Класифікуй email: "${emailSubject}"`,
});

// category === 'work' (типізований як один з enum)
```

---

## 5.4 Поради для написання схем

### Використовуйте .describe() для кожного поля

```typescript
// ❌ Без описів — модель вгадує
z.object({
  score: z.number(),
  flag: z.boolean(),
});

// ✅ З описами — модель розуміє контекст
z.object({
  score: z.number().min(0).max(10).describe('Оцінка якості від 0 до 10'),
  flag: z.boolean().describe('true якщо потрібна ручна перевірка'),
});
```

### Обмежуйте можливі значення через enum

```typescript
// ❌ string — модель може написати будь-що
z.object({ status: z.string() });

// ✅ enum — лише допустимі значення
z.object({ status: z.enum(['active', 'inactive', 'pending', 'archived']) });
```

### nullable для опціональних полів

```typescript
z.object({
  phone: z.string().nullable().describe('Телефон, null якщо не вказано в тексті'),
  // НЕ використовуйте .optional() — використовуйте .nullable()
});
```

---

## 5.5 streamObject — стрімінг структурованих даних

Для великих об'єктів можна стрімити по частинах:

```typescript
import { streamObject } from 'ai';

const result = streamObject({
  model: openai('gpt-4o-mini'),
  schema: InvoiceSchema,
  prompt: `Витягни дані з рахунку: ${text}`,
});

// Отримуємо часткові результати по мірі генерації
for await (const partialObject of result.partialObjectStream) {
  console.log('Проміжний результат:', partialObject);
  // { invoiceNumber: "INV-001" }
  // { invoiceNumber: "INV-001", date: "2026-01-15" }
  // { invoiceNumber: "INV-001", date: "2026-01-15", seller: { name: "..." } }
  // ...
}

// Або дочекатись повного об'єкту
const finalObject = await result.object;
```

---

## Перевір себе

1. Чому `generateText` + `JSON.parse` гірше ніж `generateObject`?
2. Напишіть Zod-схему для витягування даних з резюме (ім'я, досвід, навички, мови)
3. Як генерувати масив об'єктів (не один об'єкт)?
4. Чому `.describe()` на полях Zod-схеми важливий?
5. Коли використовувати `streamObject` замість `generateObject`?

---

**Назад:** [← Модуль 04 — Промпт-інженерія](04-prompting.md) | **Далі:** [Модуль 06 — Function Calling →](06-function-calling.md)
