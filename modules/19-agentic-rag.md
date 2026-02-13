# Модуль 19: Agentic RAG — Розумний пошук по даних

## 🎯 Що ви отримаєте з цього модуля

Після проходження ви будете:
- Розуміти обмеження "наївного" RAG та як їх подолати
- Реалізовувати query decomposition — розбиття складних запитів на прості
- Використовувати re-ranking для покращення якості результатів
- Будувати multi-hop retrieval — пошук що вимагає кількох кроків
- Створювати агента який сам вирішує коли, що і де шукати

**Які задачі це дозволяє вирішувати:** Перейти від "знайди схожий текст" до "знайди відповідь на складне питання, навіть якщо вона розкидана по десятках документів". Agentic RAG — це різниця між ctrl+F та досвідченим аналітиком.

---

## 19.1 Проблеми наївного RAG

Модуль 10 навчив вас базовому RAG: embed query → знайти схожі chunks → передати в LLM. Це працює для простих питань, але ламається на складних:

### Проблема 1: Неточний запит

```
Користувач: "Порівняй наші умови повернення з конкурентами"

Наївний RAG: шукає chunks схожі на "умови повернення конкуренти"
→ Знаходить загальну інформацію про повернення
→ НЕ знаходить дані про конкурентів (їх немає в базі)
→ Відповідь неповна або галюцинація
```

### Проблема 2: Відповідь розкидана по документах

```
Користувач: "Яка загальна сума всіх контрактів за Q3?"

Наївний RAG: знаходить 3 chunks з окремими контрактами
→ Але пропускає інші 7 контрактів
→ Відповідь: "$150,000" (насправді $480,000)
```

### Проблема 3: Потрібен контекст з різних джерел

```
Користувач: "Чому клієнт X відмовився від продовження?"

Потрібно знайти:
1. Історію листування з клієнтом (email)
2. Тікети підтримки клієнта (helpdesk)
3. Записи зустрічей (meeting notes)
4. Дані про використання продукту (analytics)
→ Наївний RAG шукає в одному місці
```

---

## 19.2 Agentic RAG: Агент керує пошуком

Замість одного пошуку — агент з tool-ами який **сам вирішує** стратегію пошуку:

```
┌──────────────────────────────────────────────────┐
│              Agentic RAG                          │
│                                                  │
│  Запит → Агент (LLM) ─┬→ decompose_query        │
│              ↕         ├→ search_documents        │
│         Оцінка         ├→ search_database         │
│         результатів    ├→ search_web              │
│              ↕         ├→ rerank_results           │
│         Достатньо? ────├→ ask_clarification       │
│           ↓ Так        └→ summarize_findings      │
│      Фінальна відповідь                          │
└──────────────────────────────────────────────────┘
```

```typescript
import { generateText, tool } from 'ai';
import { openai } from '@ai-sdk/openai';
import { z } from 'zod';

const { text } = await generateText({
  model: openai('gpt-4o-mini'),
  system: `Ти — дослідницький агент. Використовуй інструменти щоб знайти повну відповідь.

Стратегія:
1. Проаналізуй запит — чи потрібно його розбити на підзапити?
2. Шукай у відповідних джерелах
3. Оціни якість знайденого — якщо недостатньо, шукай ще
4. Сформуй відповідь ТІЛЬКИ на основі знайденого`,

  tools: {
    decomposeQuery: tool({
      description: 'Розбити складний запит на простіші підзапити для пошуку',
      parameters: z.object({
        originalQuery: z.string(),
        subQueries: z.array(z.string()).max(5),
      }),
      execute: async ({ subQueries }) => {
        return { subQueries, message: 'Тепер шукай по кожному підзапиту окремо' };
      },
    }),

    searchDocuments: tool({
      description: 'Семантичний пошук по корпоративних документах (wiki, docs, guides)',
      parameters: z.object({
        query: z.string(),
        collection: z.enum(['all', 'policies', 'technical', 'sales', 'hr']),
        limit: z.number().default(5),
      }),
      execute: async ({ query, collection, limit }) => {
        return await vectorSearch(query, { collection, limit });
      },
    }),

    searchTickets: tool({
      description: 'Пошук по тікетах підтримки та листуванню з клієнтами',
      parameters: z.object({
        query: z.string(),
        customerId: z.string().optional(),
        dateRange: z.enum(['last_week', 'last_month', 'last_quarter', 'all']).default('all'),
      }),
      execute: async ({ query, customerId, dateRange }) => {
        return await ticketSearch(query, { customerId, dateRange });
      },
    }),

    evaluateResults: tool({
      description: 'Оцінити чи знайдені результати достатні для відповіді',
      parameters: z.object({
        question: z.string(),
        foundInfo: z.string(),
        confidence: z.number().min(0).max(1),
        missingInfo: z.array(z.string()),
      }),
      execute: async ({ confidence, missingInfo }) => {
        if (confidence < 0.7) {
          return { sufficient: false, suggestion: `Потрібно дошукати: ${missingInfo.join(', ')}` };
        }
        return { sufficient: true };
      },
    }),
  },

  maxSteps: 10,
  prompt: userQuestion,
});
```

