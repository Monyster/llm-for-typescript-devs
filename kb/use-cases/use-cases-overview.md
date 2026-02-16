# Модуль 30: Real-world Use Cases — Детальні кейси з повним кодом

## 🎯 Що ви отримаєте з цього модуля

Чотири production-ready кейси з повним кодом, архітектурою та метриками вартості.

---

## 30.1 Customer Support Bot

**Задача:** Автоматизувати 70% запитів підтримки інтернет-магазину.

**Архітектура:** RAG по базі знань + tools для дій + reasoning для складних кейсів + ескалація на людину.

### System Prompt з XML-тегами

XML-теги структурують промпт так, що модель чітко розділяє інструкції, контекст, і дані. Це зменшує "галюцінації" і дає передбачувану поведінку.

```typescript
function buildSystemPrompt(params: {
  knowledgeBase: string;
  userProfile: UserProfile;
  recentOrders: Order[];
  conversationSummary?: string;
}): string {
  return `
<role>
Ти — AI-асистент підтримки інтернет-магазину TechShop.
Твоє ім'я: Тех. Ти спілкуєшся українською.
</role>

<rules>
- Відповідай ВИКЛЮЧНО на основі інформації з <knowledge_base> та результатів інструментів
- Якщо відповіді немає в базі знань і інструменти не допомогли — чесно скажи що не маєш інформації і запропонуй ескалацію
- НІКОЛИ не вигадуй ціни, терміни доставки, наявність товарів, номери замовлень
- Максимум 3-4 речення у відповіді, якщо клієнт не просить детальніше
- Якщо клієнт роздратований — визнай проблему, вибачся, запропонуй конкретне рішення
- Не пропонуй знижки і компенсації самостійно — ескалюй на оператора
</rules>

<tone>
Ввічливий, лаконічний, конкретний. Без надмірної формальності.
Добре: "Ваше замовлення ORD-12345 вже у дорозі, очікуйте завтра."
Погано: "Шановний клієнте! Дякуємо за ваше звернення! Ми з радістю повідомляємо..."
</tone>

<escalation_triggers>
Негайно ескалюй на людину якщо:
- Клієнт вимагає повернення грошей
- Клієнт скаржиться на якість товару
- Клієнт згадує юриста, суд, або захист прав споживачів
- Питання про персональні дані або GDPR
- Ти не можеш вирішити проблему за 3 кроки
</escalation_triggers>

<knowledge_base>
${params.knowledgeBase}
</knowledge_base>

<customer_profile>
Ім'я: ${params.userProfile.name}
Email: ${params.userProfile.email}
Тип: ${params.userProfile.tier} <!-- standard | premium | vip -->
Клієнт з: ${params.userProfile.since}
Відкритих тікетів: ${params.userProfile.openTickets}
</customer_profile>

<recent_orders>
${params.recentOrders.map(o => 
  `- ${o.id}: ${o.status} | ${o.items.join(', ')} | ${o.total} грн | ${o.date}`
).join('\n')}
</recent_orders>

${params.conversationSummary ? `
<conversation_summary>
${params.conversationSummary}
</conversation_summary>
` : ''}

<response_format>
Відповідай напряму клієнту. Не пиши "Відповідь:" чи інші мета-коментарі.
Якщо використовуєш інструмент — дочекайся результату перед відповіддю клієнту.
</response_format>
`;
}
```

### Tools: дії агента

```typescript
import { generateText, tool } from 'ai';
import { anthropic } from '@ai-sdk/anthropic';
import { z } from 'zod';

const supportTools = {
  checkOrderStatus: tool({
    description: `Перевіряє поточний статус замовлення за його номером.
