# Модуль 02: Перший запит — Chat API на практиці

## 🎯 Що ви отримаєте з цього модуля

Після проходження ви будете:
- Розуміти структуру Chat API (messages, roles, response)
- Вміти відправляти запити до OpenAI, Anthropic, Google через їхні API
- Розуміти потік даних: ваш код → API → модель → відповідь
- Вміти обробляти помилки та edge cases

**Які задачі це дозволяє вирішувати:** Побудувати будь-який застосунок де потрібна генерація тексту — від простого чатбота до складної системи обробки документів. Це фундамент для всього іншого.

---

## 💰 Бізнес-цінність

**Проблема клієнта:** "Нам потрібен простий AI-асистент для внутрішніх задач."
**Рішення з цього модуля:** Прямий виклик Chat API — мінімальний MVP за кілька годин.
**Як продати:** "Прототип AI-асистента за 1 день" — проект на $500–2K. Клієнт бачить результат, ви отримуєте довіру на більший проект.

## 2.1 Структура Chat API

Всі сучасні LLM працюють через **Chat API** — ви надсилаєте список повідомлень, модель відповідає наступним повідомленням.

### Анатомія запиту

```typescript
// Це основна структура БУДЬ-ЯКОГО запиту до LLM
const request = {
  model: 'gpt-4o-mini',        // Яку модель використовувати
  messages: [                    // Масив повідомлень (діалог)
    {
      role: 'system',            // Інструкція для моделі
      content: 'Ти — корисний асистент. Відповідай українською.'
    },
    {
      role: 'user',              // Повідомлення від користувача
      content: 'Що таке TypeScript?'
    }
  ],
  temperature: 0.7,              // Опціонально
  max_tokens: 500,               // Опціонально
};
```

### Три ролі в діалозі

| Роль | Хто це | Коли використовувати |
|------|--------|---------------------|
| `system` | Ваша інструкція моделі | Один раз на початку. Задає поведінку, тон, обмеження |
| `user` | Повідомлення від користувача | Кожен запит від кінцевого користувача |
| `assistant` | Попередні відповіді моделі | Для збереження контексту діалогу |

### Як працює діалог (multi-turn)

```typescript
// Крок 1: Користувач питає
const messages = [
  { role: 'system', content: 'Ти — TypeScript-експерт.' },
  { role: 'user', content: 'Що таке generic у TypeScript?' },
];
// → Модель відповідає про generic-и

// Крок 2: Ви додаєте відповідь моделі + нове питання
messages.push(
  { role: 'assistant', content: '...(відповідь моделі про generic-и)...' },
  { role: 'user', content: 'Покажи приклад з масивом' },
);
// → Модель бачить ВСЮ історію і відповідає в контексті попередньої розмови
```

**Ключовий момент:** Модель **не пам'ятає** попередні запити. Ви самі передаєте всю історію з кожним запитом. Це означає що з кожним повідомленням ви платите за всю історію знову.

---

## 2.2 Прямий виклик API (без SDK)

Перш ніж використовувати SDK, корисно зрозуміти що відбувається під капотом — це звичайний HTTP POST запит:

```typescript
// raw-api.ts — без SDK, чистий fetch
import 'dotenv/config';

async function callOpenAI() {
  const response = await fetch('https://api.openai.com/v1/chat/completions', {
    method: 'POST',
    headers: {
      'Content-Type': 'application/json',
      'Authorization': `Bearer ${process.env.OPENAI_API_KEY}`,
    },
    body: JSON.stringify({
      model: 'gpt-4o-mini',
      messages: [
        { role: 'system', content: 'Відповідай коротко, одним реченням.' },
        { role: 'user', content: 'Що таке Promise в JavaScript?' },
      ],
      temperature: 0.3,
    }),
  });

  const data = await response.json();

  // Структура відповіді
  console.log('Відповідь:', data.choices[0].message.content);
  console.log('Токени:', data.usage);
  // { prompt_tokens: 28, completion_tokens: 35, total_tokens: 63 }
  console.log('Причина зупинки:', data.choices[0].finish_reason);
  // "stop" — модель закінчила сама
  // "length" — досягнуто max_tokens
}

callOpenAI();
```

