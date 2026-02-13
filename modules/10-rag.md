# Модуль 10: RAG — Пошук по власних даних

## 🎯 Що ви отримаєте з цього модуля

Після проходження ви будете:
- Розуміти архітектуру RAG: indexing → retrieval → generation
- Створювати embeddings та зберігати їх у векторній базі
- Реалізовувати chunking стратегії для різних типів документів
- Будувати повний RAG pipeline від документа до відповіді

**Які задачі це дозволяє вирішувати:** Дати LLM доступ до ВАШИХ даних — корпоративна база знань, документація, PDF-файли, внутрішні wiki. Замість загальних знань модель відповідає на основі конкретних документів вашої компанії.

---

## 10.1 Проблема: LLM не знає ваших даних

LLM навчена на публічних даних з інтернету. Вона не знає:
- Внутрішню документацію вашої компанії
- Вашу базу знань підтримки
- Контракти, специфікації, технічні описи
- Дані що з'явились після дати навчання

**RAG (Retrieval Augmented Generation)** — це патерн де ви спочатку **шукаєте** релевантні документи, а потім **передаєте** їх моделі разом із запитом.

```
Без RAG:
  Питання → LLM → Відповідь (з загальних знань, можливо галюцинація)

З RAG:
  Питання → Пошук по вашій базі → Релевантні фрагменти + Питання → LLM → Відповідь
```

---

## 10.2 Архітектура RAG

RAG складається з двох фаз:

### Фаза 1: Indexing (один раз)

```
Документи → Chunking → Embedding → Векторна база
```

1. **Chunking** — розбиваємо документ на фрагменти (chunks)
2. **Embedding** — перетворюємо кожен фрагмент на числовий вектор
3. **Store** — зберігаємо вектори у базу (pgvector, Pinecone, тощо)

### Фаза 2: Retrieval + Generation (кожен запит)

```
Запит → Embedding → Пошук у базі → Топ-N фрагментів → LLM → Відповідь
```

1. **Embed query** — перетворюємо запит користувача на вектор
2. **Search** — шукаємо найсхожіші фрагменти у базі
3. **Generate** — передаємо знайдені фрагменти + запит до LLM

---

## 10.3 Embeddings: Текст → Вектор

Embedding — це числовий вектор (масив чисел) який "кодує" зміст тексту. Схожі тексти мають схожі вектори.

```typescript
import { embed, embedMany, cosineSimilarity } from 'ai';
import { openai } from '@ai-sdk/openai';

// Один текст → один вектор
const { embedding: v1 } = await embed({
  model: openai.embedding('text-embedding-3-small'),
  value: 'Як налаштувати TypeScript проект?',
});
// v1 = [0.023, -0.041, 0.087, ...] — 1536 чисел

const { embedding: v2 } = await embed({
  model: openai.embedding('text-embedding-3-small'),
  value: 'Конфігурація tsconfig.json для нового проекту',
});

const { embedding: v3 } = await embed({
  model: openai.embedding('text-embedding-3-small'),
  value: 'Рецепт борщу з буряком',
});

// Порівнюємо схожість
console.log('TS setup vs tsconfig:', cosineSimilarity(v1, v2)); // ~0.85 (схожі!)
console.log('TS setup vs борщ:', cosineSimilarity(v1, v3));     // ~0.12 (різні!)
```

### Моделі для embeddings

| Модель | Розмір вектора | Ціна за 1M токенів | Якість |
|--------|---------------|-------------------|--------|
| text-embedding-3-small | 1536 | $0.02 | Достатня для більшості задач |
| text-embedding-3-large | 3072 | $0.13 | Найвища якість |
| Gemini text-embedding | 768 | Безкоштовно | Хороша, безкоштовна |

---

## 10.4 Chunking: Як розбити документ

Chunking — це **критичний** крок. Поганий chunking = поганий RAG.

### Стратегія 1: Fixed Size (найпростіша)

```typescript
function chunkBySize(text: string, chunkSize = 500, overlap = 100): string[] {
  const chunks: string[] = [];
  let start = 0;

  while (start < text.length) {
    const end = Math.min(start + chunkSize, text.length);
    chunks.push(text.slice(start, end));
    start += chunkSize - overlap; // Overlap щоб не розривати контекст
  }

  return chunks;
}
```

**Мінус:** Може розрізати речення посередині.

### Стратегія 2: По реченнях/абзацах (краща)

```typescript
function chunkBySentences(text: string, maxTokens = 300): string[] {
  const sentences = text.split(/(?<=[.!?])\s+/);
  const chunks: string[] = [];
  let currentChunk = '';

  for (const sentence of sentences) {
    if ((currentChunk + ' ' + sentence).length > maxTokens * 4) { // ~4 chars per token
      if (currentChunk) chunks.push(currentChunk.trim());
      currentChunk = sentence;
    } else {
      currentChunk += ' ' + sentence;
    }
  }

  if (currentChunk) chunks.push(currentChunk.trim());
  return chunks;
}
```

### Стратегія 3: По структурі документа (найкраща)

```typescript
function chunkByMarkdownHeaders(markdown: string): Array<{ heading: string; content: string }> {
  const sections = markdown.split(/(?=^##\s)/m);

  return sections
    .filter(s => s.trim())
    .map(section => {
      const lines = section.split('\n');
      const heading = lines[0].replace(/^#+\s/, '').trim();
      const content = lines.slice(1).join('\n').trim();
      return { heading, content };
    });
}
```

