# 🚀 Основи LLM та AI-агентів для розробників — Part 4

---

# Модуль 8: Production та оптимізація

> 🎯 **Мета модуля:** Навчитись розгортати LLM-застосунки в production з оптимальною вартістю та надійністю.

## 8.1 Архітектура Production LLM системи

```
┌─────────────────────────────────────────────────────────────────────────┐
│                     PRODUCTION ARCHITECTURE                              │
└─────────────────────────────────────────────────────────────────────────┘

     Users
       │
       ▼
┌─────────────┐     ┌─────────────┐     ┌─────────────┐
│   CDN /     │────▶│   Load      │────▶│   API       │
│   WAF       │     │   Balancer  │     │   Gateway   │
└─────────────┘     └─────────────┘     └─────────────┘
                                              │
                    ┌─────────────────────────┼─────────────────────────┐
                    │                         │                         │
                    ▼                         ▼                         ▼
            ┌─────────────┐           ┌─────────────┐           ┌─────────────┐
            │   Cache     │           │   App       │           │   Queue     │
            │   (Redis)   │           │   Servers   │           │   (SQS)     │
            └─────────────┘           └─────────────┘           └─────────────┘
                                            │
                    ┌───────────────────────┼───────────────────────┐
                    │                       │                       │
                    ▼                       ▼                       ▼
            ┌─────────────┐         ┌─────────────┐         ┌─────────────┐
            │   OpenAI    │         │  Anthropic  │         │   Vector    │
            │   API       │         │   API       │         │   DB        │
            └─────────────┘         └─────────────┘         └─────────────┘
```

## 8.2 Caching стратегії

### Exact Match Cache

```typescript
// cache/exact-match.ts
import Redis from 'ioredis';
import crypto from 'crypto';

class LLMCache {
  private redis: Redis;
  private ttlSeconds: number;
  
  constructor(redisUrl: string, ttlSeconds: number = 3600) {
    this.redis = new Redis(redisUrl);
    this.ttlSeconds = ttlSeconds;
  }
  
  private generateKey(prompt: string, model: string, params: object): string {
    const data = JSON.stringify({ prompt, model, params });
    return `llm:${crypto.createHash('sha256').update(data).digest('hex')}`;
  }
  
  async get(prompt: string, model: string, params: object): Promise<string | null> {
    const key = this.generateKey(prompt, model, params);
    const cached = await this.redis.get(key);
    
    if (cached) {
      console.log('📦 Cache HIT');
      return cached;
    }
    
    console.log('📭 Cache MISS');
    return null;
  }
  
  async set(prompt: string, model: string, params: object, response: string): Promise<void> {
    const key = this.generateKey(prompt, model, params);
    await this.redis.setex(key, this.ttlSeconds, response);
    console.log('💾 Cached response');
  }
}

// Використання
const cache = new LLMCache(process.env.REDIS_URL!);

async function generateWithCache(prompt: string) {
  const model = 'gpt-4o-mini';
  const params = { temperature: 0 }; // Детерміновані параметри для кешу
  
  // Спочатку перевіряємо кеш
  const cached = await cache.get(prompt, model, params);
  if (cached) return cached;
  
  // Якщо немає — генеруємо
  const { text } = await generateText({
    model: openai(model),
    temperature: 0,
    prompt
  });
  
  // Зберігаємо в кеш
  await cache.set(prompt, model, params, text);
  
  return text;
}
```

### Semantic Cache (для схожих запитів)

```typescript
// cache/semantic-cache.ts
import { embed } from 'ai';
import { openai } from '@ai-sdk/openai';

class SemanticCache {
  private entries: Array<{
    embedding: number[];
    prompt: string;
    response: string;
    createdAt: Date;
  }> = [];
  
  private similarityThreshold = 0.95; // Висока схожість для кешу
  
  async get(prompt: string): Promise<string | null> {
    const { embedding } = await embed({
      model: openai.embedding('text-embedding-3-small'),
      value: prompt
    });
    
    for (const entry of this.entries) {
      const similarity = this.cosine(embedding, entry.embedding);
      
      if (similarity >= this.similarityThreshold) {
        console.log(`📦 Semantic cache HIT (${(similarity * 100).toFixed(1)}% similar)`);
        return entry.response;
      }
    }
    
    return null;
  }
  
  async set(prompt: string, response: string): Promise<void> {
    const { embedding } = await embed({
      model: openai.embedding('text-embedding-3-small'),
      value: prompt
    });
    
    this.entries.push({
      embedding,
      prompt,
      response,
      createdAt: new Date()
    });
  }
  
  private cosine(a: number[], b: number[]): number {
    let dot = 0, normA = 0, normB = 0;
    for (let i = 0; i < a.length; i++) {
      dot += a[i] * b[i];
      normA += a[i] * a[i];
      normB += b[i] * b[i];
    }
    return dot / (Math.sqrt(normA) * Math.sqrt(normB));
  }
}
```

