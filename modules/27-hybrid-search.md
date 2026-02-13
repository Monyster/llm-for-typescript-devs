# Модуль 27: Hybrid Search — BM25 + Vector + Re-ranking

## 🎯 Що ви отримаєте з цього модуля

Після проходження ви будете:
- Розуміти різницю між keyword (BM25) та vector search
- Комбінувати обидва підходи для кращих результатів
- Використовувати Reciprocal Rank Fusion (RRF) для злиття результатів
- Інтегрувати re-ranking (Cohere, cross-encoder) у RAG pipeline

**Які задачі це дозволяє вирішувати:** Покращити якість пошуку у RAG на 20-40%. Vector search пропускає exact matches ("помилка ERR-4521"), keyword search пропускає семантику ("як полагодити авторизацію" ≠ "login error"). Hybrid знаходить обоє.

---

## 27.1 Проблема: кожен пошук має слабкі місця

### Vector search (semantic)

```
Запит: "проблема з оплатою"
✅ Знаходить: "не вдалось провести транзакцію кредитною карткою"
✅ Знаходить: "платіж було відхилено банком"
❌ Пропускає: "Error code: PAYMENT_ERR_429" (нема семантичного зв'язку)
```

### Keyword search (BM25)

```
Запит: "проблема з оплатою"
✅ Знаходить: "Якщо у вас проблема з оплатою, зверніться..."
✅ Знаходить: "Error code: PAYMENT_ERR_429 — проблема оплати"
❌ Пропускає: "транзакцію було відхилено" (нема точних слів)
```

### Hybrid search = обидва разом

```
Запит: "проблема з оплатою"
✅ "не вдалось провести транзакцію кредитною карткою" (vector)
✅ "Error code: PAYMENT_ERR_429 — проблема оплати" (keyword)
✅ "платіж було відхилено банком" (vector)
```

---

## 27.2 BM25 — Keyword Search

BM25 (Best Match 25) — класичний алгоритм текстового пошуку. Шукає документи з точними або схожими словами:

```typescript
// Простий BM25 з PostgreSQL full-text search
async function keywordSearch(query: string, limit = 10) {
  const results = await db.query(`
    SELECT id, content, metadata,
           ts_rank(search_vector, plainto_tsquery('ukrainian', $1)) as rank
    FROM documents
    WHERE search_vector @@ plainto_tsquery('ukrainian', $1)
    ORDER BY rank DESC
    LIMIT $2
  `, [query, limit]);

  return results.rows;
}

// Створення індексу
await db.query(`
  ALTER TABLE documents ADD COLUMN search_vector tsvector;
  UPDATE documents SET search_vector = to_tsvector('ukrainian', content);
  CREATE INDEX idx_search ON documents USING gin(search_vector);
`);
```

---

## 27.3 Reciprocal Rank Fusion (RRF) — злиття результатів

RRF комбінує ранжування з різних пошукових систем:

```typescript
interface SearchResult {
  id: string;
  text: string;
  score: number;
}

function reciprocalRankFusion(
  resultSets: SearchResult[][],
  k = 60 // Константа згладжування
): SearchResult[] {
  const scores = new Map<string, { score: number; text: string }>();

  for (const results of resultSets) {
    results.forEach((result, rank) => {
      const rrfScore = 1 / (k + rank + 1);
      const existing = scores.get(result.id);
      scores.set(result.id, {
        score: (existing?.score ?? 0) + rrfScore,
        text: result.text,
      });
    });
  }

  return Array.from(scores.entries())
    .map(([id, { score, text }]) => ({ id, text, score }))
    .sort((a, b) => b.score - a.score);
}

// Використання
async function hybridSearch(query: string, topK = 5): Promise<SearchResult[]> {
  // Паралельно запускаємо обидва типи пошуку
  const [vectorResults, keywordResults] = await Promise.all([
    vectorSearch(query, { limit: 20 }),
    keywordSearch(query, 20),
  ]);

  // Злиття через RRF
  const fused = reciprocalRankFusion([vectorResults, keywordResults]);

  return fused.slice(0, topK);
}
```

---

## 27.4 Re-ranking як фінальний шар

Після hybrid search — re-ranking для максимальної точності:

```
Query → [BM25: 20 results] ──┐
                              ├── RRF → 15 results → Re-ranker → Top 5
Query → [Vector: 20 results] ┘
```

```typescript
async function fullHybridPipeline(query: string): Promise<SearchResult[]> {
  // Крок 1: Hybrid search (BM25 + Vector)
  const candidates = await hybridSearch(query, 15);

  // Крок 2: Re-ranking через Cohere
  const response = await fetch('https://api.cohere.com/v2/rerank', {
    method: 'POST',
    headers: {
      'Authorization': `Bearer ${process.env.COHERE_API_KEY}`,
      'Content-Type': 'application/json',
    },
    body: JSON.stringify({
      model: 'rerank-v3.5',
      query,
      documents: candidates.map(c => c.text),
      top_n: 5,
    }),
  });

  const reranked = await response.json();

  return reranked.results.map((r: any) => ({
    ...candidates[r.index],
    score: r.relevance_score,
  }));
}
```

---

## 27.5 pgvector: Hybrid Search в одній базі

PostgreSQL з pgvector підтримує і vector, і full-text search:

```sql
-- Одна таблиця, обидва типи пошуку
CREATE TABLE documents (
  id TEXT PRIMARY KEY,
  content TEXT NOT NULL,
  metadata JSONB DEFAULT '{}',
  embedding vector(1536),                              -- Vector search
  search_vector tsvector GENERATED ALWAYS AS            -- Keyword search
    (to_tsvector('ukrainian', content)) STORED
);

CREATE INDEX idx_embedding ON documents USING ivfflat (embedding vector_cosine_ops);
CREATE INDEX idx_fulltext ON documents USING gin(search_vector);

-- Hybrid query: обидва результати в одному запиті
WITH vector_results AS (
  SELECT id, content, 1 - (embedding <=> $1::vector) as vector_score
  FROM documents
  ORDER BY embedding <=> $1::vector
  LIMIT 20
),
keyword_results AS (
  SELECT id, content, ts_rank(search_vector, plainto_tsquery('ukrainian', $2)) as keyword_score
  FROM documents
  WHERE search_vector @@ plainto_tsquery('ukrainian', $2)
  ORDER BY keyword_score DESC
  LIMIT 20
)
SELECT COALESCE(v.id, k.id) as id,
       COALESCE(v.content, k.content) as content,
       COALESCE(v.vector_score, 0) * 0.7 + COALESCE(k.keyword_score, 0) * 0.3 as hybrid_score
FROM vector_results v
FULL OUTER JOIN keyword_results k ON v.id = k.id
ORDER BY hybrid_score DESC
LIMIT 10;
```

---

## Перевір себе

1. Чому vector search пропускає exact matches (error codes)?
2. Що таке Reciprocal Rank Fusion і як воно працює?
3. Реалізуйте hybrid search з BM25 + vector для вашої бази документів
4. Коли re-ranking дає значне покращення, а коли зайвий?
5. Чому pgvector зручний для hybrid search?

---

**Назад:** [← Модуль 26 — Long-running agents](26-long-running-agents.md) | **Далі:** [Модуль 28 — Event-driven AI →](28-event-driven.md)