---

## 19.3 Query Decomposition — розбиття складних запитів

Ключова техніка: перетворити один складний запит на кілька простих.

```typescript
import { generateObject } from 'ai';
import { z } from 'zod';

async function decomposeQuery(query: string) {
  const { object } = await generateObject({
    model: openai('gpt-4o-mini'),
    schema: z.object({
      isComplex: z.boolean().describe('Чи потребує запит декомпозиції'),
      subQueries: z.array(z.object({
        query: z.string(),
        source: z.enum(['documents', 'database', 'api', 'web']),
        priority: z.enum(['required', 'optional']),
      })),
      strategy: z.enum(['parallel', 'sequential']).describe('parallel якщо підзапити незалежні'),
    }),
    temperature: 0,
    prompt: `Проаналізуй запит і визнач чи його потрібно розбити на підзапити.

Запит: "${query}"

Якщо запит простий — isComplex: false і один subQuery.
Якщо складний — розбий на 2-5 конкретних підзапитів.`,
  });

  return object;
}

// Приклад
const result = await decomposeQuery(
  'Порівняй продуктивність команди за Q2 та Q3 і поясни чому клієнт Acme відмовився від продовження'
);

// {
//   isComplex: true,
//   subQueries: [
//     { query: "продуктивність команди Q2 метрики", source: "database", priority: "required" },
//     { query: "продуктивність команди Q3 метрики", source: "database", priority: "required" },
//     { query: "клієнт Acme причина відмови", source: "documents", priority: "required" },
//     { query: "клієнт Acme тікети підтримки", source: "database", priority: "optional" },
//   ],
//   strategy: "parallel"
// }
```

---

## 19.4 Re-ranking — покращення якості результатів

Семантичний пошук повертає "схожі" результати, але не завжди "релевантні". Re-ranking переставляє результати за реальною релевантністю:

```
Етап 1 (Vector Search): Швидко знайти 20 кандидатів (cheap, fast)
Етап 2 (Re-ranking): Переранжувати 20 → залишити топ-5 (expensive, accurate)
```

### LLM-based Re-ranking

```typescript
import { generateObject } from 'ai';
import { z } from 'zod';

interface SearchResult {
  id: string;
  text: string;
  score: number;
}

async function rerankWithLLM(
  query: string,
  candidates: SearchResult[],
  topK = 5
): Promise<SearchResult[]> {
  const { object } = await generateObject({
    model: openai('gpt-4o-mini'),
    schema: z.object({
      rankings: z.array(z.object({
        id: z.string(),
        relevanceScore: z.number().min(0).max(10),
        reasoning: z.string(),
      })),
    }),
    temperature: 0,
    prompt: `Оціни релевантність кожного результату до запиту.
    
Запит: "${query}"

Результати:
${candidates.map((c, i) => `[${c.id}] ${c.text.slice(0, 200)}`).join('\n\n')}

Оціни кожен результат від 0 (нерелевантний) до 10 (ідеально відповідає).`,
  });

  // Сортуємо за новим score і беремо topK
  const reranked = object.rankings
    .sort((a, b) => b.relevanceScore - a.relevanceScore)
    .slice(0, topK);

  return reranked.map(r => ({
    ...candidates.find(c => c.id === r.id)!,
    score: r.relevanceScore / 10,
  }));
}
```

### Cross-Encoder Re-ranking (без LLM, швидше)