## 8.3 Калькулятор вартості

```typescript
// cost/calculator.ts

interface ModelPricing {
  inputPer1M: number;  // $ per 1M input tokens
  outputPer1M: number; // $ per 1M output tokens
}

const PRICING: Record<string, ModelPricing> = {
  'gpt-4o': { inputPer1M: 2.50, outputPer1M: 10.00 },
  'gpt-4o-mini': { inputPer1M: 0.15, outputPer1M: 0.60 },
  'claude-3-5-sonnet': { inputPer1M: 3.00, outputPer1M: 15.00 },
  'claude-3-5-haiku': { inputPer1M: 0.80, outputPer1M: 4.00 },
  'gemini-1.5-pro': { inputPer1M: 1.25, outputPer1M: 5.00 },
  'gemini-1.5-flash': { inputPer1M: 0.075, outputPer1M: 0.30 },
};

interface UsageEstimate {
  requestsPerDay: number;
  avgInputTokens: number;
  avgOutputTokens: number;
}

function calculateMonthlyCost(
  model: string,
  usage: UsageEstimate
): {
  daily: number;
  monthly: number;
  breakdown: { input: number; output: number };
} {
  const pricing = PRICING[model];
  if (!pricing) throw new Error(`Unknown model: ${model}`);
  
  const dailyInputTokens = usage.requestsPerDay * usage.avgInputTokens;
  const dailyOutputTokens = usage.requestsPerDay * usage.avgOutputTokens;
  
  const dailyInputCost = (dailyInputTokens / 1_000_000) * pricing.inputPer1M;
  const dailyOutputCost = (dailyOutputTokens / 1_000_000) * pricing.outputPer1M;
  
  const dailyTotal = dailyInputCost + dailyOutputCost;
  const monthlyTotal = dailyTotal * 30;
  
  return {
    daily: Math.round(dailyTotal * 100) / 100,
    monthly: Math.round(monthlyTotal * 100) / 100,
    breakdown: {
      input: Math.round(dailyInputCost * 30 * 100) / 100,
      output: Math.round(dailyOutputCost * 30 * 100) / 100
    }
  };
}

// Порівняння моделей
function compareModels(usage: UsageEstimate): void {
  console.log('\n💰 Cost Comparison (Monthly)\n');
  console.log('Model                | Monthly Cost | Daily Cost');
  console.log('---------------------|--------------|------------');
  
  const results = Object.entries(PRICING)
    .map(([model]) => ({
      model,
      cost: calculateMonthlyCost(model, usage)
    }))
    .sort((a, b) => a.cost.monthly - b.cost.monthly);
  
  results.forEach(({ model, cost }) => {
    console.log(
      `${model.padEnd(20)} | $${cost.monthly.toFixed(2).padStart(10)} | $${cost.daily.toFixed(2).padStart(9)}`
    );
  });
}

// Приклад
const myUsage: UsageEstimate = {
  requestsPerDay: 1000,
  avgInputTokens: 500,
  avgOutputTokens: 200
};

compareModels(myUsage);
```

**📊 Вивід:**
```
💰 Cost Comparison (Monthly)

Model                | Monthly Cost | Daily Cost
---------------------|--------------|------------
gemini-1.5-flash     |       $2.93 |      $0.10
gpt-4o-mini          |       $5.85 |      $0.20
claude-3-5-haiku     |      $36.00 |      $1.20
gemini-1.5-pro       |      $48.75 |      $1.63
gpt-4o               |     $97.50 |      $3.25
claude-3-5-sonnet    |    $135.00 |      $4.50
```