### Структура відповіді API

```typescript
// Те що повертає API OpenAI
interface ChatCompletion {
  id: string;                     // Унікальний ID запиту
  object: 'chat.completion';
  created: number;                // Unix timestamp
  model: string;                  // Яка модель реально відповіла
  choices: [{
    index: number;
    message: {
      role: 'assistant';
      content: string;            // ← САМА ВІДПОВІДЬ
    };
    finish_reason: 'stop' | 'length' | 'tool_calls';
  }];
  usage: {
    prompt_tokens: number;        // Скільки токенів у вашому запиті
    completion_tokens: number;    // Скільки токенів у відповіді
    total_tokens: number;         // Сума
  };
}
```

### А тепер — те ж саме для Anthropic та Google

Кожен провайдер має **свій формат запитів та відповідей**. Порівняйте:

```typescript
// Anthropic Claude API — ІНШИЙ формат!
async function callAnthropic() {
  const response = await fetch('https://api.anthropic.com/v1/messages', {
    method: 'POST',
    headers: {
      'Content-Type': 'application/json',
      'x-api-key': process.env.ANTHROPIC_API_KEY!,
      'anthropic-version': '2023-06-01',        // Обов'язковий заголовок версії
    },
    body: JSON.stringify({
      model: 'claude-sonnet-4-5-20250929',
      max_tokens: 1024,                          // В Anthropic — обов'язковий параметр!
      system: 'Відповідай коротко, одним реченням.',  // system — окреме поле, НЕ в messages
      messages: [
        { role: 'user', content: 'Що таке Promise в JavaScript?' },
      ],
    }),
  });

  const data = await response.json();

  // Структура відповіді — ІНША ніж у OpenAI
  console.log('Відповідь:', data.content[0].text);   // content[0].text, НЕ choices[0].message.content
  console.log('Токени:', data.usage);
  // { input_tokens: 25, output_tokens: 30 }          // input/output, НЕ prompt/completion
  console.log('Причина зупинки:', data.stop_reason);  // stop_reason, НЕ finish_reason
}

// Google Gemini API — ТРЕТІЙ формат!
async function callGemini() {
  const response = await fetch(
    `https://generativelanguage.googleapis.com/v1beta/models/gemini-2.5-flash:generateContent?key=${process.env.GOOGLE_API_KEY}`,
    {
      method: 'POST',
      headers: { 'Content-Type': 'application/json' },
      body: JSON.stringify({
        systemInstruction: {                              // system prompt — окрема структура
          parts: [{ text: 'Відповідай коротко, одним реченням.' }],
        },
        contents: [{                                      // contents, НЕ messages
          role: 'user',
          parts: [{ text: 'Що таке Promise в JavaScript?' }],  // parts[].text, НЕ content
        }],
        generationConfig: {                               // Параметри — в окремому об'єкті
          temperature: 0.3,
        },
      }),
    }
  );

  const data = await response.json();

  // Структура відповіді — ЗНОВУ ІНША
  console.log('Відповідь:', data.candidates[0].content.parts[0].text);
  // candidates[0].content.parts[0].text — найглибше вкладена відповідь з трьох API
  console.log('Токени:', data.usageMetadata);
  // { promptTokenCount: 22, candidatesTokenCount: 28, totalTokenCount: 50 }
}
```

### Порівняння трьох API

| Аспект | OpenAI | Anthropic | Google Gemini |
|--------|--------|-----------|---------------|
| Auth | `Authorization: Bearer` | `x-api-key` + `anthropic-version` | `?key=` у URL |
| System prompt | `role: 'system'` в messages | Окреме поле `system` | `systemInstruction.parts` |
| Messages | `messages[].content` | `messages[].content` | `contents[].parts[].text` |
| Відповідь | `choices[0].message.content` | `content[0].text` | `candidates[0].content.parts[0].text` |
| Токени | `prompt_tokens` / `completion_tokens` | `input_tokens` / `output_tokens` | `promptTokenCount` / `candidatesTokenCount` |
| max_tokens | Опціональний | **Обов'язковий** | Опціональний (`maxOutputTokens`) |
| Стоп-причина | `finish_reason` | `stop_reason` | `finishReason` (camelCase!) |

**Саме тому AI SDK існує** — щоб ви не мали справу з цим зоопарком.

### 🆕 Reasoning параметри (2025–2026)

Сучасні моделі підтримують **reasoning** — окремий етап "обдумування" перед відповіддю. Це новий базовий параметр API, поряд з `temperature` і `max_tokens`:

```typescript
// OpenAI — reasoning_effort
const response = await fetch('https://api.openai.com/v1/responses', {
  headers: { 'Authorization': `Bearer ${apiKey}`, 'Content-Type': 'application/json' },
  method: 'POST',
  body: JSON.stringify({
    model: 'o4-mini',
    input: 'Знайди помилку в цьому SQL запиті...',
    reasoning: { effort: 'high' }, // 'low' | 'medium' | 'high'
  }),
});

