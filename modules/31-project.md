# Модуль 31: Наскрізний проект — AI Customer Support Platform

## 🎯 Що ви отримаєте з цього модуля

Покрокова збірка повноцінної AI-платформи підтримки клієнтів — від `npm init` до деплою. Цей проект об'єднує знання з усього курсу.

---

## 31.1 Архітектура проекту

```
┌─────────────────────────────────────────────────┐
│                  Frontend (Next.js)             │
│   Chat UI (useChat) │ Admin Dashboard           │
├─────────────────────────────────────────────────┤
│                  Backend (API Routes)           │
│   /api/chat │ /api/tickets │ /api/analytics     │
├─────────────────────────────────────────────────┤
│              AI Layer                            │
│   Supervisor Agent → Specialist Agents          │
│   RAG (pgvector) │ Memory │ Tools              │
├─────────────────────────────────────────────────┤
│              Data Layer                          │
│   PostgreSQL │ Redis │ Vector Index             │
└─────────────────────────────────────────────────┘
```

### Технології

- **Frontend:** Next.js 15, AI SDK React (useChat), Tailwind
- **AI:** Vercel AI SDK 6, OpenAI GPT-4o-mini, Claude Sonnet (fallback)
- **Database:** PostgreSQL + pgvector, Redis (кеш, черги)
- **Deploy:** Vercel (frontend) + Railway/Fly.io (DB)

---

## 31.2 Крок 1: Ініціалізація проекту

```bash
npx create-next-app@latest ai-support --typescript --tailwind --app --src-dir
cd ai-support

# AI SDK
npm install ai @ai-sdk/openai @ai-sdk/anthropic

# Database
npm install drizzle-orm postgres
npm install -D drizzle-kit

# Утиліти
npm install zod bullmq ioredis
```

### Структура файлів

```
src/
├── app/
│   ├── api/
│   │   ├── chat/route.ts          # Chat endpoint
│   │   ├── tickets/route.ts       # Ticket management
│   │   └── webhooks/route.ts      # External integrations
│   ├── page.tsx                    # Chat widget
│   └── admin/page.tsx              # Admin dashboard
├── lib/
│   ├── ai/
│   │   ├── agents.ts              # Agent definitions
│   │   ├── tools.ts               # Tool implementations
│   │   ├── prompts.ts             # System prompts
│   │   └── rag.ts                 # RAG pipeline
│   ├── db/
│   │   ├── schema.ts              # Drizzle schema
│   │   └── index.ts               # DB connection
│   └── services/
│       ├── tickets.ts             # Ticket service
│       └── analytics.ts           # Analytics
└── scripts/
    └── seed-knowledge-base.ts     # Index FAQ/docs
```

---

## 31.3 Крок 2: База даних

```typescript
// src/lib/db/schema.ts
import { pgTable, text, timestamp, jsonb, vector, integer } from 'drizzle-orm/pg-core';

export const conversations = pgTable('conversations', {
  id: text('id').primaryKey(),
  userId: text('user_id').notNull(),
  status: text('status').notNull().default('active'), // active, resolved, escalated
  metadata: jsonb('metadata').default({}),
  createdAt: timestamp('created_at').defaultNow(),
});

export const messages = pgTable('messages', {
  id: text('id').primaryKey(),
  conversationId: text('conversation_id').references(() => conversations.id),
  role: text('role').notNull(), // user, assistant, system, tool
  content: text('content').notNull(),
  toolCalls: jsonb('tool_calls'),
  createdAt: timestamp('created_at').defaultNow(),
});

export const knowledgeChunks = pgTable('knowledge_chunks', {
  id: text('id').primaryKey(),
  content: text('content').notNull(),
  source: text('source').notNull(),
  category: text('category'),
  embedding: vector('embedding', { dimensions: 1536 }),
});

export const tickets = pgTable('tickets', {
  id: text('id').primaryKey(),
  conversationId: text('conversation_id').references(() => conversations.id),
  category: text('category').notNull(),
  priority: text('priority').notNull(),
  status: text('status').notNull().default('open'),
  assignedTo: text('assigned_to'),
  resolvedAt: timestamp('resolved_at'),
  createdAt: timestamp('created_at').defaultNow(),
});
```

