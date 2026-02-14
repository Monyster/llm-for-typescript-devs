# Модуль 07: Стрімінг та UI — Реальний час у браузері

## 🎯 Що ви отримаєте з цього модуля

Після проходження ви будете:
- Розуміти як працює стрімінг відповідей LLM (SSE, ReadableStream)
- Будувати серверний endpoint для стрімінгу
- Створювати chat UI з `useChat` хуком у React
- Відображати tool calls, loading states та помилки в реальному часі

**Які задачі це дозволяє вирішувати:** Побудувати повноцінний chat-інтерфейс як у ChatGPT/Claude. Показувати відповіді по мірі генерації замість очікування повної відповіді (що може тривати 10-30 секунд).

---

## 💰 Бізнес-цінність

**Проблема клієнта:** "Користувачі чекають 10 секунд на відповідь і закривають чат."
**Рішення з цього модуля:** Стрімінг показує відповідь в реальному часі, UX як у ChatGPT.
**Як продати:** Стрімінг — базова вимога будь-якого чат-інтерфейсу. Без нього продукт виглядає аматорським. Включайте в кожний проект.

## 7.1 Навіщо стрімінг

Без стрімінгу користувач чекає 5-30 секунд поки модель згенерує повну відповідь. Зі стрімінгом — перші слова з'являються через 200-500мс.

```
Без стрімінгу:
[Запит] ................ 15 секунд ................ [Вся відповідь одразу]

Зі стрімінгом:
[Запит] .200мс. [Перше] [слово] [за] [словом] [з'являється] [миттєво]
```

### Як це працює технічно

LLM генерує відповідь **по одному токену** (~50-150 токенів/секунду). Стрімінг відправляє кожен токен клієнту через **Server-Sent Events (SSE)** — однонаправлений HTTP-потік.

```
Сервер → Клієнт (SSE):
data: {"type":"text-delta","textDelta":"Привіт"}
data: {"type":"text-delta","textDelta":", "}
data: {"type":"text-delta","textDelta":"я "}
data: {"type":"text-delta","textDelta":"AI"}
data: {"type":"text-delta","textDelta":"-асистент."}
data: {"type":"finish","finishReason":"stop"}
```

---

## 7.2 Стрімінг на сервері (Node.js / Express)

### Мінімальний приклад з Express

```typescript
// server.ts
import express from 'express';
import { streamText } from 'ai';
import { openai } from '@ai-sdk/openai';

const app = express();
app.use(express.json());

app.post('/api/chat', async (req, res) => {
  const { messages } = req.body;

  const result = streamText({
    model: openai('gpt-4o-mini'),
    system: 'Ти — корисний AI-асистент. Відповідай українською.',
    messages,
  });

  // AI SDK автоматично конвертує в правильний SSE формат
  result.pipeDataStreamToResponse(res);
});

app.listen(3000, () => console.log('Server running on :3000'));
```

### З Next.js App Router

```typescript
// app/api/chat/route.ts
import { streamText } from 'ai';
import { openai } from '@ai-sdk/openai';

export async function POST(req: Request) {
  const { messages } = await req.json();

  const result = streamText({
    model: openai('gpt-4o-mini'),
    system: 'Ти — корисний AI-асистент.',
    messages,
  });

  return result.toDataStreamResponse();
}
```

### Стрімінг під капотом: SSE у різних провайдерів

AI SDK абстрагує стрімінг, але кожен провайдер відправляє SSE-чанки **в різному форматі**:

```typescript
// OpenAI SSE — кожен чанк має поле delta
// data: {"choices":[{"delta":{"content":"Привіт"},"index":0}]}
// data: {"choices":[{"delta":{"content":" світ"},"index":0}]}
// data: [DONE]

// Anthropic SSE — event-based, різні типи подій
// event: message_start
// data: {"type":"message_start","message":{"id":"msg_..."}}
// event: content_block_delta
// data: {"type":"content_block_delta","delta":{"type":"text_delta","text":"Привіт"}}
// event: content_block_delta
// data: {"type":"content_block_delta","delta":{"type":"text_delta","text":" світ"}}
// event: message_stop

// Google Gemini SSE — масив candidates
// data: {"candidates":[{"content":{"parts":[{"text":"Привіт"}]}}]}
// data: {"candidates":[{"content":{"parts":[{"text":" світ"}]}}]}
```