## 8.4 Fallback та Circuit Breaker

```typescript
// reliability/fallback.ts
import { generateText } from 'ai';
import { openai } from '@ai-sdk/openai';
import { anthropic } from '@ai-sdk/anthropic';

interface FallbackConfig {
  primary: { provider: 'openai' | 'anthropic'; model: string };
  fallbacks: Array<{ provider: 'openai' | 'anthropic'; model: string }>;
  maxRetries: number;
}

const config: FallbackConfig = {
  primary: { provider: 'openai', model: 'gpt-4o' },
  fallbacks: [
    { provider: 'anthropic', model: 'claude-3-5-sonnet-20241022' },
    { provider: 'openai', model: 'gpt-4o-mini' },
  ],
  maxRetries: 3
};

class CircuitBreaker {
  private failures = new Map<string, { count: number; lastFailure: number }>();
  private threshold = 5;
  private resetTimeMs = 60000;
  
  isOpen(key: string): boolean {
    const state = this.failures.get(key);
    if (!state) return false;
    
    // Reset after timeout
    if (Date.now() - state.lastFailure > this.resetTimeMs) {
      this.failures.delete(key);
      return false;
    }
    
    return state.count >= this.threshold;
  }
  
  recordFailure(key: string): void {
    const state = this.failures.get(key) || { count: 0, lastFailure: 0 };
    state.count++;
    state.lastFailure = Date.now();
    this.failures.set(key, state);
  }
  
  recordSuccess(key: string): void {
    this.failures.delete(key);
  }
}

const circuitBreaker = new CircuitBreaker();

async function generateWithFallback(prompt: string): Promise<string> {
  const allProviders = [config.primary, ...config.fallbacks];
  
  for (const { provider, model } of allProviders) {
    const key = `${provider}:${model}`;
    
    // Skip if circuit is open
    if (circuitBreaker.isOpen(key)) {
      console.log(`⚡ Circuit open for ${key}, skipping`);
      continue;
    }
    
    try {
      console.log(`🔄 Trying ${key}...`);
      
      const modelFn = provider === 'openai' 
        ? openai(model) 
        : anthropic(model);
      
      const { text } = await generateText({
        model: modelFn,
        prompt
      });
      
      circuitBreaker.recordSuccess(key);
      console.log(`✅ Success with ${key}`);
      return text;
      
    } catch (error) {
      console.error(`❌ Failed ${key}:`, error instanceof Error ? error.message : error);
      circuitBreaker.recordFailure(key);
    }
  }
  
  throw new Error('All providers failed');
}
```

## 8.5 Production Checklist

```markdown
## ✅ Checklist для запуску в Production

### Інфраструктура
- [ ] Load balancer налаштований
- [ ] Auto-scaling для API серверів
- [ ] CDN для статики
- [ ] Redis для кешування

### Reliability
- [ ] Fallback провайдери налаштовані
- [ ] Circuit breaker реалізований
- [ ] Retry з exponential backoff
- [ ] Health checks для всіх сервісів
- [ ] Graceful degradation при проблемах

### Security
- [ ] API keys в secrets manager
- [ ] Rate limiting per user/IP
- [ ] Input validation
- [ ] Output filtering
- [ ] Audit logging

### Monitoring
- [ ] Latency метрики (p50, p95, p99)
- [ ] Error rate alerts
- [ ] Token usage tracking
- [ ] Cost alerts
- [ ] Anomaly detection

### Cost Optimization
- [ ] Caching layer (exact + semantic)
- [ ] Model selection per use case
- [ ] Budget limits per user
- [ ] Batch processing де можливо

### Compliance
- [ ] Data retention policy
- [ ] PII handling documented
- [ ] Terms of service updated
- [ ] GDPR/privacy compliance
```

---

# Модуль 9: Практичні завдання

> 🎯 **Мета:** Закріпити знання через практику.

## 📝 Завдання 1: Simple Chatbot

**Складність:** ⭐ Easy

**Опис:** Створіть простого чат-бота для консолі з пам'яттю розмови.

### 📋 Skeleton Code

