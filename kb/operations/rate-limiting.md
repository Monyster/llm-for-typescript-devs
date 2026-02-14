# Модуль 34: Rate limiting та quota management

## 🎯 Що ви отримаєте з цього модуля

Після проходження ви будете:
- Реалізовувати per-user rate limiting для AI-запитів
- Будувати token budget system (ліміт витрат на користувача)
- Обробляти 429 помилки від провайдерів з retry та backoff
- Реалізовувати graceful degradation коли ліміти вичерпані

**Які задачі це дозволяє вирішувати:** Захистити бюджет від одного користувача який витратить весь API-ліміт. Справедливо розподілити ресурси між клієнтами. Не впасти коли OpenAI повертає 429.

---

## 34.1 Per-User Rate Limiting

```typescript
import Redis from 'ioredis';

const redis = new Redis(process.env.REDIS_URL);

interface RateLimitConfig {
  maxRequests: number;  // Максимум запитів
  windowMs: number;     // За який період
}

const PLANS: Record<string, RateLimitConfig> = {
  free:       { maxRequests: 20,  windowMs: 60 * 60 * 1000 },   // 20/год
  pro:        { maxRequests: 200, windowMs: 60 * 60 * 1000 },   // 200/год
  enterprise: { maxRequests: 2000, windowMs: 60 * 60 * 1000 },  // 2000/год
};

async function checkRateLimit(userId: string, plan: string): Promise<{
  allowed: boolean;
  remaining: number;
  resetAt: Date;
}> {
  const config = PLANS[plan] ?? PLANS.free;
  const key = `ratelimit:${userId}:${Math.floor(Date.now() / config.windowMs)}`;

  const current = await redis.incr(key);
  if (current === 1) {
    await redis.pexpire(key, config.windowMs);
  }

  const remaining = Math.max(0, config.maxRequests - current);
  const ttl = await redis.pttl(key);

  return {
    allowed: current <= config.maxRequests,
    remaining,
    resetAt: new Date(Date.now() + ttl),
  };
}

// Middleware
async function rateLimitMiddleware(req: Request, res: Response, next: NextFunction) {
  const { allowed, remaining, resetAt } = await checkRateLimit(req.userId, req.userPlan);

  res.setHeader('X-RateLimit-Remaining', remaining);
  res.setHeader('X-RateLimit-Reset', resetAt.toISOString());

  if (!allowed) {
    return res.status(429).json({
      error: 'Rate limit exceeded',
      retryAfter: Math.ceil((resetAt.getTime() - Date.now()) / 1000),
      upgradeUrl: '/pricing',
    });
  }

  next();
}
```

---

## 34.2 Token Budget System

Ліміт не на кількість запитів, а на кількість витрачених токенів (=грошей):

```typescript
interface TokenBudget {
  dailyLimit: number;     // Максимум токенів на день
  monthlyLimit: number;   // Максимум на місяць
}

const BUDGETS: Record<string, TokenBudget> = {
  free:       { dailyLimit: 50_000,   monthlyLimit: 500_000 },
  pro:        { dailyLimit: 500_000,  monthlyLimit: 10_000_000 },
  enterprise: { dailyLimit: 5_000_000, monthlyLimit: 100_000_000 },
};

async function checkTokenBudget(userId: string, plan: string, estimatedTokens: number): Promise<{
  allowed: boolean;
  dailyUsed: number;
  monthlyUsed: number;
  dailyRemaining: number;
}> {
  const budget = BUDGETS[plan] ?? BUDGETS.free;
  const today = new Date().toISOString().slice(0, 10);
  const month = today.slice(0, 7);

  const [dailyUsed, monthlyUsed] = await Promise.all([
    redis.get(`tokens:${userId}:${today}`).then(v => parseInt(v ?? '0')),
    redis.get(`tokens:${userId}:${month}`).then(v => parseInt(v ?? '0')),
  ]);

  const allowed =
    dailyUsed + estimatedTokens <= budget.dailyLimit &&
    monthlyUsed + estimatedTokens <= budget.monthlyLimit;

  return {
    allowed,
    dailyUsed,
    monthlyUsed,
    dailyRemaining: budget.dailyLimit - dailyUsed,
  };
}

async function recordTokenUsage(userId: string, tokensUsed: number) {
  const today = new Date().toISOString().slice(0, 10);
  const month = today.slice(0, 7);

  await Promise.all([
    redis.incrby(`tokens:${userId}:${today}`, tokensUsed),
    redis.incrby(`tokens:${userId}:${month}`, tokensUsed),
  ]);

  // TTL: daily — 48 годин, monthly — 35 днів
  await redis.expire(`tokens:${userId}:${today}`, 48 * 3600);
  await redis.expire(`tokens:${userId}:${month}`, 35 * 24 * 3600);
}

// Використання з AI SDK
const budgetCheck = await checkTokenBudget(userId, userPlan, 2000);
if (!budgetCheck.allowed) {
  return { error: 'Token budget exceeded', dailyRemaining: budgetCheck.dailyRemaining };
}

const { text, usage } = await generateText({ model: openai('gpt-4o-mini'), prompt: message });
await recordTokenUsage(userId, usage.totalTokens);
```

---

## 34.3 Handling Provider Rate Limits (429)

```typescript
async function generateWithRetry(
  params: Parameters<typeof generateText>[0],
  maxRetries = 3
): Promise<ReturnType<typeof generateText>> {
  for (let attempt = 0; attempt <= maxRetries; attempt++) {
    try {
      return await generateText(params);
    } catch (error: any) {
      if (error.status === 429 && attempt < maxRetries) {
        // Exponential backoff: 1s, 2s, 4s
        const delay = Math.pow(2, attempt) * 1000;
        const retryAfter = error.headers?.['retry-after'];
        const waitMs = retryAfter ? parseInt(retryAfter) * 1000 : delay;

        console.warn(`Rate limited. Retry ${attempt + 1}/${maxRetries} in ${waitMs}ms`);
        await new Promise(r => setTimeout(r, waitMs));
        continue;
      }
      throw error; // Не 429 або вичерпані retry
    }
  }
  throw new Error('Max retries exceeded');
}
```

---

## 34.4 Graceful Degradation

Коли ліміти вичерпані — не падаємо, а зменшуємо якість:

```typescript
async function generateWithDegradation(userId: string, message: string) {
  const budget = await checkTokenBudget(userId, userPlan, 2000);

  if (budget.dailyRemaining > 10_000) {
    // Повна якість
    return generateText({ model: openai('gpt-4o-mini'), prompt: message });
  }

  if (budget.dailyRemaining > 1_000) {
    // Зменшуємо якість: коротша відповідь
    return generateText({
      model: openai('gpt-4o-mini'),
      maxTokens: 100,
      system: 'Відповідай максимально коротко, 1-2 речення.',
      prompt: message,
    });
  }

  // Ліміт майже вичерпано: fallback без AI
  return {
    text: 'Ваш денний ліміт AI-запитів вичерпано. Оновіть план або спробуйте завтра.',
    usage: { totalTokens: 0 },
  };
}
```

---

## Перевір себе

1. Чим rate limiting за запитами відрізняється від rate limiting за токенами?
2. Реалізуйте per-user token budget з Redis
3. Як правильно обробляти 429 від OpenAI?
4. Що таке graceful degradation і як його реалізувати?
5. Як пояснити клієнту різницю між Free/Pro/Enterprise лімітами?

---

**Назад:** [← Модуль 33 — AI для e-commerce](33-ecommerce.md) | **Далі:** [Модуль 35 — Data pipelines для RAG →](35-rag-pipelines.md)
