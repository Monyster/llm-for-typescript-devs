# Модуль 39: Prompt Management Platforms — Langfuse, Braintrust

## 🎯 Що ви отримаєте з цього модуля

Після проходження ви будете:
- Централізовано управляти промптами (версіонування, deploy, rollback)
- Використовувати Langfuse Prompts для production prompt management
- Розділяти відповідальність: prompt engineer змінює промпт без деплою коду
- Моніторити якість промптів у реальному часі

**Які задачі це дозволяє вирішувати:** Промпт hardcoded у коді → кожна зміна потребує деплою. Prompt management platform дозволяє змінювати промпти без зміни коду, A/B тестувати, відслідковувати якість.

---

## 39.1 Проблема: Промпт як код

```typescript
// ❌ Промпт hardcoded — зміна потребує PR + review + deploy
const SYSTEM_PROMPT = `Ти — асистент підтримки TechShop.
Відповідай ввічливо та коротко.
Якщо не знаєш — скажи що не знаєш.`;

const { text } = await generateText({
  model: openai('gpt-4o-mini'),
  system: SYSTEM_PROMPT,
  prompt: userMessage,
});
```

**Проблеми:**
- Prompt engineer не може змінити промпт без розробника
- Немає версіонування промптів
- Немає rollback якщо новий промпт гірший
- Немає моніторингу якості по версіях

---

## 39.2 Langfuse Prompts

Langfuse — open-source platform для LLM observability + prompt management:

```typescript
import { Langfuse } from 'langfuse';

const langfuse = new Langfuse({
  publicKey: process.env.LANGFUSE_PUBLIC_KEY,
  secretKey: process.env.LANGFUSE_SECRET_KEY,
});

// ✅ Промпт завантажується з Langfuse (не з коду)
async function getSystemPrompt(): Promise<string> {
  const prompt = await langfuse.getPrompt('support-agent');
  return prompt.compile(); // Повертає поточну production версію
}

// З змінними (шаблон)
async function getPromptWithVars(vars: Record<string, string>): Promise<string> {
  const prompt = await langfuse.getPrompt('ticket-classifier');
  return prompt.compile(vars);
  // Шаблон: "Класифікуй тікет для {{company_name}}: {{ticket_text}}"
  // Результат: "Класифікуй тікет для TechShop: Не можу оплатити..."
}

// Використання
const systemPrompt = await getSystemPrompt();
const { text } = await generateText({
  model: openai('gpt-4o-mini'),
  system: systemPrompt,
  prompt: userMessage,
});
```

### Langfuse Dashboard

У веб-інтерфейсі Langfuse можна:
- Створювати та редагувати промпти
- Версіонувати (v1, v2, v3...)
- Позначати версію як "production"
- Бачити метрики якості по кожній версії
- Робити rollback одним кліком

---

## 39.3 Braintrust

Braintrust фокусується на **eval + prompt management** разом:

```typescript
import { initLogger, wrapOpenAI } from 'braintrust';

const logger = initLogger({ projectName: 'support-bot' });

// Braintrust автоматично логує всі запити
const client = wrapOpenAI(new OpenAI());

// Промпти з Braintrust
const prompt = await logger.loadPrompt('support-system-prompt', {
  defaults: { model: 'gpt-4o-mini', temperature: 0.3 },
});

const response = await client.chat.completions.create({
  ...prompt,
  messages: [...prompt.messages, { role: 'user', content: userMessage }],
});
```

---

## 39.4 DIY Prompt Management

Для менших проектів — можна зробити самостійно:

```typescript
interface PromptVersion {
  id: string;
  name: string;
  version: number;
  content: string;
  isProduction: boolean;
  createdAt: Date;
  createdBy: string;
}

class SimplePromptManager {
  // Отримати production версію
  async getPrompt(name: string): Promise<string> {
    const row = await db.query(
      'SELECT content FROM prompts WHERE name = $1 AND is_production = true',
      [name]
    );
    if (!row.rows[0]) throw new Error(`Prompt "${name}" not found`);
    return row.rows[0].content;
  }

  // Створити нову версію
  async createVersion(name: string, content: string, author: string): Promise<number> {
    const result = await db.query(`
      INSERT INTO prompts (name, version, content, is_production, created_by)
      VALUES ($1, COALESCE((SELECT MAX(version) FROM prompts WHERE name = $1), 0) + 1,
              $2, false, $3)
      RETURNING version
    `, [name, content, author]);
    return result.rows[0].version;
  }

  // Промоутнути версію в production
  async promote(name: string, version: number) {
    await db.query('UPDATE prompts SET is_production = false WHERE name = $1', [name]);
    await db.query('UPDATE prompts SET is_production = true WHERE name = $1 AND version = $2',
      [name, version]);
  }

  // Rollback
  async rollback(name: string) {
    const current = await db.query(
      'SELECT version FROM prompts WHERE name = $1 AND is_production = true', [name]
    );
    const prevVersion = current.rows[0].version - 1;
    await this.promote(name, prevVersion);
  }
}
```

---

## 39.5 Порівняння підходів

| Підхід | Плюси | Мінуси | Коли |
|--------|-------|--------|------|
| **Hardcoded** | Просто, під контролем git | Зміна = деплой | MVP, 1-2 промпти |
| **Env variables** | Зміна без деплою | Немає версіонування | Простий production |
| **DIY (DB)** | Повний контроль | Потрібно будувати UI | Середній проект |
| **Langfuse** | Observability + prompts + evals | Додаткова залежність | Production з командою |
| **Braintrust** | Eval-first підхід | SaaS lock-in | Коли eval критичний |

---

## Перевір себе

1. Чому промпт у коді — це проблема для production?
2. Як Langfuse дозволяє змінювати промпт без деплою коду?
3. Реалізуйте простий prompt manager з версіонуванням і rollback
4. Коли DIY достатньо, а коли потрібен Langfuse?
5. Як prompt management інтегрується з A/B testing промптів?

---

**Назад:** [← Модуль 38 — AI Dev Tools](38-ai-dev-tools.md) | **Далі:** [Модуль 40 — Debugging LLM apps →](40-debugging.md)