// Anthropic — extended thinking
const response = await fetch('https://api.anthropic.com/v1/messages', {
  headers: { 'x-api-key': apiKey, 'anthropic-version': '2023-06-01', 'Content-Type': 'application/json' },
  method: 'POST',
  body: JSON.stringify({
    model: 'claude-sonnet-4-5-20250929',
    max_tokens: 16000,
    thinking: { type: 'enabled', budget_tokens: 4096 }, // мін. 1024
    messages: [{ role: 'user', content: 'Проаналізуй цей контракт на ризики...' }],
  }),
});
// Відповідь містить thinking blocks + text blocks
```

Reasoning tokens — це реальні output-токени які коштують грошей. Вмикайте тільки для складних задач (аналіз, дебаг, планування). Детальніше: [Reasoning Models](reasoning-models.md).

---

## 2.3 Те ж саме через Vercel AI SDK (набагато зручніше)

```typescript
// ai-sdk.ts — з AI SDK
import 'dotenv/config';
import { generateText } from 'ai';
import { openai } from '@ai-sdk/openai';

async function main() {
  const { text, usage, finishReason } = await generateText({
    model: openai('gpt-4o-mini'),
    system: 'Відповідай коротко, одним реченням.',
    prompt: 'Що таке Promise в JavaScript?',
    temperature: 0.3,
  });

  console.log('Відповідь:', text);
  console.log('Токени:', usage);
  console.log('Причина зупинки:', finishReason);
}

main();
```

**Що дає AI SDK порівняно з голим fetch:**
- Однаковий код для OpenAI, Anthropic, Google (міняєте лише `model`)
- Типізація TypeScript з коробки
- Стрімінг, structured output, tool calling — одним API
- Обробка помилок і retry вже вбудовані

---

## 2.4 Один код — будь-який провайдер

Головна перевага AI SDK — **провайдер-агностичний код**:

```typescript
import 'dotenv/config';
import { generateText } from 'ai';
import { openai } from '@ai-sdk/openai';
import { anthropic } from '@ai-sdk/anthropic';
import { google } from '@ai-sdk/google';

// Одна і та ж функція, різні моделі
async function askQuestion(question: string) {
  const models = [
    { name: 'GPT-4o-mini', model: openai('gpt-4o-mini') },
    { name: 'Claude Haiku', model: anthropic('claude-haiku-4-5-20251001') },
    { name: 'Gemini Flash', model: google('gemini-2.5-flash-preview-04-17') },
  ];

  for (const { name, model } of models) {
    try {
      const { text, usage } = await generateText({
        model,
        prompt: question,
        temperature: 0,
      });
      console.log(`[${name}] ${text}`);
      console.log(`  Токенів: ${usage.totalTokens}\n`);
    } catch (error) {
      console.log(`[${name}] Помилка: ${error.message}\n`);
    }
  }
}

askQuestion('Поясни різницю між let і const в JavaScript одним реченням.');
```

---

## 2.5 Обробка помилок

LLM API можуть відмовити з різних причин. Ось як це правильно обробляти:

```typescript
import { generateText, APICallError } from 'ai';
import { openai } from '@ai-sdk/openai';