Якщо вам потрібен raw streaming **без AI SDK**:

```typescript
// OpenAI streaming — напряму
async function streamOpenAI(prompt: string) {
  const response = await fetch('https://api.openai.com/v1/chat/completions', {
    method: 'POST',
    headers: { 'Authorization': `Bearer ${OPENAI_KEY}`, 'Content-Type': 'application/json' },
    body: JSON.stringify({
      model: 'gpt-4o-mini',
      messages: [{ role: 'user', content: prompt }],
      stream: true,  // ← Увімкнути стрімінг
    }),
  });

  const reader = response.body!.getReader();
  const decoder = new TextDecoder();

  while (true) {
    const { done, value } = await reader.read();
    if (done) break;
    const text = decoder.decode(value);
    // Парсимо SSE: "data: {...}\n\n"
    for (const line of text.split('\n')) {
      if (line.startsWith('data: ') && line !== 'data: [DONE]') {
        const json = JSON.parse(line.slice(6));
        process.stdout.write(json.choices[0]?.delta?.content ?? '');
      }
    }
  }
}

// Anthropic streaming — інший протокол
async function streamAnthropic(prompt: string) {
  const response = await fetch('https://api.anthropic.com/v1/messages', {
    method: 'POST',
    headers: {
      'x-api-key': ANTHROPIC_KEY,
      'anthropic-version': '2023-06-01',
      'Content-Type': 'application/json',
    },
    body: JSON.stringify({
      model: 'claude-sonnet-4-5-20250929',
      max_tokens: 1024,
      stream: true,  // ← Увімкнути стрімінг
      messages: [{ role: 'user', content: prompt }],
    }),
  });

  const reader = response.body!.getReader();
  const decoder = new TextDecoder();

  while (true) {
    const { done, value } = await reader.read();
    if (done) break;
    const text = decoder.decode(value);
    for (const line of text.split('\n')) {
      if (line.startsWith('data: ')) {
        const json = JSON.parse(line.slice(6));
        // Anthropic: тільки content_block_delta містить текст
        if (json.type === 'content_block_delta') {
          process.stdout.write(json.delta.text ?? '');
        }
      }
    }
  }
}
```

| Аспект | OpenAI | Anthropic | Google |
|--------|--------|-----------|--------|
| Увімкнення | `stream: true` | `stream: true` | `streamGenerateContent` endpoint |
| Формат чанків | `choices[0].delta.content` | `delta.text` (в `content_block_delta`) | `candidates[0].content.parts[0].text` |
| Кінець стріму | `data: [DONE]` | `event: message_stop` | Потік закривається |
| Event types | Немає | `message_start`, `content_block_delta`, `message_stop` та ін. | Немає |

**AI SDK обробляє все це за вас** — ви просто пишете `streamText({ model: ... })` і отримуєте уніфікований потік.

---

## 7.3 Chat UI з React (useChat)

AI SDK має React-хук `useChat` який робить 90% роботи за вас:

