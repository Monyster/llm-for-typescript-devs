# Модуль 22: AI SDK UI глибше — useChat, Next.js, Generative UI

## 🎯 Що ви отримаєте з цього модуля

Після проходження ви будете:
- Використовувати useChat, useCompletion, useObject хуки в React
- Інтегрувати AI SDK з Next.js App Router (Server Actions, Route Handlers)
- Будувати Generative UI — коли LLM повертає React-компоненти
- Реалізовувати chat UI з tool calls, loading states, error handling

**Які задачі це дозволяє вирішувати:** Побудувати повноцінний chat-інтерфейс у вашому продукті: стрімінг відповідей, відображення tool calls як UI-карток, генерація інтерактивних компонентів.

---

## 22.1 useChat — Chat-інтерфейс за 10 хвилин

### Серверна частина (Next.js App Router)

```typescript
// app/api/chat/route.ts
import { streamText } from 'ai';
import { openai } from '@ai-sdk/openai';

export async function POST(req: Request) {
  const { messages } = await req.json();

  const result = streamText({
    model: openai('gpt-4o-mini'),
    system: 'Ти — дружній AI-асистент. Відповідай українською.',
    messages,
  });

  return result.toDataStreamResponse();
}
```

### Клієнтська частина (React)

```tsx
// app/page.tsx
'use client';
import { useChat } from '@ai-sdk/react';

export default function ChatPage() {
  const { messages, input, handleInputChange, handleSubmit, isLoading, error } = useChat({
    api: '/api/chat',
  });

  return (
    <div className="flex flex-col h-screen max-w-2xl mx-auto p-4">
      {/* Повідомлення */}
      <div className="flex-1 overflow-y-auto space-y-4">
        {messages.map((msg) => (
          <div
            key={msg.id}
            className={`p-3 rounded-lg ${
              msg.role === 'user' ? 'bg-blue-100 ml-auto max-w-[80%]' : 'bg-gray-100 max-w-[80%]'
            }`}
          >
            {msg.content}
          </div>
        ))}
        {isLoading && (
          <div className="bg-gray-100 p-3 rounded-lg animate-pulse">Думаю...</div>
        )}
      </div>

      {/* Помилка */}
      {error && (
        <div className="bg-red-100 text-red-700 p-2 rounded mb-2">
          Помилка: {error.message}
        </div>
      )}

      {/* Форма вводу */}
      <form onSubmit={handleSubmit} className="flex gap-2 mt-4">
        <input
          value={input}
          onChange={handleInputChange}
          placeholder="Напишіть повідомлення..."
          className="flex-1 border rounded-lg p-2"
          disabled={isLoading}
        />
        <button
          type="submit"
          disabled={isLoading || !input.trim()}
          className="bg-blue-500 text-white px-4 py-2 rounded-lg disabled:opacity-50"
        >
          Надіслати
        </button>
      </form>
    </div>
  );
}
```

### Ключові можливості useChat

```tsx
const {
  messages,           // Масив повідомлень
  input,              // Поточний текст у полі вводу
  handleInputChange,  // onChange для input
  handleSubmit,       // onSubmit для form
  isLoading,          // Чи чекаємо відповідь
  error,              // Помилка (якщо є)
  append,             // Програмно додати повідомлення
  reload,             // Повторити останній запит
  stop,               // Зупинити генерацію
  setMessages,        // Замінити всі повідомлення
} = useChat({
  api: '/api/chat',
  initialMessages: [],           // Попередні повідомлення
  onFinish: (message) => {},     // Коли стрім завершено
  onError: (error) => {},        // Обробка помилок
  maxSteps: 5,                   // Дозволити tool calls
});
```

---

## 22.2 useObject — Стрімінг структурованих даних

Для задач де LLM генерує об'єкт (не текст):

### Сервер

