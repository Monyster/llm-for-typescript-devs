# Модуль 30: Real-world Use Cases — Детальні кейси з повним кодом

## 🎯 Що ви отримаєте з цього модуля

Чотири production-ready кейси з повним кодом, архітектурою та метриками вартості.

---

## 30.1 Customer Support Bot

**Задача:** Автоматизувати 70% запитів підтримки.

**Архітектура:** RAG по базі знань + tools для перевірки замовлень + ескалація на людину.

```typescript
import { generateText, tool } from 'ai';
import { openai } from '@ai-sdk/openai';
import { z } from 'zod';

const supportBot = async (message: string, userId: string) => {
  const context = await ragSearch(message); // RAG по FAQ
  const userHistory = await getRecentTickets(userId);

  const { text, toolCalls } = await generateText({
    model: openai('gpt-4o-mini'),
    system: `Ти — асистент підтримки TechShop.
Відповідай на основі бази знань. Якщо не знаєш — ескалюй на людину.
Не вигадуй інформацію. Будь ввічливим.

База знань:
${context}

Історія клієнта:
${userHistory}`,
    tools: {
      checkOrder: tool({
        description: 'Перевірити статус замовлення',
        parameters: z.object({ orderId: z.string() }),
        execute: async ({ orderId }) => orderService.getStatus(orderId),
      }),
      escalateToHuman: tool({
        description: 'Передати запит живому оператору (якщо не можеш вирішити)',
        parameters: z.object({ reason: z.string(), priority: z.enum(['normal', 'urgent']) }),
        execute: async ({ reason, priority }) => ticketService.escalate(userId, reason, priority),
      }),
    },
    maxSteps: 5,
    prompt: message,
  });

  return text;
};
```

**Метрики:** $0.002/запит (GPT-4o-mini), 70% auto-resolution, P95 латентність <3с.

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