```tsx
// components/Chat.tsx
'use client';
import { useChat } from '@ai-sdk/react';

export function Chat() {
  const { messages, input, handleInputChange, handleSubmit, isLoading, error } = useChat({
    api: '/api/chat',  // Ваш серверний endpoint
  });

  return (
    <div className="flex flex-col h-screen max-w-2xl mx-auto p-4">
      {/* Повідомлення */}
      <div className="flex-1 overflow-y-auto space-y-4">
        {messages.map((msg) => (
          <div
            key={msg.id}
            className={`p-3 rounded-lg ${
              msg.role === 'user'
                ? 'bg-blue-100 ml-auto max-w-[80%]'
                : 'bg-gray-100 mr-auto max-w-[80%]'
            }`}
          >
            <p className="text-sm font-medium">
              {msg.role === 'user' ? 'Ви' : 'AI'}
            </p>
            <p className="whitespace-pre-wrap">{msg.content}</p>
          </div>
        ))}

        {isLoading && (
          <div className="bg-gray-100 p-3 rounded-lg animate-pulse">
            AI друкує...
          </div>
        )}

        {error && (
          <div className="bg-red-100 p-3 rounded-lg text-red-700">
            Помилка: {error.message}
          </div>
        )}
      </div>

      {/* Форма вводу */}
      <form onSubmit={handleSubmit} className="flex gap-2 pt-4">
        <input
          value={input}
          onChange={handleInputChange}
          placeholder="Напишіть повідомлення..."
          className="flex-1 p-2 border rounded-lg"
          disabled={isLoading}
        />
        <button
          type="submit"
          disabled={isLoading || !input.trim()}
          className="px-4 py-2 bg-blue-500 text-white rounded-lg disabled:opacity-50"
        >
          Надіслати
        </button>
      </form>
    </div>
  );
}
```

### Що useChat робить автоматично

- Зберігає історію повідомлень (messages)
- Стрімить відповідь по токенах
- Оновлює UI в реальному часі
- Обробляє loading і error стани
- Правильно форматує запити до сервера

---

## 7.4 Стрімінг Structured Output

Можна стрімити навіть структуровані об'єкти:

```tsx
// На сервері
import { streamObject } from 'ai';

app.post('/api/analyze', async (req, res) => {
  const result = streamObject({
    model: openai('gpt-4o-mini'),
    schema: AnalysisSchema,
    prompt: req.body.text,
  });
  result.pipeTextStreamToResponse(res);
});

// На клієнті
import { useObject } from '@ai-sdk/react';

function AnalysisPanel() {
  const { object, submit, isLoading } = useObject({
    api: '/api/analyze',
    schema: AnalysisSchema,
  });

  return (
    <div>
      <button onClick={() => submit({ text: inputText })}>Аналізувати</button>

      {/* Поля з'являються поступово по мірі генерації */}
      {object && (
        <div>
          <p>Категорія: {object.category ?? '...'}</p>
          <p>Пріоритет: {object.priority ?? '...'}</p>
          <p>Опис: {object.summary ?? '...'}</p>
        </div>
      )}
    </div>
  );
}
```

---

## 7.5 Відображення Tool Calls в UI

Коли агент викликає інструменти, це теж можна показувати:

```tsx
import { useChat } from '@ai-sdk/react';

function AgentChat() {
  const { messages, input, handleInputChange, handleSubmit } = useChat({
    api: '/api/agent',
    maxSteps: 5,
  });

  return (
    <div>
      {messages.map((msg) => (
        <div key={msg.id}>
          {msg.role === 'user' && <p>Ви: {msg.content}</p>}
          {msg.role === 'assistant' && (
            <>
              {/* Показуємо tool calls */}
              {msg.toolInvocations?.map((tool, i) => (
                <div key={i} className="bg-yellow-50 p-2 rounded text-sm my-1">
                  🔧 {tool.toolName}({JSON.stringify(tool.args)})
                  {tool.state === 'result' && (
                    <span className="text-green-600"> ✓</span>
                  )}
                </div>
              ))}
              {/* Текстова відповідь */}
              {msg.content && <p>AI: {msg.content}</p>}
            </>
          )}
        </div>
      ))}
    </div>
  );
}
```

---

## Перевір себе

1. Чому стрімінг критично важливий для UX чатбота?
2. Що таке SSE (Server-Sent Events) і чим відрізняється від WebSocket?
3. Які методи AI SDK використовуються для стрімінгу на сервері?
4. Що робить хук `useChat` і які стани він надає?
5. Як відобразити tool calls у UI під час стрімінгу?

---

**Назад:** [← Модуль 06 — Function Calling](06-function-calling.md) | **Далі:** [Модуль 08 — AI Агенти →](08-agents.md)
