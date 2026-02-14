# Модуль 09: Пам'ять агентів — Короткострокова та довгострокова

## 🎯 Що ви отримаєте з цього модуля

Після проходження ви будете:
- Розуміти 3 типи пам'яті агента: in-context, extracted, vectorized
- Реалізовувати збереження та відновлення контексту між сесіями
- Використовувати sliding window та summarization для довгих розмов
- Будувати hybrid memory з SQL + векторним пошуком

**Які задачі це дозволяє вирішувати:** Зробити агента який "пам'ятає" вас. Персоналізація відповідей, збереження контексту проекту між сесіями, накопичення знань з розмов — все що перетворює одноразовий чатбот на постійного асистента.

---

## 💰 Бізнес-цінність

**Проблема клієнта:** "Клієнт пише в чат вже третій раз, а AI знову питає 'як вас звати?'"
**Рішення з цього модуля:** Пам'ять між сесіями — AI знає історію клієнта і контекст.
**Як продати:** Персоналізація = більше конверсій. Клієнти що відчувають що їх "знають" — лояльніші на 40%. Додає $3–5K до проекту.

## 9.1 Проблема: LLM нічого не пам'ятає

За замовчуванням LLM — **stateless**. Кожен запит — це окремий виклик без будь-якого зв'язку з попередніми:

```typescript
// Сесія 1
await generateText({ prompt: 'Мене звати Олег, я працюю над проектом ShopAI' });
// → "Привіт, Олег! Розкажіть більше про ShopAI..."

// Сесія 2 (нова розмова)
await generateText({ prompt: 'Яке моє ім\'я?' });
// → "Я не знаю вашого імені, ви не повідомляли його."
// LLM НЕ пам'ятає попередню сесію!
```

Пам'ять — це **ваша відповідальність**. Модель бачить тільки те, що ви передаєте в `messages`.

---

## 9.2 Три типи пам'яті

### In-Context Memory (короткострокова)

Найпростіший підхід — тримати всю історію в масиві `messages`:

```typescript
import { generateText, CoreMessage } from 'ai';
import { openai } from '@ai-sdk/openai';

// Зберігаємо історію в пам'яті процесу
const conversationHistory: CoreMessage[] = [
  { role: 'system', content: 'Ти — асистент розробника. Запам\'ятовуй контекст розмови.' },
];

async function chat(userMessage: string): Promise<string> {
  conversationHistory.push({ role: 'user', content: userMessage });

  const { text, usage } = await generateText({
    model: openai('gpt-4o-mini'),
    messages: conversationHistory,
  });

  conversationHistory.push({ role: 'assistant', content: text });

  console.log(`[${usage.totalTokens} токенів, ${conversationHistory.length} повідомлень]`);
  return text;
}

// Працює в межах однієї сесії:
await chat('Мене звати Олег');          // "Привіт, Олег!"
await chat('Яке моє ім\'я?');           // "Вас звати Олег."
await chat('Я працюю над ShopAI');      // "Цікаво! Розкажіть про ShopAI..."
// Але з кожним повідомленням токенів більше і вартість росте!
```

**Проблема:** Вікно контексту скінчиться + вартість зростає лінійно. Після 100 повідомлень ви відправляєте ВСЮ історію з кожним запитом.

### Sliding Window — обмежуємо історію

```typescript
const MAX_MESSAGES = 20; // Тримаємо останні 20 повідомлень

async function chatWithWindow(userMessage: string): Promise<string> {
  conversationHistory.push({ role: 'user', content: userMessage });

  // Завжди тримаємо system prompt + останні N повідомлень
  const systemMessage = conversationHistory[0];
  const recentMessages = conversationHistory.slice(-MAX_MESSAGES);
  const messages = [systemMessage, ...recentMessages];

  const { text } = await generateText({
    model: openai('gpt-4o-mini'),
    messages,
  });

  conversationHistory.push({ role: 'assistant', content: text });
  return text;
}
```

**Мінус:** Агент "забуває" що було на початку розмови.

### Summarization — стиснення старої історії