### Практичне правило вибору chunking

| Тип документа | Стратегія | Розмір chunk |
|--------------|-----------|-------------|
| Markdown/Wiki | По заголовках | Один розділ |
| Код | По функціях/класах | Одна функція |
| PDF/Договори | По абзацах з overlap | 300-500 токенів |
| FAQ | Кожне питання-відповідь окремо | 1 Q&A |
| Чат-логи | По повідомленнях | Блок повідомлень |

---

## 10.5 Повний RAG Pipeline

```typescript
import { embed, embedMany, generateText, cosineSimilarity } from 'ai';
import { openai } from '@ai-sdk/openai';

// === КРОК 1: Indexing ===

interface Chunk {
  id: string;
  text: string;
  embedding: number[];
  metadata: { source: string; heading?: string };
}

const knowledgeBase: Chunk[] = []; // В production — векторна база

async function indexDocuments(documents: Array<{ name: string; content: string }>) {
  for (const doc of documents) {
    const chunks = chunkBySentences(doc.content, 300);

    const { embeddings } = await embedMany({
      model: openai.embedding('text-embedding-3-small'),
      values: chunks,
    });

    for (let i = 0; i < chunks.length; i++) {
      knowledgeBase.push({
        id: `${doc.name}-${i}`,
        text: chunks[i],
        embedding: embeddings[i],
        metadata: { source: doc.name },
      });
    }
  }
  console.log(`Indexed ${knowledgeBase.length} chunks from ${documents.length} documents`);
}

// === КРОК 2: Retrieval ===

async function findRelevantChunks(query: string, topK = 3): Promise<Chunk[]> {
  const { embedding: queryEmbedding } = await embed({
    model: openai.embedding('text-embedding-3-small'),
    value: query,
  });

  // Рахуємо схожість з усіма chunks
  const scored = knowledgeBase.map(chunk => ({
    ...chunk,
    score: cosineSimilarity(queryEmbedding, chunk.embedding),
  }));

  // Повертаємо топ-K найсхожіших
  return scored
    .sort((a, b) => b.score - a.score)
    .slice(0, topK);
}

// === КРОК 3: Generation ===

async function askWithRAG(question: string): Promise<string> {
  const relevantChunks = await findRelevantChunks(question);

  const context = relevantChunks
    .map((c, i) => `[Джерело: ${c.metadata.source}]\n${c.text}`)
    .join('\n\n---\n\n');

  const { text } = await generateText({
    model: openai('gpt-4o-mini'),
    system: `Ти — асистент що відповідає на питання ТІЛЬКИ на основі наданого контексту.
Якщо відповіді немає в контексті — скажи "Я не знайшов відповідь у базі знань."
Завжди вказуй джерело інформації.`,
    prompt: `Контекст:
${context}

Питання: ${question}`,
  });

  return text;
}

// === Використання ===

// 1. Індексуємо документи (один раз)
await indexDocuments([
  { name: 'setup-guide.md', content: '...' },
  { name: 'api-reference.md', content: '...' },
  { name: 'troubleshooting.md', content: '...' },
]);

// 2. Задаємо питання (кожен запит)
const answer = await askWithRAG('Як налаштувати автентифікацію?');
console.log(answer);
```

---

## 10.6 Векторні бази даних

Для production замість масиву в пам'яті використовують спеціалізовані бази:

| База | Тип | Коли використовувати |
|------|-----|---------------------|
| **pgvector** | Розширення PostgreSQL | Вже є PostgreSQL, до 1M документів |
| **Pinecone** | Managed SaaS | Не хочете керувати інфраструктурою |
| **Qdrant** | Self-hosted | Потрібна повна контроль, великі обсяги |
| **Chroma** | Embedded | Прототипування, малі обсяги |

### Приклад з pgvector

```typescript
// Створення таблиці
await db.query(`
  CREATE EXTENSION IF NOT EXISTS vector;
  
  CREATE TABLE documents (
    id TEXT PRIMARY KEY,
    content TEXT NOT NULL,
    metadata JSONB DEFAULT '{}',
    embedding vector(1536)  -- розмір text-embedding-3-small
  );
  
  CREATE INDEX ON documents USING ivfflat (embedding vector_cosine_ops);
`);

// Пошук
const results = await db.query(`
  SELECT content, metadata, 1 - (embedding <=> $1) as similarity
  FROM documents
  WHERE 1 - (embedding <=> $1) > 0.7  -- мінімальна схожість
  ORDER BY embedding <=> $1
  LIMIT 5
`, [queryEmbedding]);
```

---

## Перевір себе

1. Поясніть різницю між RAG і fine-tuning. Коли що використовувати?
2. Чому chunking — критичний крок? Що буде якщо chunks занадто великі або маленькі?
3. Реалізуйте RAG для FAQ-бази з 10 питань-відповідей
4. Що таке cosine similarity і навіщо мінімальний поріг?
5. Коли pgvector достатньо, а коли потрібна Pinecone?

---

**Назад:** [← Модуль 09 — Пам'ять агентів](09-memory.md) | **Далі:** [Модуль 11 — MCP →](11-mcp.md)