```typescript
// app/api/analyze/route.ts
import { streamObject } from 'ai';
import { openai } from '@ai-sdk/openai';
import { z } from 'zod';

const AnalysisSchema = z.object({
  sentiment: z.enum(['positive', 'negative', 'neutral']),
  topics: z.array(z.string()),
  actionItems: z.array(z.object({
    task: z.string(),
    priority: z.enum(['low', 'medium', 'high']),
  })),
  summary: z.string(),
});

export async function POST(req: Request) {
  const { text } = await req.json();

  const result = streamObject({
    model: openai('gpt-4o-mini'),
    schema: AnalysisSchema,
    prompt: `Проаналізуй цей текст:\n${text}`,
  });

  return result.toTextStreamResponse();
}
```

### Клієнт

```tsx
'use client';
import { useObject } from '@ai-sdk/react';
import { z } from 'zod';

// Та ж схема що на сервері
const AnalysisSchema = z.object({
  sentiment: z.enum(['positive', 'negative', 'neutral']),
  topics: z.array(z.string()),
  actionItems: z.array(z.object({
    task: z.string(),
    priority: z.enum(['low', 'medium', 'high']),
  })),
  summary: z.string(),
});

export default function AnalyzerPage() {
  const { object, submit, isLoading } = useObject({
    api: '/api/analyze',
    schema: AnalysisSchema,
  });

  return (
    <div>
      <button onClick={() => submit({ text: reviewText })}>Аналізувати</button>

      {/* object заповнюється поступово по мірі стрімінгу */}
      {object && (
        <div>
          {object.sentiment && <p>Тональність: {object.sentiment}</p>}
          {object.topics && <p>Теми: {object.topics.join(', ')}</p>}
          {object.actionItems?.map((item, i) => (
            <div key={i}>📋 [{item.priority}] {item.task}</div>
          ))}
          {object.summary && <p>Підсумок: {object.summary}</p>}
        </div>
      )}
    </div>
  );
}
```

---

## 22.3 Tool Calls в UI

Відображення tool calls як інтерактивних карток:

### Сервер з tools

```typescript
// app/api/chat/route.ts
import { streamText, tool } from 'ai';
import { openai } from '@ai-sdk/openai';
import { z } from 'zod';

export async function POST(req: Request) {
  const { messages } = await req.json();

  const result = streamText({
    model: openai('gpt-4o-mini'),
    messages,
    tools: {
      getWeather: tool({
        description: 'Отримати погоду для міста',
        parameters: z.object({
          city: z.string(),
        }),
        execute: async ({ city }) => {
          const data = await fetchWeather(city);
          return { city, temp: data.temp, condition: data.condition, icon: data.icon };
        },
      }),
      searchProducts: tool({
        description: 'Пошук товарів',
        parameters: z.object({ query: z.string(), maxPrice: z.number().optional() }),
        execute: async ({ query, maxPrice }) => {
          return await productSearch(query, maxPrice);
        },
      }),
    },
    maxSteps: 5,
  });

  return result.toDataStreamResponse();
}
```

### Клієнт з рендерингом tool results

