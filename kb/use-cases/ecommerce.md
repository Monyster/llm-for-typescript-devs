# Модуль 33: AI для e-commerce — Описи, рекомендації, аналіз відгуків

## 🎯 Що ви отримаєте з цього модуля

Після проходження ви будете:
- Генерувати SEO-оптимізовані описи товарів масово (batch)
- Будувати AI recommendation engine на основі embeddings
- Автоматизувати аналіз відгуків: sentiment, теми, actionable insights
- Реалізовувати AI-powered пошук по каталогу

**Які задачі це дозволяє вирішувати:** Автоматизація контенту для інтернет-магазину з 10,000+ товарів. Персоналізовані рекомендації. Моніторинг відгуків для product-менеджера.

---

## 33.1 Генерація описів товарів

### Batch-генерація через OpenAI Batch API

```typescript
import { generateObject } from 'ai';
import { openai } from '@ai-sdk/openai';
import { z } from 'zod';

const ProductDescription = z.object({
  title: z.string().describe('SEO-заголовок, до 70 символів'),
  shortDescription: z.string().describe('Короткий опис для каталогу, 1-2 речення'),
  fullDescription: z.string().describe('Повний опис для сторінки товару, 3-5 абзаців'),
  bulletPoints: z.array(z.string()).max(5).describe('Ключові переваги'),
  seoKeywords: z.array(z.string()).max(10),
  metaDescription: z.string().max(160),
});

async function generateProductDescription(product: {
  name: string;
  category: string;
  specs: Record<string, string>;
  targetAudience?: string;
}) {
  const { object } = await generateObject({
    model: openai('gpt-4o-mini'),
    schema: ProductDescription,
    temperature: 0.7,
    system: `Ти — копірайтер для e-commerce магазину електроніки.
Пиши українською. Стиль: професійний але дружній.
SEO: використовуй ключові слова природно, не спамь.
Уникай кліше типу "найкращий", "унікальний", "неперевершений".`,
    prompt: `Створи опис товару:
Назва: ${product.name}
Категорія: ${product.category}
Характеристики: ${JSON.stringify(product.specs)}
${product.targetAudience ? `Цільова аудиторія: ${product.targetAudience}` : ''}`,
  });

  return object;
}

// Batch обробка 1000 товарів
async function batchGenerateDescriptions(products: any[]) {
  const CONCURRENCY = 10; // 10 паралельних запитів

  for (let i = 0; i < products.length; i += CONCURRENCY) {
    const batch = products.slice(i, i + CONCURRENCY);
    const results = await Promise.all(
      batch.map(p => generateProductDescription(p))
    );

    for (let j = 0; j < results.length; j++) {
      await db.products.update(batch[j].id, { description: results[j] });
    }

    console.log(`Оброблено ${Math.min(i + CONCURRENCY, products.length)}/${products.length}`);
  }
}
```

**Вартість:** ~$0.002/товар (GPT-4o-mini) → 10,000 товарів = ~$20.

---

## 33.2 AI Recommendation Engine

Рекомендації на основі embeddings — "клієнти які купили X, також купили Y":

```typescript
import { embed, cosineSimilarity } from 'ai';
import { openai } from '@ai-sdk/openai';

// Крок 1: Створюємо embedding для кожного товару
async function indexProducts(products: Array<{ id: string; description: string }>) {
  for (const product of products) {
    const { embedding } = await embed({
      model: openai.embedding('text-embedding-3-small'),
      value: product.description,
    });

    await db.query(
      'UPDATE products SET embedding = $1 WHERE id = $2',
      [JSON.stringify(embedding), product.id]
    );
  }
}

// Крок 2: Рекомендації "Схожі товари"
async function getSimilarProducts(productId: string, limit = 5) {
  const product = await db.query('SELECT embedding FROM products WHERE id = $1', [productId]);

  const results = await db.query(`
    SELECT id, name, price, 1 - (embedding <=> $1) as similarity
    FROM products
    WHERE id != $2
    ORDER BY embedding <=> $1
    LIMIT $3
  `, [product.embedding, productId, limit]);

  return results.rows;
}

// Крок 3: Персоналізовані рекомендації на основі історії покупок
async function getPersonalizedRecommendations(userId: string, limit = 10) {
  const purchaseHistory = await db.query(
    'SELECT p.embedding FROM orders o JOIN products p ON o.product_id = p.id WHERE o.user_id = $1',
    [userId]
  );

  // Середній вектор інтересів користувача
  const avgEmbedding = averageVectors(purchaseHistory.rows.map(r => r.embedding));

  const results = await db.query(`
    SELECT id, name, price, 1 - (embedding <=> $1) as relevance
    FROM products
    WHERE id NOT IN (SELECT product_id FROM orders WHERE user_id = $2)
    ORDER BY embedding <=> $1
    LIMIT $3
  `, [JSON.stringify(avgEmbedding), userId, limit]);

  return results.rows;
}
```