Використовуй коли клієнт запитує "де моє замовлення" або називає номер ORD-XXXXX.
НЕ використовуй якщо клієнт не надав номер — спитай його спочатку.`,
    parameters: z.object({
      orderId: z.string().describe('Номер замовлення у форматі ORD-XXXXX'),
    }),
    execute: async ({ orderId }) => {
      const order = await orderService.getStatus(orderId);
      if (!order) return { found: false, message: 'Замовлення не знайдено' };
      return {
        found: true,
        id: order.id,
        status: order.status,
        estimatedDelivery: order.estimatedDelivery,
        trackingUrl: order.trackingUrl,
        items: order.items.map(i => i.name),
      };
    },
  }),

  searchKnowledgeBase: tool({
    description: `Шукає додаткову інформацію в базі знань TechShop.
Використовуй коли відповіді немає в <knowledge_base> з system prompt.
Наприклад: характеристики конкретного товару, умови акції, технічні деталі.`,
    parameters: z.object({
      query: z.string().describe('Пошуковий запит українською, максимально конкретний'),
    }),
    execute: async ({ query }) => {
      const results = await vectorSearch(query, { topK: 3, minScore: 0.7 });
      if (results.length === 0) return { found: false, message: 'Нічого не знайдено' };
      return {
        found: true,
        results: results.map(r => ({
          content: r.content,
          source: r.metadata.source,
          relevance: r.score,
        })),
      };
    },
  }),

  checkProductAvailability: tool({
    description: `Перевіряє наявність товару на складі.
Використовуй коли клієнт питає "чи є в наявності", "коли буде".`,
    parameters: z.object({
      productName: z.string().describe('Назва товару або його частина'),
    }),
    execute: async ({ productName }) => {
      const products = await catalogService.search(productName);
      return products.map(p => ({
        name: p.name,
        inStock: p.quantity > 0,
        quantity: p.quantity,
        price: p.price,
        estimatedRestock: p.quantity === 0 ? p.restockDate : null,
      }));
    },
  }),

  escalateToHuman: tool({
    description: `Передає розмову живому оператору.
Використовуй ОБОВ'ЯЗКОВО коли:
- Спрацьовують triggers з <escalation_triggers>
- Ти не можеш вирішити проблему за 3 кроки
- Клієнт прямо просить поговорити з людиною`,
    parameters: z.object({
      reason: z.string().describe('Чому ескалюємо (для оператора, клієнт це не бачить)'),
      summary: z.string().describe('Короткий опис проблеми клієнта (2-3 речення)'),
      priority: z.enum(['normal', 'high', 'urgent']).describe(
        'normal: загальне питання. high: незадоволений клієнт. urgent: VIP або юридичні загрози'
      ),
    }),
    execute: async ({ reason, summary, priority }) => {
      const ticket = await ticketService.escalate({
        reason, summary, priority,
        conversationHistory: await getConversationHistory(),
      });
      return {
        ticketId: ticket.id,
        estimatedWaitTime: ticket.estimatedWait,
        message: `Оператор отримає всю історію розмови і зв'яжеться протягом ${ticket.estimatedWait}.`,
      };
    },
  }),
};
```

### Головний обробник

```typescript
interface SupportRequest {
  message: string;
  userId: string;
  conversationId: string;
}