---

## 31.4 Крок 3: RAG Pipeline

```typescript
// src/lib/ai/rag.ts
import { embed, embedMany } from 'ai';
import { openai } from '@ai-sdk/openai';
import { db } from '../db';
import { knowledgeChunks } from '../db/schema';
import { cosineDistance, desc, sql } from 'drizzle-orm';

export async function indexDocuments(docs: Array<{ content: string; source: string; category: string }>) {
  for (const doc of docs) {
    const chunks = splitIntoChunks(doc.content, 500, 100);
    const { embeddings } = await embedMany({
      model: openai.embedding('text-embedding-3-small'),
      values: chunks,
    });

    for (let i = 0; i < chunks.length; i++) {
      await db.insert(knowledgeChunks).values({
        id: `${doc.source}-${i}`,
        content: chunks[i],
        source: doc.source,
        category: doc.category,
        embedding: embeddings[i],
      });
    }
  }
}

export async function searchKnowledge(query: string, limit = 5) {
  const { embedding } = await embed({
    model: openai.embedding('text-embedding-3-small'),
    value: query,
  });

  const results = await db
    .select({
      content: knowledgeChunks.content,
      source: knowledgeChunks.source,
      similarity: sql<number>`1 - (${knowledgeChunks.embedding} <=> ${JSON.stringify(embedding)})`,
    })
    .from(knowledgeChunks)
    .orderBy(sql`${knowledgeChunks.embedding} <=> ${JSON.stringify(embedding)}`)
    .limit(limit);

  return results.filter(r => r.similarity > 0.7);
}
```

---

## 31.5 Крок 4: AI Agents та Tools

```typescript
// src/lib/ai/tools.ts
import { tool } from 'ai';
import { z } from 'zod';

export const supportTools = {
  searchKnowledgeBase: tool({
    description: 'Пошук відповіді в базі знань компанії (FAQ, документація)',
    parameters: z.object({ query: z.string() }),
    execute: async ({ query }) => {
      const results = await searchKnowledge(query);
      return results.length > 0
        ? results.map(r => `[${r.source}]: ${r.content}`).join('\n\n')
        : 'Нічого не знайдено в базі знань.';
    },
  }),

  checkOrderStatus: tool({
    description: 'Перевірити статус замовлення клієнта',
    parameters: z.object({ orderId: z.string().regex(/^ORD-\d{4,8}$/) }),
    execute: async ({ orderId }) => {
      const order = await orderService.getByID(orderId);
      if (!order) return { error: `Замовлення ${orderId} не знайдено` };
      return { id: order.id, status: order.status, estimatedDelivery: order.eta };
    },
  }),

  createTicket: tool({
    description: 'Створити тікет підтримки для ескалації на людину',
    parameters: z.object({
      summary: z.string(),
      category: z.enum(['billing', 'technical', 'shipping', 'other']),
      priority: z.enum(['low', 'medium', 'high']),
    }),
    execute: async ({ summary, category, priority }) => {
      const ticket = await ticketService.create({ summary, category, priority });
      return { ticketId: ticket.id, message: `Тікет #${ticket.id} створено. Оператор зв'яжеться з вами.` };
    },
  }),
};
```

```typescript
// src/lib/ai/prompts.ts
export const SUPPORT_SYSTEM_PROMPT = `Ти — AI-асистент підтримки TechShop.

ПРАВИЛА:
1. Спочатку шукай відповідь у базі знань (searchKnowledgeBase)
2. Якщо клієнт питає про замовлення — перевір статус (checkOrderStatus)
3. Якщо не можеш вирішити проблему — створи тікет (createTicket)
4. НІКОЛИ не вигадуй інформацію
5. Будь ввічливим, коротким і корисним
6. Відповідай українською`;
```

---

## 31.6 Крок 5: Chat API

```typescript
// src/app/api/chat/route.ts
import { streamText } from 'ai';
import { openai } from '@ai-sdk/openai';
import { supportTools } from '@/lib/ai/tools';
import { SUPPORT_SYSTEM_PROMPT } from '@/lib/ai/prompts';