```typescript
// task1-chatbot.ts
import { generateText } from 'ai';
import { openai } from '@ai-sdk/openai';
import * as readline from 'readline';

interface Message {
  role: 'user' | 'assistant' | 'system';
  content: string;
}

const conversationHistory: Message[] = [
  {
    role: 'system',
    content: 'Ти — дружній асистент. Відповідай коротко та корисно.'
  }
];

async function chat(userMessage: string): Promise<string> {
  // TODO: 
  // 1. Додати userMessage до history
  // 2. Викликати generateText з messages: conversationHistory
  // 3. Додати відповідь до history
  // 4. Повернути відповідь
  
  throw new Error('Not implemented');
}

async function main() {
  const rl = readline.createInterface({
    input: process.stdin,
    output: process.stdout
  });
  
  console.log('🤖 Chatbot ready! Type "exit" to quit.\n');
  
  const askQuestion = () => {
    rl.question('You: ', async (input) => {
      if (input.toLowerCase() === 'exit') {
        console.log('Bye!');
        rl.close();
        return;
      }
      
      const response = await chat(input);
      console.log(`Bot: ${response}\n`);
      askQuestion();
    });
  };
  
  askQuestion();
}

main();
```

### ✅ Критерії успіху

- [ ] Бот відповідає на повідомлення
- [ ] Бот пам'ятає контекст розмови
- [ ] Команда "exit" завершує програму
- [ ] Помилки API обробляються gracefully

### 💡 Підказки

<details>
<summary>Підказка 1</summary>
Використовуйте `messages` замість `prompt` в generateText
</details>

<details>
<summary>Підказка 2</summary>
Не забудьте додати відповідь assistant до history
</details>

### 🔗 Приклад рішення