async function handleSupportMessage(req: SupportRequest) {
  // 1. Збираємо контекст (Context Engineering)
  const [userProfile, recentOrders, ragContext, conversationHistory] = await Promise.all([
    getUserProfile(req.userId),
    getRecentOrders(req.userId, { limit: 5 }),
    ragSearch(req.message, { topK: 3 }),
    getOptimizedHistory(req.conversationId, { maxMessages: 10 }),
  ]);

  // 2. Compaction для довгих розмов
  const optimizedHistory = conversationHistory.length > 10
    ? await compactHistory(conversationHistory)
    : conversationHistory;

  // 3. Визначаємо чи потрібен reasoning (складна скарга vs просте питання)
  const needsReasoning = userProfile.tier === 'vip' 
    || conversationHistory.some(m => m.sentiment === 'negative');

  // 4. Генеруємо відповідь
  const { text, toolCalls, usage } = await generateText({
    model: anthropic('claude-sonnet-4-5-20250929'),
    system: buildSystemPrompt({
      knowledgeBase: ragContext.map(r => r.content).join('\n---\n'),
      userProfile,
      recentOrders,
      conversationSummary: optimizedHistory.summary,
    }),
    messages: [
      ...optimizedHistory.messages,
      { role: 'user', content: req.message },
    ],
    tools: supportTools,
    maxSteps: 5,
    ...(needsReasoning && {
      providerOptions: {
        anthropic: {
          thinking: { type: 'enabled', budget_tokens: 2048 },
        },
      },
      maxTokens: 4096,
    }),
  });

  // 5. Зберігаємо і логуємо
  await saveMessage(req.conversationId, req.message, text);
  await logMetrics({
    conversationId: req.conversationId,
    tokensUsed: usage?.totalTokens,
    toolsUsed: toolCalls?.map(t => t.toolName),
    escalated: toolCalls?.some(t => t.toolName === 'escalateToHuman'),
    reasoning: needsReasoning,
  });

  return { text, escalated: toolCalls?.some(t => t.toolName === 'escalateToHuman') };
}
```

### Приклад діалогу

```
Клієнт: Привіт, я замовив навушники 3 дні тому, досі нічого. Замовлення ORD-84721

[thinking] Клієнт чекає 3 дні і незадоволений. Перевірю статус замовлення.
→ tool_call: checkOrderStatus({ orderId: "ORD-84721" })
← tool_result: { found: true, status: "shipped", estimatedDelivery: "завтра, 17 лютого",
                  trackingUrl: "https://novaposhta.ua/track/2040..." }

Бот: Ваше замовлення ORD-84721 вже у дорозі! Очікувана доставка — завтра, 17 лютого.
Відстежити можна тут: https://novaposhta.ua/track/2040...
Якщо завтра не отримаєте — напишіть, розберемося.

---

Клієнт: А можна дізнатись чи є AirPods Pro 3 в наявності?

→ tool_call: checkProductAvailability({ productName: "AirPods Pro 3" })
← tool_result: [{ name: "Apple AirPods Pro 3", inStock: true, quantity: 12, price: 9999 }]

Бот: Так, AirPods Pro 3 є в наявності — 9 999 грн. Можу допомогти з замовленням?

---

Клієнт: Навушники прийшли битими! Хочу повернення грошей!

[thinking] Клієнт скаржиться на якість + вимагає повернення. 
Це escalation_trigger — ескалюю одразу.
→ tool_call: escalateToHuman({ 
     reason: "Пошкоджений товар, вимагає refund",
     summary: "Клієнт отримав пошкоджені навушники з ORD-84721, вимагає повернення коштів",
     priority: "high" 
   })

Бот: Дуже шкода що навушники прийшли пошкодженими. Я передав ваш запит оператору — 
він зв'яжеться протягом 15 хвилин і допоможе з поверненням. 
Вся історія нашої розмови у нього буде.
```

### Чому XML-теги працюють краще

```
Без XML-тегів:
  "Ти асистент. Ось правила: ... Ось база знань: ... Ось профіль: ..."
  → Модель плутає де закінчуються правила і починаються дані
  → Іноді "цитує" правила клієнту

З XML-тегами:
  <rules>...</rules>  <knowledge_base>...</knowledge_base>  <customer_profile>...</customer_profile>
  → Чітке розділення інструкцій і даних
  → Модель розуміє що <rules> — для неї, а <knowledge_base> — джерело фактів
  → Можна посилатися: "Відповідай на основі <knowledge_base>"
  → Зручний prompt caching: <rules> + <tone> кешуються, <knowledge_base> динамічний