```typescript
async function summarizeOldMessages(messages: CoreMessage[]): Promise<string> {
  const { text } = await generateText({
    model: openai('gpt-4o-mini'),
    temperature: 0,
    system: 'Стисни діалог у короткий підсумок. Збережи: імена, факти, рішення, задачі.',
    prompt: messages.map(m => `${m.role}: ${m.content}`).join('\n'),
  });
  return text;
}

async function chatWithSummary(userMessage: string): Promise<string> {
  conversationHistory.push({ role: 'user', content: userMessage });

  // Якщо історія стала занадто довгою — стискаємо
  if (conversationHistory.length > 30) {
    const oldMessages = conversationHistory.slice(1, -10); // Без system та останніх 10
    const summary = await summarizeOldMessages(oldMessages);

    // Замінюємо старі повідомлення одним summary
    conversationHistory.splice(1, conversationHistory.length - 11, {
      role: 'assistant',
      content: `[Підсумок попередньої розмови: ${summary}]`,
    });
  }

  const { text } = await generateText({
    model: openai('gpt-4o-mini'),
    messages: conversationHistory,
  });

  conversationHistory.push({ role: 'assistant', content: text });
  return text;
}
```

---

## 9.3 Extracted Memory — факти з розмов

Витягуємо важливі факти з кожної розмови і зберігаємо окремо:

```typescript
import { generateObject } from 'ai';
import { z } from 'zod';

const MemoryFact = z.object({
  facts: z.array(z.object({
    category: z.enum(['personal', 'project', 'preference', 'task', 'decision']),
    key: z.string().describe('Короткий ключ факту'),
    value: z.string().describe('Сам факт'),
    confidence: z.number().min(0).max(1),
  })),
});

async function extractMemories(messages: CoreMessage[]): Promise<z.infer<typeof MemoryFact>> {
  const { object } = await generateObject({
    model: openai('gpt-4o-mini'),
    schema: MemoryFact,
    temperature: 0,
    system: `Витягни важливі факти з діалогу.
Збережи тільки те, що може бути корисним у майбутніх розмовах.
Не зберігай загальні знання — тільки специфічне про цього користувача.`,
    prompt: messages.map(m => `${m.role}: ${m.content}`).join('\n'),
  });
  return object;
}

// Приклад результату після розмови:
// {
//   facts: [
//     { category: 'personal', key: 'name', value: 'Олег', confidence: 1.0 },
//     { category: 'project', key: 'current_project', value: 'ShopAI — AI-powered e-commerce', confidence: 0.9 },
//     { category: 'preference', key: 'language', value: 'TypeScript', confidence: 1.0 },
//     { category: 'preference', key: 'style', value: 'Короткі відповіді з прикладами коду', confidence: 0.7 },
//   ]
// }
```

### Збереження в базу

```typescript
// Проста SQL-схема для memory
const SCHEMA = `
CREATE TABLE IF NOT EXISTS memories (
  id TEXT PRIMARY KEY,
  user_id TEXT NOT NULL,
  category TEXT NOT NULL,
  key TEXT NOT NULL,
  value TEXT NOT NULL,
  confidence REAL DEFAULT 1.0,
  created_at DATETIME DEFAULT CURRENT_TIMESTAMP,
  updated_at DATETIME DEFAULT CURRENT_TIMESTAMP
);

CREATE INDEX idx_memories_user ON memories(user_id);
CREATE INDEX idx_memories_category ON memories(user_id, category);
`;

class MemoryStore {
  async save(userId: string, facts: MemoryFact['facts']) {
    for (const fact of facts) {
      // Upsert — оновлюємо якщо факт вже існує
      await db.query(`
        INSERT INTO memories (id, user_id, category, key, value, confidence)
        VALUES ($1, $2, $3, $4, $5, $6)
        ON CONFLICT (id) DO UPDATE SET
          value = $5, confidence = $6, updated_at = CURRENT_TIMESTAMP
      `, [
        `${userId}:${fact.category}:${fact.key}`,
        userId, fact.category, fact.key, fact.value, fact.confidence
      ]);
    }
  }

  async getForUser(userId: string): Promise<string> {
    const rows = await db.query(
      'SELECT category, key, value FROM memories WHERE user_id = $1 ORDER BY updated_at DESC LIMIT 50',
      [userId]
    );
    return rows.map(r => `[${r.category}] ${r.key}: ${r.value}`).join('\n');
  }
}
```