---

## 33.3 Аналіз відгуків

```typescript
const ReviewAnalysis = z.object({
  sentiment: z.enum(['positive', 'negative', 'mixed', 'neutral']),
  rating_predicted: z.number().min(1).max(5),
  topics: z.array(z.enum([
    'quality', 'price', 'delivery', 'support', 'packaging', 'design', 'functionality', 'durability',
  ])),
  pros: z.array(z.string()),
  cons: z.array(z.string()),
  actionable_insight: z.string().nullable().describe('Конкретна рекомендація для product team'),
  is_fake: z.boolean().describe('Підозра на фейковий відгук'),
});

async function analyzeReviews(reviews: Array<{ id: string; text: string; rating: number }>) {
  const results = [];

  for (const review of reviews) {
    const { object } = await generateObject({
      model: openai('gpt-4o-mini'),
      schema: ReviewAnalysis,
      temperature: 0,
      prompt: `Проаналізуй відгук (рейтинг: ${review.rating}/5):
"${review.text}"`,
    });

    results.push({ reviewId: review.id, ...object });
  }

  return results;
}

// Агрегований звіт для product-менеджера
async function generateReviewReport(productId: string) {
  const analyses = await db.getReviewAnalyses(productId);

  const { text } = await generateText({
    model: openai('gpt-4o-mini'),
    system: 'Ти — product analyst. Створи звіт на основі аналізу відгуків.',
    prompt: `Продукт: ${productId}
Всього відгуків: ${analyses.length}
Розподіл sentiment: ${JSON.stringify(countBy(analyses, 'sentiment'))}
Топ теми: ${JSON.stringify(countBy(analyses.flatMap(a => a.topics)))}
Всі insights: ${analyses.filter(a => a.actionable_insight).map(a => a.actionable_insight).join('; ')}

Створи короткий звіт з рекомендаціями.`,
  });

  return text;
}
```

---

## 33.4 AI-powered Product Search

```typescript
// Семантичний пошук: "щось для бігу під дощем" → waterproof running shoes
async function semanticProductSearch(query: string, filters?: {
  category?: string;
  minPrice?: number;
  maxPrice?: number;
}) {
  const { embedding } = await embed({
    model: openai.embedding('text-embedding-3-small'),
    value: query,
  });

  let sql = `
    SELECT id, name, price, short_description,
           1 - (embedding <=> $1) as relevance
    FROM products
    WHERE 1=1
  `;
  const params: any[] = [JSON.stringify(embedding)];

  if (filters?.category) {
    params.push(filters.category);
    sql += ` AND category = $${params.length}`;
  }
  if (filters?.maxPrice) {
    params.push(filters.maxPrice);
    sql += ` AND price <= $${params.length}`;
  }

  sql += ` ORDER BY embedding <=> $1 LIMIT 20`;

  return db.query(sql, params);
}
```

---

## Перевір себе

1. Скільки коштує згенерувати описи для 10,000 товарів?
2. Як embedding-based рекомендації відрізняються від collaborative filtering?
3. Реалізуйте аналіз відгуків для 5 прикладів зі свого магазину
4. Як виявити фейкові відгуки через AI?
5. Чому семантичний пошук краще за keyword для e-commerce?

---

**Назад:** [← Модуль 32 — AI для DevOps](32-devops.md) | **Далі:** [Модуль 34 — Rate limiting та quota management →](34-rate-limiting.md)