```

**Метрики:** ~$0.003/запит (Sonnet, без reasoning) або ~$0.02/запит (з reasoning для складних кейсів). 70% auto-resolution, P95 латентність <3с.

---

## 30.2 Code Review Assistant

**Задача:** Автоматичний рев'ю PR на баги, безпеку, code style.

```typescript
const reviewPR = async (diff: string, context: string) => {
  const { object } = await generateObject({
    model: openai('gpt-4o-mini'),
    schema: z.object({
      summary: z.string(),
      issues: z.array(z.object({
        severity: z.enum(['critical', 'warning', 'suggestion']),
        file: z.string(),
        line: z.number().optional(),
        description: z.string(),
        suggestion: z.string(),
      })),
      securityConcerns: z.array(z.string()),
      overallScore: z.number().min(1).max(10),
      approved: z.boolean(),
    }),
    system: `Ти — senior TypeScript code reviewer.
Перевіряй: баги, безпека, TypeScript best practices, чистота коду.
Будь конкретним — вказуй файл та рядок.`,
    prompt: `PR Diff:\n${diff}\n\nContext:\n${context}`,
  });

  return object;
};
```

---

## 30.3 Document Processing Pipeline

**Задача:** Автоматичне витягування даних з PDF рахунків.

```typescript
const processInvoice = async (pdfBuffer: Buffer) => {
  const { object } = await generateObject({
    model: anthropic('claude-sonnet-4-5-20250929'),
    schema: z.object({
      invoiceNumber: z.string(),
      date: z.string(),
      vendor: z.object({ name: z.string(), taxId: z.string().nullable() }),
      items: z.array(z.object({
        description: z.string(),
        quantity: z.number(),
        unitPrice: z.number(),
        total: z.number(),
      })),
      totalAmount: z.number(),
      currency: z.string(),
    }),
    messages: [{
      role: 'user',
      content: [
        { type: 'document', source: { type: 'base64', mediaType: 'application/pdf', data: pdfBuffer.toString('base64') } },
        { type: 'text', text: 'Витягни всі дані з цього рахунку.' },
      ],
    }],
  });

  await db.invoices.create(object);
  return object;
};
```

**Метрики:** $0.05/рахунок (Claude Sonnet), 95% accuracy, 5с на документ.

---

## 30.4 Email Automation

**Задача:** Класифікація + автоматичні відповіді на типові email.

```typescript
const processEmail = async (email: { from: string; subject: string; body: string }) => {
  // Крок 1: Класифікація
  const { object: classification } = await generateObject({
    model: openai('gpt-4o-mini'),
    schema: z.object({
      category: z.enum(['inquiry', 'complaint', 'order', 'spam', 'other']),
      urgency: z.enum(['low', 'medium', 'high']),
      autoReplyPossible: z.boolean(),
    }),
    prompt: `From: ${email.from}\nSubject: ${email.subject}\n\n${email.body}`,
  });

  if (classification.category === 'spam') return { action: 'archived' };

  // Крок 2: Автовідповідь (якщо можливо)
  if (classification.autoReplyPossible) {
    const { text: reply } = await generateText({
      model: openai('gpt-4o-mini'),
      system: 'Напиши професійну відповідь на email. Підпис: TechShop Support Team.',
      prompt: `Email:\n${email.body}\n\nНапиши відповідь.`,
    });

    return { action: 'auto_reply', reply, classification };
  }

  return { action: 'forwarded_to_human', classification };
};
```

---

## Перевір себе

1. Яку архітектуру має Customer Support Bot? Які компоненти?
2. Як Code Review Assistant визначає severity помилки?
3. Чому Claude Sonnet краще для PDF ніж GPT-4o-mini?
4. Реалізуйте один з кейсів для свого проекту
5. Порахуйте місячну вартість кожного кейсу при 1000 запитів/день

---

**Назад:** [← Модуль 29 — Edge AI](29-edge-local.md) | **Далі:** [Модуль 31 — Наскрізний проект →](31-project.md)