[GitHub Gist: task1-solution.ts](https://gist.github.com/example/task1-chatbot-solution)

---

## 📝 Завдання 2: Structured Data Extractor

**Складність:** ⭐⭐ Medium

**Опис:** Витягніть структуровані дані з тексту резюме.

### 📋 Skeleton Code

```typescript
// task2-extractor.ts
import { generateObject } from 'ai';
import { openai } from '@ai-sdk/openai';
import { z } from 'zod';

// TODO: Визначте Zod схему для резюме
const ResumeSchema = z.object({
  // name: ...
  // email: ...
  // phone: ...
  // skills: ...
  // experience: ...
  // education: ...
});

async function extractResume(text: string) {
  // TODO: Використайте generateObject для витягування даних
  throw new Error('Not implemented');
}

// Тестові дані
const sampleResume = `
Олена Коваленко
Email: olena.k@email.com | Телефон: +380 67 123 4567

ДОСВІД РОБОТИ:
Senior Developer, TechCorp (2020-2024)
- Розробка мікросервісів на Node.js
- Керування командою з 5 розробників

Junior Developer, StartupXYZ (2018-2020)
- Frontend розробка на React

ОСВІТА:
КПІ, Комп'ютерні науки (2014-2018)

НАВИЧКИ:
TypeScript, Node.js, React, PostgreSQL, Docker, AWS
`;

// Запуск
const result = await extractResume(sampleResume);
console.log(JSON.stringify(result, null, 2));
```

### ✅ Критерії успіху

- [ ] Витягує ім'я, email, телефон
- [ ] Витягує список навичок як масив
- [ ] Парсить досвід роботи (компанія, роль, період)
- [ ] Парсить освіту
- [ ] Обробляє nullable поля (якщо щось відсутнє)

---

## 📝 Завдання 3: Weather Agent with Tools

**Складність:** ⭐⭐⭐ Hard

**Опис:** Створіть агента що відповідає на питання про погоду.

### 📋 Skeleton Code

```typescript
// task3-weather-agent.ts
import { generateText } from 'ai';
import { openai } from '@ai-sdk/openai';
import { z } from 'zod';

// Симуляція API погоди
async function getWeatherAPI(city: string): Promise<{
  temp: number;
  condition: string;
  humidity: number;
}> {
  // Симуляція
  return {
    temp: Math.floor(Math.random() * 30) - 5,
    condition: ['sunny', 'cloudy', 'rainy'][Math.floor(Math.random() * 3)],
    humidity: Math.floor(Math.random() * 50) + 30
  };
}

// TODO: Визначте tools
const tools = {
  // getWeather: { description: ..., parameters: ..., execute: ... }
  // getWeatherComparison: для порівняння двох міст
};

async function weatherAgent(question: string): Promise<string> {
  // TODO: Використайте generateText з tools та maxSteps
  throw new Error('Not implemented');
}

// Тести
const questions = [
  'Яка погода в Києві?',
  'Де тепліше — в Києві чи Львові?',
  'Чи потрібна парасолька в Одесі?'
];

for (const q of questions) {
  console.log(`Q: ${q}`);
  const answer = await weatherAgent(q);
  console.log(`A: ${answer}\n`);
}
```

### ✅ Критерії успіху

- [ ] Відповідає на прості питання про погоду
- [ ] Може порівнювати погоду в різних містах
- [ ] Дає рекомендації (парасолька, куртка)
- [ ] Логує tool calls для debugging

---

## 📝 Завдання 4: Mini RAG System

**Складність:** ⭐⭐⭐⭐ Expert

**Опис:** Побудуйте RAG систему для відповідей на питання по документації.

### 📋 Skeleton Code

```typescript
// task4-rag.ts
import { generateText, embed, embedMany } from 'ai';
import { openai } from '@ai-sdk/openai';

class MiniRAG {
  private documents: Array<{
    id: string;
    chunks: Array<{ text: string; embedding: number[] }>;
  }> = [];
  
  // TODO: Implement addDocument (chunking + embedding)
  async addDocument(id: string, content: string): Promise<void> {
    throw new Error('Not implemented');
  }
  
  // TODO: Implement search (find top K similar chunks)
  async search(query: string, topK: number = 3): Promise<string[]> {
    throw new Error('Not implemented');
  }
  
  // TODO: Implement query (search + generate answer)
  async query(question: string): Promise<string> {
    throw new Error('Not implemented');
  }
}

// Тестові документи
const docs = [
  {
    id: 'doc1',
    content: `# API Documentation
    
## Authentication
All API requests require a Bearer token in the Authorization header.
Tokens can be obtained from the /auth/token endpoint.

## Rate Limits
- Free tier: 100 requests per minute
- Pro tier: 1000 requests per minute
- Enterprise: unlimited`
  },
  {
    id: 'doc2',
    content: `# Pricing

## Plans
- Free: $0/month, basic features
- Pro: $29/month, advanced features
- Enterprise: custom pricing

## Billing
Billing occurs on the 1st of each month.
You can cancel anytime from Settings > Billing.`
  }
];

// Тест
const rag = new MiniRAG();
for (const doc of docs) {
  await rag.addDocument(doc.id, doc.content);
}

const questions = [
  'Як автентифікуватись в API?',
  'Скільки коштує Pro план?',
  'Які ліміти для безкоштовного плану?'
];

for (const q of questions) {
  console.log(`Q: ${q}`);
  const answer = await rag.query(q);
  console.log(`A: ${answer}\n`);
}
```

### ✅ Критерії успіху

- [ ] Документи розбиваються на chunks
- [ ] Chunks ембедяться і зберігаються
- [ ] Пошук знаходить релевантні chunks
- [ ] Відповіді базуються на знайдених документах
- [ ] Система каже коли не знає відповіді

---

## 🏆 Bonus Challenge: Full-Stack LLM App

**Складність:** ⭐⭐⭐⭐⭐ Master

**Опис:** Об'єднайте всі знання в production-ready застосунок.

### Вимоги:

1. **Frontend (Next.js)**
   - Chat interface
   - File upload для RAG
   - Streaming responses

2. **Backend (Node.js)**
   - Authentication
   - Rate limiting
   - Cost tracking

3. **Features**
   - Conversation memory
   - RAG over uploaded docs
   - Tool calling (weather, calculator)

4. **Production**
   - Caching
   - Error handling
   - Logging
   - Security

**Час виконання:** 8-16 годин

---

# 📚 FAQ — Найчастіші питання

## Загальні питання

### Чому TypeScript, а не Python?

TypeScript обрано через:
1. **Типізація** — менше runtime помилок
2. **Екосистема** — Next.js, Vercel AI SDK, повний стек
3. **Performance** — V8 швидший для API серверів
4. **Популярність** — більшість веб-розробників знають JS/TS

Python залишається популярним для ML/Data Science, але для production API застосунків TypeScript часто кращий вибір.

### Скільки коштує пройти весь курс?

**Мінімальний бюджет:** ~$5-10
- Використовуйте безкоштовні tier (Google AI, нові акаунти OpenAI/Anthropic)
- Використовуйте дешеві моделі (GPT-4o-mini, Gemini Flash)
- Кешуйте повторювані запити

**Комфортний бюджет:** ~$20-30
- Можливість тестувати різні моделі
- Більше експериментів

### Який провайдер найкращий для початківців?

**Рекомендація: OpenAI**
- Найкраща документація
- Найбільше прикладів в інтернеті
- Стабільний API
- GPT-4o-mini — відмінне співвідношення ціна/якість

**Альтернатива: Google AI**
- Безкоштовний tier (60 req/min)
- Gemini Flash дуже дешевий
- Великий context window

### Як перейти з development в production?

1. **Тиждень 1:** Локальна розробка, прототип
2. **Тиждень 2:** Додати caching, rate limiting, logging
3. **Тиждень 3:** Тестування, security review
4. **Тиждень 4:** Staging deployment, load testing
5. **Тиждень 5:** Production launch з моніторингом

### Чи потрібен GPU для цього курсу?

**Ні!** Весь курс використовує cloud API. GPU потрібен тільки якщо ви хочете:
- Запускати локальні моделі (Ollama, llama.cpp)
- Fine-tuning моделей
- Embeddings локально

Для навчання та більшості production use cases — cloud API достатньо.

## Технічні питання

### Як вибрати між streaming і non-streaming?

**Streaming (`streamText`):**
- Chat interfaces
- Long responses
- Better UX (показує прогрес)

**Non-streaming (`generateText`):**
- Background processing
- Batch operations
- Простіший код

### Коли використовувати RAG vs Fine-tuning?

**RAG:**
- Дані часто змінюються
- Потрібні цитати/джерела
- Швидке впровадження
- Менший бюджет

**Fine-tuning:**
- Специфічний стиль/тон
- Доменна термінологія
- Дуже часті однотипні запити
- Конфіденційність даних

### Як зменшити галюцинації?

1. **Temperature = 0** для фактичних даних
2. **Чіткі інструкції** — "Якщо не знаєш — скажи"
3. **RAG** — давайте факти в контексті
4. **Structured output** — обмежуйте можливі відповіді
5. **Verification** — перевіряйте критичні факти

### Як тестувати LLM застосунки?

1. **Deterministic tests** — temperature=0, seed
2. **Behavior tests** — перевіряємо поведінку, не exact match
3. **Multiple runs** — запускаємо тест N разів, рахуємо pass rate
4. **Golden datasets** — набір питань з очікуваними відповідями
5. **A/B testing** — порівняння версій на реальних користувачах

---

# 📖 Glossary — Глосарій термінів

| Термін | Визначення |
|--------|------------|
| **LLM** | Large Language Model — велика мовна модель |
| **Token** | Одиниця тексту для LLM (~4 символи англ.) |
| **Context Window** | Максимальний обсяг тексту що модель може обробити |
| **Temperature** | Параметр випадковості (0 = детерміновано, 1+ = креативно) |
| **Prompt** | Текст що передається моделі |
| **System Prompt** | Інструкції для моделі про її роль |
| **Completion** | Відповідь моделі |
| **Embedding** | Числовий вектор що представляє текст |
| **RAG** | Retrieval Augmented Generation — генерація з пошуком |
| **Fine-tuning** | Донавчання моделі на своїх даних |
| **Function Calling** | Можливість моделі викликати зовнішні функції |
| **Tool Use** | Синонім Function Calling |
| **Agent** | LLM що може планувати та виконувати дії |
| **ReAct** | Reason + Act — патерн для агентів |
| **MCP** | Model Context Protocol — протокол для інтеграцій |
| **Vector DB** | База даних для зберігання embeddings |
| **Chunking** | Розбиття документів на частини |
| **Semantic Search** | Пошук по змісту, не по ключових словах |
| **Hallucination** | Вигадана інформація від моделі |
| **Prompt Injection** | Атака через шкідливі інструкції |
| **Jailbreak** | Обхід обмежень моделі |
| **Streaming** | Поступова передача відповіді токен за токеном |
| **Circuit Breaker** | Патерн для захисту від каскадних збоїв |
| **Fallback** | Резервний варіант при помилці |

---

# ✅ Чекліст прогресу

```
Модуль 1: Базові концепції
☐ Розумію що таке токени
☐ Розумію context window
☐ Знаю основні параметри (temperature, maxTokens)
☐ Можу оцінити вартість запиту

Модуль 2: Chat API
☐ Використовую generateText
☐ Використовую streamText
☐ Використовую generateObject
☐ Розумію ролі (system, user, assistant)
☐ Працюю з зображеннями (multimodal)

Модуль 2.5: Помилки
☐ Обробляю context length exceeded
☐ Обробляю rate limiting
☐ Обробляю invalid JSON
☐ Захищаюсь від галюцинацій

Модуль 3: Function Calling
☐ Визначаю tools з Zod схемами
☐ Обробляю tool calls
☐ Використовую maxSteps для multi-step
☐ Обробляю помилки в tools

Модуль 4: AI Агенти
☐ Розумію ReAct pattern
☐ Реалізую memory (short-term, long-term)
☐ Встановлюю критерії зупинки
☐ Логую кроки агента

Модуль 5: MCP
☐ Налаштовую готові MCP сервери
☐ Створюю власний MCP сервер
☐ Розумію різницю tools vs resources

Модуль 6: RAG
☐ Реалізую chunking
☐ Створюю embeddings
☐ Виконую semantic search
☐ Генерую відповіді з контекстом
☐ Вимірюю recall/precision

Модуль 7: Безпека
☐ Захищаюсь від prompt injection
☐ Детектую та редактую PII
☐ Валідую контент для RAG
☐ Обмежую витрати (rate limiting, budgets)

Модуль 8: Production
☐ Налаштовую caching
☐ Реалізую fallback
☐ Використовую circuit breaker
☐ Моніторю витрати та latency
```

---

# 🚀 Наступні кроки після курсу

## Технології для вивчення

1. **Просунутий RAG**
   - Hybrid search (BM25 + semantic)
   - Re-ranking
   - Query expansion
   - Multi-hop reasoning

2. **Agent Frameworks**
   - LangGraph
   - CrewAI
   - AutoGen

3. **Evaluation**
   - RAGAS
   - DeepEval
   - Custom metrics

4. **Fine-tuning**
   - OpenAI fine-tuning API
   - LoRA adapters
   - Instruction tuning

## Проекти для портфоліо

1. **Customer Support Bot** — повна реалізація з RAG
2. **Code Assistant** — помічник для IDE
3. **Document Analyzer** — обробка контрактів/рахунків
4. **Personal Knowledge Base** — Notion/Obsidian з AI
5. **Coding Interview Prep** — тренажер з feedback

## Де шукати роботу з LLM

**Позиції:**
- AI/ML Engineer
- LLM Application Developer
- Prompt Engineer
- AI Product Manager

**Компанії:**
- AI startups (YC portfolio)
- Big Tech AI teams
- Consulting (Accenture AI, McKinsey)
- Freelance (Toptal, Upwork)

**Платформи:**
- LinkedIn (keyword: LLM, GenAI)
- AngelList / Wellfound
- AI-specific: ai-jobs.net

## Community та ресурси

**Discord:**
- Anthropic Discord
- OpenAI Discord
- LangChain Discord

**Newsletters:**
- The Batch (deeplearning.ai)
- Ahead of AI
- AI Engineering Newsletter

**Podcasts:**
- Latent Space
- Practical AI
- The AI Breakdown

**YouTube:**
- Andrej Karpathy
- Yannic Kilcher
- AI Explained

---

# 🎉 Вітаємо з завершенням курсу!

Ви тепер маєте всі базові знання для створення production-ready LLM застосунків.

**Що далі:**
1. Виконайте практичні завдання
2. Створіть свій перший проект
3. Поділіться в портфоліо
4. Продовжуйте вчитись!

---

*© 2026 LLM Course for Developers | Версія 2.0*