```tsx
'use client';
import { useChat } from '@ai-sdk/react';

// Компоненти для відображення результатів tools
function WeatherCard({ data }: { data: any }) {
  return (
    <div className="bg-blue-50 p-3 rounded-lg border">
      <h3 className="font-bold">{data.city}</h3>
      <p className="text-2xl">{data.temp}°C {data.icon}</p>
      <p className="text-gray-600">{data.condition}</p>
    </div>
  );
}

function ProductList({ data }: { data: any[] }) {
  return (
    <div className="grid grid-cols-2 gap-2">
      {data.map((p, i) => (
        <div key={i} className="border p-2 rounded">
          <p className="font-bold">{p.name}</p>
          <p className="text-green-600">{p.price} грн</p>
        </div>
      ))}
    </div>
  );
}

export default function ChatWithTools() {
  const { messages, input, handleInputChange, handleSubmit } = useChat({
    api: '/api/chat',
    maxSteps: 5,
  });

  return (
    <div>
      {messages.map((msg) => (
        <div key={msg.id}>
          {/* Текстовий контент */}
          {msg.content && <p>{msg.content}</p>}

          {/* Tool calls та їх результати */}
          {msg.toolInvocations?.map((tool, i) => (
            <div key={i} className="my-2">
              {tool.state === 'result' && tool.toolName === 'getWeather' && (
                <WeatherCard data={tool.result} />
              )}
              {tool.state === 'result' && tool.toolName === 'searchProducts' && (
                <ProductList data={tool.result} />
              )}
              {tool.state === 'call' && (
                <div className="animate-pulse text-gray-400">
                  🔄 Виконую {tool.toolName}...
                </div>
              )}
            </div>
          ))}
        </div>
      ))}

      <form onSubmit={handleSubmit}>
        <input value={input} onChange={handleInputChange} />
        <button type="submit">Надіслати</button>
      </form>
    </div>
  );
}
```

---

## 22.4 Generative UI — LLM повертає React-компоненти

AI SDK 6 дозволяє моделі "обирати" який UI-компонент показати:

```typescript
// app/api/chat/route.ts
import { streamUI } from 'ai/rsc';
import { openai } from '@ai-sdk/openai';
import { z } from 'zod';

export async function submitMessage(userMessage: string) {
  'use server';

  const result = await streamUI({
    model: openai('gpt-4o-mini'),
    system: 'Ти — асистент магазину. Використовуй UI-компоненти для відображення даних.',
    messages: [{ role: 'user', content: userMessage }],
    tools: {
      showProductCard: {
        description: 'Показати картку товару',
        parameters: z.object({
          name: z.string(),
          price: z.number(),
          image: z.string().url(),
          rating: z.number(),
        }),
        generate: async function* ({ name, price, image, rating }) {
          yield <div className="animate-pulse bg-gray-200 h-48 rounded" />;
          // LLM бачить параметри, а користувач бачить React-компонент
          return (
            <div className="border rounded-lg p-4 shadow-md">
              <img src={image} alt={name} className="w-full h-48 object-cover rounded" />
              <h3 className="text-lg font-bold mt-2">{name}</h3>
              <p className="text-green-600 text-xl">{price} грн</p>
              <p>{'⭐'.repeat(Math.round(rating))} ({rating})</p>
              <button className="bg-blue-500 text-white w-full py-2 rounded mt-2">
                Додати в кошик
              </button>
            </div>
          );
        },
      },
    },
  });

  return result.value;
}
```

---

## 22.5 Корисні паттерни

### Оптимістичні повідомлення

```tsx
const { append } = useChat();

// Показуємо повідомлення одразу, не чекаючи відповіді
function sendQuickReply(text: string) {
  append({ role: 'user', content: text });
}

// Кнопки швидких дій
<div className="flex gap-2">
  <button onClick={() => sendQuickReply('Де моє замовлення?')}>📦 Замовлення</button>
  <button onClick={() => sendQuickReply('Хочу повернути товар')}>↩️ Повернення</button>
</div>
```

### Зупинка генерації

```tsx
const { stop, isLoading } = useChat();

{isLoading && (
  <button onClick={stop} className="text-red-500">
    ⏹ Зупинити
  </button>
)}
```

---

## Перевір себе

1. Чим useChat відрізняється від useObject? Коли що використовувати?
2. Як відобразити tool calls як UI-картки в чаті?
3. Що таке Generative UI і яку проблему воно вирішує?
4. Побудуйте chat UI з useChat який підтримує мінімум один tool
5. Як реалізувати кнопку "Зупинити генерацію"?

---

**Назад:** [← Модуль 21 — Voice та Realtime](21-voice-realtime.md) | **Далі:** [Модуль 23 — Mastra та інші фреймворки →](23-mastra.md)