```typescript
// Використання Cohere Rerank API
async function rerankWithCohere(query: string, documents: string[]): Promise<number[]> {
  const response = await fetch('https://api.cohere.com/v2/rerank', {
    method: 'POST',
    headers: {
      'Authorization': `Bearer ${process.env.COHERE_API_KEY}`,
      'Content-Type': 'application/json',
    },
    body: JSON.stringify({
      model: 'rerank-v3.5',
      query,
      documents,
      top_n: 5,
    }),
  });

  const data = await response.json();
  return data.results.map((r: any) => r.index); // Індекси відсортовані за релевантністю
}
```

---

## 19.5 Multi-Hop Retrieval — пошук у кілька кроків

Деякі питання вимагають послідовного пошуку — результат першого пошуку визначає що шукати далі:

```typescript
async function multiHopSearch(question: string): Promise<string> {
  let context = '';
  let currentQuery = question;

  for (let hop = 0; hop < 3; hop++) {
    // Пошук
    const results = await vectorSearch(currentQuery, { limit: 3 });
    context += '\n\n' + results.map(r => r.text).join('\n');

    // Перевірка: чи достатньо інформації?
    const { object: evaluation } = await generateObject({
      model: openai('gpt-4o-mini'),
      schema: z.object({
        hasAnswer: z.boolean(),
        nextQuery: z.string().nullable().describe('Що ще потрібно знайти, null якщо достатньо'),
      }),
      temperature: 0,
      prompt: `Питання: ${question}
Знайдена інформація: ${context}
Чи достатньо інформації для повної відповіді? Якщо ні — що ще шукати?`,
    });

    if (evaluation.hasAnswer || !evaluation.nextQuery) break;
    currentQuery = evaluation.nextQuery; // Наступний hop
  }

  // Генерація фінальної відповіді
  const { text } = await generateText({
    model: openai('gpt-4o-mini'),
    system: 'Відповідай ТІЛЬКИ на основі наданого контексту. Якщо інформації немає — скажи.',
    prompt: `Контекст:\n${context}\n\nПитання: ${question}`,
  });

  return text;
}
```

---

## 19.6 Повна архітектура Agentic RAG

```typescript
// Зведення всього разом
async function agenticRAG(question: string): Promise<{
  answer: string;
  sources: string[];
  hops: number;
  confidence: number;
}> {
  // 1. Декомпозиція
  const decomposed = await decomposeQuery(question);

  // 2. Пошук по кожному підзапиту
  let allResults: SearchResult[] = [];

  if (decomposed.strategy === 'parallel') {
    const searches = decomposed.subQueries.map(sq =>
      vectorSearch(sq.query, { collection: sq.source, limit: 5 })
    );
    const results = await Promise.all(searches);
    allResults = results.flat();
  } else {
    for (const sq of decomposed.subQueries) {
      const results = await vectorSearch(sq.query, { collection: sq.source, limit: 5 });
      allResults.push(...results);
    }
  }

  // 3. Re-ranking
  const reranked = await rerankWithLLM(question, allResults, 8);

  // 4. Multi-hop якщо потрібно
  // (додатковий пошук на основі знайденого)

  // 5. Генерація відповіді
  const context = reranked.map(r => `[${r.id}] ${r.text}`).join('\n---\n');

  const { text } = await generateText({
    model: openai('gpt-4o-mini'),
    system: `Відповідай на основі контексту. Вказуй джерела у квадратних дужках [id].
Якщо інформації недостатньо — чесно скажи що саме невідомо.`,
    prompt: `Контекст:\n${context}\n\nПитання: ${question}`,
  });

  return {
    answer: text,
    sources: reranked.map(r => r.id),
    hops: decomposed.subQueries.length,
    confidence: reranked[0]?.score ?? 0,
  };
}
```

---

## Перевір себе

1. Назвіть 3 проблеми наївного RAG та як Agentic RAG їх вирішує
2. Що таке query decomposition? Коли parallel, а коли sequential?
3. Чим re-ranking відрізняється від початкового vector search?
4. Реалізуйте multi-hop retrieval для запиту що вимагає 2 кроки пошуку
5. Коли достатньо наївного RAG, а коли потрібен Agentic RAG?

---

**Назад:** [← Модуль 18 — Бізнес](18-business.md) | **Далі:** [Модуль 20 — Fine-tuning та Model Distillation →](20-fine-tuning.md)
