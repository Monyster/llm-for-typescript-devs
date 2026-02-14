# Модуль 26: Long-running agents та durable execution

## 🎯 Що ви отримаєте з цього модуля

Після проходження ви будете:
- Розуміти проблему stateless агентів та як її вирішити
- Використовувати Inngest та Temporal для durable workflows
- Реалізовувати human-in-the-loop з очікуванням днями/тижнями
- Будувати агентів що переживають рестарти серверів

**Які задачі це дозволяє вирішувати:** Агент який запускає процес повернення товару → чекає підтвердження менеджера (2 дні) → обробляє повернення → надсилає email. Все автоматично, навіть якщо сервер перезавантажувався.

---

## 26.1 Проблема: Агенти та час

Звичайний агент працює в одному HTTP-запиті (секунди-хвилини). Але реальні бізнес-процеси тривають дні:

```
Запит на повернення:
  Хвилина 0:  Клієнт просить повернення
  Хвилина 1:  Агент перевіряє замовлення, створює заявку
  День 1-2:   Менеджер переглядає та підтверджує    ← ОЧІКУВАННЯ
  День 2:     Агент ініціює повернення коштів
  День 3-5:   Банк обробляє транзакцію              ← ОЧІКУВАННЯ
  День 5:     Агент надсилає підтвердження клієнту
```

Звичайний `generateText` з `maxSteps` не може "чекати" 2 дні.

---

## 26.2 Durable Execution: Стан що переживає всі

Durable execution — це підхід де стан workflow зберігається в базі даних і може бути відновлений після будь-якої перерви.

```
Звичайний код:                    Durable код:
  ┌─────────┐                      ┌─────────┐
  │ Process  │ ← вмирає            │ Process  │ ← вмирає
  │ RAM only │    при рестарті      │ + DB     │    але стан в DB
  └─────────┘                      └─────────┘
                                        ↓
                                   ┌─────────┐
                                   │ Process  │ ← відновлює
                                   │ + DB     │    з останнього кроку
                                   └─────────┘
```

---

## 26.3 Inngest — найпростіший спосіб

Inngest — serverless durable execution. Ідеально для Next.js та Vercel:

```typescript
import { Inngest } from 'inngest';
import { generateText, generateObject } from 'ai';
import { openai } from '@ai-sdk/openai';
import { z } from 'zod';

const inngest = new Inngest({ id: 'ai-support' });

// Durable AI workflow
const refundWorkflow = inngest.createFunction(
  { id: 'process-refund', name: 'Refund Processing Agent' },
  { event: 'support/refund.requested' },
  async ({ event, step }) => {
    const { orderId, customerId, reason } = event.data;

    // Крок 1: AI аналізує запит (виконується один раз)
    const analysis = await step.run('analyze-request', async () => {
      const { object } = await generateObject({
        model: openai('gpt-4o-mini'),
        schema: z.object({
          isEligible: z.boolean(),
          refundAmount: z.number(),
          reasoning: z.string(),
        }),
        prompt: `Замовлення ${orderId}, причина: "${reason}". Чи підлягає поверненню?`,
      });
      return object;
    });

    if (!analysis.isEligible) {
      // AI генерує відмову
      const { text } = await step.run('generate-rejection', async () => {
        return generateText({
          model: openai('gpt-4o-mini'),
          prompt: `Напиши ввічливу відмову у поверненні. Причина: ${analysis.reasoning}`,
        });
      });
      await step.run('send-rejection', () => sendEmail(customerId, text.text));
      return { status: 'rejected', reason: analysis.reasoning };
    }

    // Крок 2: Чекаємо підтвердження менеджера (до 7 днів!)
    const approval = await step.waitForEvent('wait-for-approval', {
      event: 'support/refund.approved',
      match: 'data.orderId',  // Матчимо по orderId
      timeout: '7d',           // Максимум 7 днів очікування
    });

    if (!approval) {
      // Timeout — ескалація
      await step.run('escalate', () => notifyManager('Refund timeout', orderId));
      return { status: 'escalated' };
    }

    // Крок 3: Обробка повернення
    await step.run('process-payment', async () => {
      return paymentService.refund(orderId, analysis.refundAmount);
    });

    // Крок 4: AI генерує підтвердження
    const confirmation = await step.run('generate-confirmation', async () => {
      const { text } = await generateText({
        model: openai('gpt-4o-mini'),
        prompt: `Напиши підтвердження повернення $${analysis.refundAmount} для замовлення ${orderId}`,
      });
      return text;
    });

    await step.run('send-confirmation', () => sendEmail(customerId, confirmation));

    return { status: 'completed', amount: analysis.refundAmount };
  }
);
```

### Ключові концепції Inngest

| Концепція | Що робить | Приклад |
|-----------|----------|---------|
| `step.run()` | Виконує крок один раз (idempotent) | AI аналіз, відправка email |
| `step.waitForEvent()` | Чекає зовнішню подію | Підтвердження менеджера |
| `step.sleep()` | Пауза на певний час | `step.sleep('wait-1h', '1h')` |
| `step.invoke()` | Викликає іншу функцію | Делегування sub-workflow |

---

## 26.4 Temporal — Enterprise-grade

Для складніших сценаріїв (банківські транзакції, compliance):

```typescript
import { proxyActivities, sleep, condition } from '@temporalio/workflow';

// Temporal Workflow
export async function refundWorkflow(input: RefundRequest): Promise<RefundResult> {
  const { analyzeRefund, processPayment, sendNotification } = proxyActivities({
    startToCloseTimeout: '30s',
    retry: { maximumAttempts: 3 },
  });

  // Крок 1: AI аналіз
  const analysis = await analyzeRefund(input);

  if (!analysis.isEligible) {
    await sendNotification(input.customerId, 'rejection', analysis.reasoning);
    return { status: 'rejected' };
  }

  // Крок 2: Чекаємо approval (signal від зовнішньої системи)
  let approved = false;
  const approvalHandler = setHandler('approval', (data: { approved: boolean }) => {
    approved = data.approved;
  });

  // Чекаємо до 7 днів
  const gotApproval = await condition(() => approved, '7 days');

  if (!gotApproval || !approved) {
    return { status: 'timeout_or_rejected' };
  }

  // Крок 3: Процесинг
  await processPayment(input.orderId, analysis.refundAmount);
  await sendNotification(input.customerId, 'confirmation', analysis.refundAmount);

  return { status: 'completed', amount: analysis.refundAmount };
}
```

---

## 26.5 Коли що обирати

| Критерій | Inngest | Temporal |
|----------|---------|---------|
| Складність setup | Мінімальна (SaaS) | Потрібен кластер |
| Serverless | ✅ (Vercel, AWS Lambda) | ❌ (потрібен worker) |
| Enterprise features | Базові | Повні (версіонування, replay) |
| Вартість | Free tier → $50+/міс | Self-hosted або Cloud |
| Ідеально для | Next.js проекти, стартапи | Enterprise, фінтех, compliance |

---

## Перевір себе

1. Чому звичайний агент не може "чекати" 2 дні?
2. Що таке durable execution і яку проблему це вирішує?
3. Чим `step.run()` відрізняється від звичайного `await`?
4. Реалізуйте workflow: AI класифікує → чекає approval → обробляє
5. Коли Inngest достатньо, а коли потрібен Temporal?

---

**Назад:** [← Модуль 25 — Multi-agent](25-multi-agent.md) | **Далі:** [Модуль 27 — Hybrid search →](27-hybrid-search.md)
