# Модуль 28: Event-driven AI architectures

## 🎯 Що ви отримаєте з цього модуля

Після проходження ви будете:
- Будувати AI-системи що реагують на події (webhook, cron, queue)
- Реалізовувати trigger-based AI обробку
- Використовувати черги для batch AI processing
- Проектувати архітектуру де AI працює у фоні

**Які задачі це дозволяє вирішувати:** AI який не чекає запиту від користувача, а сам реагує: нове повідомлення → автоматична класифікація, новий PR → автоматичний review, кожен день → звіт по метриках.

---

## 28.1 Три тригери для AI

### Webhook — зовнішня подія

```typescript
// Новий тікет у Zendesk → AI класифікує та відповідає
app.post('/webhooks/zendesk', async (req, res) => {
  const ticket = req.body;

  // AI класифікація
  const { object } = await generateObject({
    model: openai('gpt-4o-mini'),
    schema: z.object({
      category: z.enum(['billing', 'technical', 'feature', 'other']),
      priority: z.enum(['low', 'medium', 'high', 'critical']),
      autoResolvable: z.boolean(),
    }),
    prompt: `Класифікуй тікет: "${ticket.subject}: ${ticket.description}"`,
  });

  // Якщо можна вирішити автоматично
  if (object.autoResolvable) {
    const { text } = await generateText({
      model: openai('gpt-4o-mini'),
      system: 'Відповідай на тікети підтримки. Будь ввічливим і корисним.',
      prompt: ticket.description,
    });
    await zendesk.reply(ticket.id, text);
    await zendesk.updatePriority(ticket.id, object.priority);
  } else {
    await zendesk.assign(ticket.id, object.category);
  }

  res.json({ status: 'processed' });
});
```

### Cron — розклад

```typescript
// Щоденний звіт о 9:00
import cron from 'node-cron';

cron.schedule('0 9 * * *', async () => {
  console.log('[Cron] Генерація щоденного звіту...');

  const metrics = await fetchDailyMetrics();

  const { text } = await generateText({
    model: openai('gpt-4o-mini'),
    system: `Ти — бізнес-аналітик. Створи короткий щоденний звіт.
Виділи: ключові зміни, аномалії, рекомендації.`,
    prompt: `Метрики за ${new Date().toLocaleDateString('uk')}:
${JSON.stringify(metrics, null, 2)}`,
  });

  await slackBot.postMessage('#daily-report', text);
});
```

### Queue — черга повідомлень

```typescript
import { Queue, Worker } from 'bullmq';
import Redis from 'ioredis';

const redis = new Redis(process.env.REDIS_URL);
const aiQueue = new Queue('ai-processing', { connection: redis });

// Producer: додаємо задачі в чергу
async function queueForProcessing(items: string[]) {
  for (const item of items) {
    await aiQueue.add('classify', { text: item }, {
      attempts: 3,
      backoff: { type: 'exponential', delay: 5000 },
    });
  }
}

// Worker: обробляємо задачі з черги
const worker = new Worker('ai-processing', async (job) => {
  const { text } = job.data;

  const { object } = await generateObject({
    model: openai('gpt-4o-mini'),
    schema: z.object({ category: z.string(), sentiment: z.string() }),
    prompt: `Класифікуй: "${text}"`,
  });

  await db.saveResult(job.id, object);
  return object;
}, {
  connection: redis,
  concurrency: 5,  // 5 паралельних AI-запитів
  limiter: {
    max: 50,
    duration: 60_000,  // Максимум 50 запитів/хвилину (rate limit)
  },
});
```

---

## 28.2 Архітектура Event-Driven AI System

```
┌─────────────┐    ┌─────────┐    ┌──────────────┐    ┌──────────┐
│  Webhooks   │──▶│  Queue   │──▶│  AI Workers  │──▶│  Results  │
│  Cron Jobs  │   │ (BullMQ/ │   │ (generateText │   │  (DB,     │
│  API calls  │   │  SQS)    │   │  generateObj) │   │  Slack,   │
└─────────────┘    └─────────┘    └──────────────┘    │  Email)   │
                                                       └──────────┘
```

### Переваги

- **Масштабування:** додайте більше workers для більшого навантаження
- **Надійність:** якщо worker впав — задача повернеться в чергу
- **Rate limiting:** контролюйте швидкість звернень до AI API
- **Пріоритети:** критичні задачі обробляються першими

---

## 28.3 Приклад: GitHub PR Auto-Review

```typescript
// Webhook від GitHub → AI review
app.post('/webhooks/github', async (req, res) => {
  if (req.body.action !== 'opened' || !req.body.pull_request) {
    return res.status(200).send('Skipped');
  }

  const pr = req.body.pull_request;

  // Додаємо в чергу (не блокуємо webhook)
  await aiQueue.add('pr-review', {
    prNumber: pr.number,
    repo: req.body.repository.full_name,
    diff: pr.diff_url,
  }, { priority: 1 });

  res.status(200).send('Queued');
});

// Worker обробляє
const reviewWorker = new Worker('ai-processing', async (job) => {
  if (job.name !== 'pr-review') return;

  const diff = await fetchDiff(job.data.diff);

  const { text } = await generateText({
    model: openai('gpt-4o-mini'),
    system: `Ти — senior code reviewer. Переглянь PR diff.
Знайди: баги, проблеми безпеки, порушення best practices.
Формат: короткий підсумок + список проблем з номерами рядків.`,
    prompt: diff,
  });

  await github.createReview(job.data.repo, job.data.prNumber, text);
});
```

---

## Перевір себе

1. Назвіть 3 тригери для event-driven AI і приклад кожного
2. Чому webhook-handler не повинен запускати AI напряму (а через чергу)?
3. Реалізуйте cron-задачу яка щогодини аналізує нові відгуки
4. Як rate limiting черги допомагає не перевищити ліміти AI API?
5. Спроектуйте event-driven систему для автоматичного code review

---

**Назад:** [← Модуль 27 — Hybrid Search](27-hybrid-search.md) | **Далі:** [Модуль 29 — Edge AI та local models →](29-edge-local.md)