export async function POST(req: Request) {
  const { messages, conversationId } = await req.json();

  const result = streamText({
    model: openai('gpt-4o-mini'),
    system: SUPPORT_SYSTEM_PROMPT,
    tools: supportTools,
    maxSteps: 5,
    messages,
    onFinish: async ({ text, toolCalls }) => {
      // Зберігаємо в БД для аналітики
      await saveMessage(conversationId, 'assistant', text, toolCalls);
    },
  });

  return result.toDataStreamResponse();
}
```

---

## 31.7 Крок 6: Frontend

```tsx
// src/app/page.tsx
'use client';
import { useChat } from '@ai-sdk/react';

export default function SupportChat() {
  const { messages, input, handleInputChange, handleSubmit, isLoading } = useChat({
    api: '/api/chat',
    maxSteps: 5,
    body: { conversationId: crypto.randomUUID() },
  });

  return (
    <div className="max-w-lg mx-auto h-screen flex flex-col bg-white shadow-xl">
      <header className="bg-blue-600 text-white p-4 text-center font-bold">
        TechShop Support
      </header>

      <div className="flex-1 overflow-y-auto p-4 space-y-3">
        {messages.map((msg) => (
          <div key={msg.id} className={`p-3 rounded-lg max-w-[85%] ${
            msg.role === 'user' ? 'bg-blue-100 ml-auto' : 'bg-gray-100'
          }`}>
            {msg.content}
            {msg.toolInvocations?.map((t, i) => (
              t.state === 'result' && t.toolName === 'createTicket' && (
                <div key={i} className="mt-2 p-2 bg-yellow-50 rounded text-sm">
                  📋 Тікет створено: #{t.result.ticketId}
                </div>
              )
            ))}
          </div>
        ))}
        {isLoading && <div className="bg-gray-100 p-3 rounded-lg animate-pulse">Думаю...</div>}
      </div>

      <form onSubmit={handleSubmit} className="p-4 border-t flex gap-2">
        <input
          value={input}
          onChange={handleInputChange}
          placeholder="Чим можу допомогти?"
          className="flex-1 border rounded-lg p-2"
        />
        <button type="submit" disabled={isLoading} className="bg-blue-600 text-white px-4 rounded-lg">
          ➤
        </button>
      </form>
    </div>
  );
}
```

---

## 31.8 Крок 7: Деплой

```bash
# Vercel (frontend + API)
vercel deploy --prod

# Railway (PostgreSQL + pgvector)
railway add --plugin postgresql

# Змінні середовища
vercel env add OPENAI_API_KEY
vercel env add DATABASE_URL

# Seed knowledge base
npx tsx scripts/seed-knowledge-base.ts
```

### Вартість production (1000 розмов/день)

| Компонент | Вартість/місяць |
|-----------|----------------|
| OpenAI GPT-4o-mini | ~$50 |
| Embedding | ~$5 |
| Vercel Pro | $20 |
| Railway (DB) | $10 |
| **Разом** | **~$85/міс** |

---

## Перевір себе

1. Які компоненти має AI Support Platform?
2. Реалізуйте повний проект з RAG, tools, та chat UI
3. Додайте admin dashboard для перегляду тікетів
4. Налаштуйте fallback на Claude якщо OpenAI недоступний
5. Яка вартість для 10,000 розмов/день?

---

**Назад:** [← Модуль 30 — Use Cases](30-use-cases.md) | **Далі:** [Модуль 32 — AI для DevOps →](32-devops.md)