### Використання в системному промпті

```typescript
const memoryStore = new MemoryStore();

async function chatWithMemory(userId: string, message: string) {
  const memories = await memoryStore.getForUser(userId);

  const { text } = await generateText({
    model: openai('gpt-4o-mini'),
    system: `Ти — персональний AI-асистент.

Ось що ти знаєш про цього користувача:
${memories || 'Поки що нічого — це нова розмова.'}

Використовуй цю інформацію для персоналізації відповідей.`,
    prompt: message,
  });

  return text;
}

// Тепер навіть у НОВІЙ сесії:
await chatWithMemory('user-123', 'Яке моє ім\'я?');
// → "Вас звати Олег! Як просувається ShopAI?"
```

---

## 9.4 Vectorized Memory — семантичний пошук по спогадах

Для великої кількості фактів SQL-пошук за ключами недостатній. Потрібен **семантичний пошук** — знаходити релевантні спогади за змістом:

```typescript
import { embed } from 'ai';
import { openai } from '@ai-sdk/openai';

class VectorMemory {
  // Зберігаємо спогад з вектором
  async store(userId: string, text: string, metadata: Record<string, string>) {
    const { embedding } = await embed({
      model: openai.embedding('text-embedding-3-small'),
      value: text,
    });

    await db.query(`
      INSERT INTO memory_vectors (user_id, text, metadata, embedding)
      VALUES ($1, $2, $3, $4)
    `, [userId, text, JSON.stringify(metadata), JSON.stringify(embedding)]);
  }

  // Шукаємо релевантні спогади
  async recall(userId: string, query: string, limit = 5): Promise<string[]> {
    const { embedding } = await embed({
      model: openai.embedding('text-embedding-3-small'),
      value: query,
    });

    // pgvector: пошук найближчих векторів
    const rows = await db.query(`
      SELECT text, 1 - (embedding <=> $1) as similarity
      FROM memory_vectors
      WHERE user_id = $2
      ORDER BY embedding <=> $1
      LIMIT $3
    `, [JSON.stringify(embedding), userId, limit]);

    return rows.map(r => r.text);
  }
}
```

Детальніше про вектори та embeddings — у Модулі 10 (RAG).

---

## 9.5 Hybrid Memory: Повна архітектура

Найкращий підхід для production — **комбінація всіх типів**:

```
┌────────────────────────────────────────────────┐
│              Запит користувача                  │
├────────────────────────────────────────────────┤
│  1. Structured lookup (SQL)                    │
│     → Ім'я, проект, preferences                │
│                                                │
│  2. Vector search (семантичний)                │
│     → Релевантні попередні розмови             │
│                                                │
│  3. In-context (messages[])                    │
│     → Поточна розмова (sliding window)         │
├────────────────────────────────────────────────┤
│         System Prompt + знайдений контекст      │
│               ↓                                │
│              LLM                               │
│               ↓                                │
│           Відповідь                             │
├────────────────────────────────────────────────┤
│  4. Після відповіді: extract & save            │
│     → Нові факти → SQL + Vector               │
└────────────────────────────────────────────────┘
```

---

## Перевір себе

1. Чому LLM нічого не пам'ятає між сесіями? Як це виправити?
2. Чим sliding window відрізняється від summarization? Плюси і мінуси кожного
3. Що таке extracted memory і коли його використовувати?
4. Реалізуйте просту memory store яка зберігає факти в JSON-файл
5. Чому hybrid memory краще ніж один підхід?

---

**Назад:** [← Модуль 08 — AI Агенти](08-agents.md) | **Далі:** [Модуль 10 — RAG →](10-rag.md)