async function safeGenerate(prompt: string): Promise<string> {
  try {
    const { text } = await generateText({
      model: openai('gpt-4o-mini'),
      prompt,
      maxRetries: 3,  // AI SDK автоматично повторює при 429/500
    });
    return text;
  } catch (error) {
    if (error instanceof APICallError) {
      switch (error.statusCode) {
        case 401:
          throw new Error('Невалідний API ключ. Перевірте .env файл.');
        case 429:
          throw new Error('Перевищено ліміт запитів. Зачекайте та спробуйте знову.');
        case 500:
          throw new Error('Помилка на стороні провайдера. Спробуйте іншу модель.');
        default:
          throw new Error(`API помилка ${error.statusCode}: ${error.message}`);
      }
    }
    throw error;
  }
}

// Використання
const result = await safeGenerate('Привіт!');
```

### Типові помилки та їх причини

| Помилка | Причина | Рішення |
|---------|---------|---------|
| `401 Unauthorized` | Невірний або прострочений ключ | Перегенеруйте ключ |
| `429 Rate limit` | Занадто багато запитів | Додайте затримку між запитами |
| `400 context_length_exceeded` | Запит + відповідь > контекстне вікно | Скоротіть вхідний текст |
| `500 Internal Server Error` | Проблема у провайдера | Спробуйте іншу модель/провайдера |
| `insufficient_quota` | Закінчились кредити | Поповніть баланс |

---

## 2.6 Практична задача: Простий чатбот в терміналі

Зберіть все разом — створіть інтерактивний чатбот:

```typescript
// chat.ts — мінімальний чатбот з пам'яттю
import 'dotenv/config';
import { generateText, CoreMessage } from 'ai';
import { openai } from '@ai-sdk/openai';
import * as readline from 'readline';

const rl = readline.createInterface({
  input: process.stdin,
  output: process.stdout,
});

const messages: CoreMessage[] = [
  {
    role: 'system',
    content: `Ти — дружній AI-асистент для TypeScript-розробників.
Відповідай коротко та по суті. Якщо питання про код — показуй приклади.
Мова: українська.`,
  },
];

let totalTokens = 0;
let totalCost = 0;

async function chat() {
  console.log('🤖 Чатбот запущено. Напишіть "exit" для виходу.\n');

  const ask = () => {
    rl.question('Ви: ', async (input) => {
      if (input.toLowerCase() === 'exit') {
        console.log(`\n📊 Статистика сесії:`);
        console.log(`   Токенів використано: ${totalTokens}`);
        console.log(`   Орієнтовна вартість: $${totalCost.toFixed(6)}`);
        rl.close();
        return;
      }

      messages.push({ role: 'user', content: input });

      try {
        const { text, usage } = await generateText({
          model: openai('gpt-4o-mini'),
          messages,
          temperature: 0.5,
        });

        messages.push({ role: 'assistant', content: text });
        totalTokens += usage.totalTokens;
        totalCost += (usage.promptTokens * 0.15 + usage.completionTokens * 0.6) / 1_000_000;

        console.log(`\n🤖: ${text}`);
        console.log(`   [${usage.totalTokens} токенів | $${((usage.promptTokens * 0.15 + usage.completionTokens * 0.6) / 1_000_000).toFixed(6)}]\n`);
      } catch (error) {
        console.log(`\n❌ Помилка: ${error.message}\n`);
      }

      ask();
    });
  };

  ask();
}

chat();
```

```bash
npx tsx chat.ts
```

**Зверніть увагу:** з кожним повідомленням вартість зростає, бо ми відправляємо ВСЮ історію. Це важливе розуміння для оптимізації в наступних модулях.

---

## Перевір себе

1. Які три ролі є в Chat API? Яка різниця між ними?
2. Чому з кожним повідомленням в чаті зростає вартість запиту?
3. Що означає `finish_reason: "length"` у відповіді API?
4. Напишіть код який відправляє один і той же запит до OpenAI і Anthropic і порівнює відповіді
5. Запустіть чатбот з практичної задачі і поспілкуйтесь 10 повідомлень. Скільки коштувала сесія?

---

**Назад:** [← Модуль 01 — Базові поняття](01-basics.md) | **Далі:** [Модуль 03 — Vercel AI SDK →](03-ai-sdk.md)
