# Модуль 12: Agent Frameworks — AI SDK 6, OpenAI Agents, Claude Agent SDK

## 🎯 Що ви отримаєте з цього модуля

Після проходження ви будете:
- Знати 4 основні TypeScript agent frameworks та їх відмінності
- Обирати правильний фреймворк під задачу
- Будувати мульти-агентні системи з handoffs
- Реалізовувати guardrails та tracing для агентів

**Які задачі це дозволяє вирішувати:** Перейти від простих агентів з попередніх модулів до production-grade систем: мульти-агентна оркестрація, durable workflows, автоматичний retry, відстеження виконання.

---

## 💰 Бізнес-цінність

**Проблема клієнта:** "Ми хочемо складну AI-систему, але не знаємо який фреймворк обрати."
**Рішення з цього модуля:** Порівняння фреймворків з чітким decision tree: AI SDK Agents, OpenAI Agents SDK, Mastra, LangGraph.
**Як продати:** Правильний вибір фреймворку на старті заощаджує 2–3 місяці розробки. Технічний консалтинг "Вибір AI-стеку" — $2–5K.

## 12.1 Ландшафт TypeScript Agent Frameworks (2026)

| Фреймворк | Фокус | Коли обирати |
|-----------|-------|-------------|
| **Vercel AI SDK 6** | Універсальний, провайдер-агностичний | Default вибір для більшості проектів |
| **OpenAI Agents SDK** | Легкий, Handoffs, Guardrails, Voice | OpenAI-first проекти, голосові агенти |
| **Claude Agent SDK** | Computer Use, bash, файлова система | Агенти що працюють з комп'ютером |
| **LangGraph.js** | Stateful графи, durable execution | Складні workflows з branching/looping |

---

## 12.2 AI SDK 6 Agents

AI SDK 6 додав Agent interface — найпростіший спосіб створити агента:

```typescript
import { generateText, tool } from 'ai';
import { openai } from '@ai-sdk/openai';
import { z } from 'zod';

// Agent — це просто generateText з tools та maxSteps
const { text, steps } = await generateText({
  model: openai('gpt-4o-mini'),
  system: `Ти — агент підтримки інтернет-магазину.
Використовуй інструменти щоб знайти інформацію та виконати дії.
Завжди перевіряй статус замовлення перед тим як обіцяти щось клієнту.`,
  tools: {
    checkOrder: tool({
      description: 'Перевірити статус замовлення за номером',
      parameters: z.object({ orderId: z.string() }),
      execute: async ({ orderId }) => {
        return await orderService.getStatus(orderId);
      },
    }),
    initiateRefund: tool({
      description: 'Ініціювати повернення коштів',
      parameters: z.object({
        orderId: z.string(),
        reason: z.string(),
      }),
      execute: async ({ orderId, reason }) => {
        return await orderService.refund(orderId, reason);
      },
    }),
    searchFAQ: tool({
      description: 'Пошук відповіді в FAQ',
      parameters: z.object({ query: z.string() }),
      execute: async ({ query }) => {
        return await faqService.search(query);
      },
    }),
  },
  maxSteps: 10,
  prompt: 'Клієнт: "Замовлення #12345 не прийшло, хочу повернення"',
});

// steps — масив кроків які агент зробив
console.log(`Агент виконав ${steps.length} кроків`);
for (const step of steps) {
  if (step.toolCalls?.length) {
    for (const call of step.toolCalls) {
      console.log(`  → ${call.toolName}(${JSON.stringify(call.args)})`);
    }
  }
}
console.log('Відповідь:', text);
```

---

## 12.3 OpenAI Agents SDK

OpenAI Agents SDK фокусується на **Handoffs** (передача між агентами) та **Guardrails** (захисні обмеження):

```typescript
import { Agent, run } from '@openai/agents';

// Визначаємо спеціалізованих агентів
const billingAgent = new Agent({
  name: 'Billing Agent',
  instructions: 'Ти вирішуєш питання з оплатою та рахунками.',
  tools: [checkPayment, processRefund, updateBilling],
});

const technicalAgent = new Agent({
  name: 'Technical Agent',
  instructions: 'Ти вирішуєш технічні проблеми з продуктом.',
  tools: [checkLogs, restartService, createBugReport],
});

// Головний агент — маршрутизатор
const triageAgent = new Agent({
  name: 'Triage Agent',
  instructions: `Визнач тип запиту і передай спеціалізованому агенту.
Billing питання → Billing Agent
Технічні проблеми → Technical Agent`,
  handoffs: [billingAgent, technicalAgent],
  // Handoffs — агент може "передати" розмову іншому агенту
});

// Guardrails — перевірки до і після LLM
const inputGuardrail = {
  name: 'content_filter',
  execute: async (input: string) => {
    if (containsPII(input)) {
      return { tripwireTriggered: true, message: 'Виявлено PII дані' };
    }
    return { tripwireTriggered: false };
  },
};

// Запуск
const result = await run(triageAgent, 'Не можу оплатити підписку, карта відхиляється');
console.log(result.finalOutput);
// Triage → визначив billing → передав billingAgent → вирішив
```

### Ключові концепції OpenAI Agents

| Концепція | Що робить |
|-----------|----------|
| **Agent** | LLM + instructions + tools |
| **Handoffs** | Передача розмови між агентами |
| **Guardrails** | Валідація вхідних/вихідних даних |
| **Tracing** | Автоматичне логування всіх кроків |

---

## 12.4 Claude Agent SDK

Claude Agent SDK дає агенту доступ до **комп'ютера** — bash, файлова система, API:

```typescript
import { Agent } from '@anthropic-ai/claude-agent-sdk';

const agent = new Agent({
  model: 'claude-sonnet-4-5-20250929',
  tools: [
    'bash',        // Виконання shell-команд
    'filesystem',  // Читання/запис файлів
    'computer',    // Computer Use (клік, друк, скріншот)
  ],
  instructions: `Ти — DevOps-агент.
Можеш запускати команди, читати логи, редагувати конфігурації.
Перед будь-якими деструктивними діями — запитай підтвердження.`,
});

const result = await agent.run(
  'Подивись логи nginx за останню годину і знайди 5xx помилки'
);
// Агент виконає: tail /var/log/nginx/error.log | grep "5xx"
// Проаналізує результат і дасть рекомендації
```

**Де використовується:** Claude Code (Anthropic's coding agent) побудований на цьому SDK.

---

## 12.5 LangGraph.js

LangGraph.js — для **складних stateful workflows** з branching, looping, human-in-the-loop:

```typescript
import { StateGraph, MessagesAnnotation } from '@langchain/langgraph';

// Визначаємо граф виконання
const workflow = new StateGraph(MessagesAnnotation)
  .addNode('classify', classifyNode)      // Класифікувати запит
  .addNode('research', researchNode)       // Дослідити тему
  .addNode('draft', draftNode)             // Написати чернетку
  .addNode('review', humanReviewNode)      // Людина переглядає
  .addNode('publish', publishNode)         // Опублікувати

  .addEdge('__start__', 'classify')
  .addConditionalEdges('classify', routeByType, {
    simple: 'draft',
    complex: 'research',
  })
  .addEdge('research', 'draft')
  .addEdge('draft', 'review')
  .addConditionalEdges('review', checkApproval, {
    approved: 'publish',
    rejected: 'draft',        // Цикл — повернення на доопрацювання
  })
  .addEdge('publish', '__end__');

const app = workflow.compile({
  checkpointer: new PostgresSaver(db),  // Durable state — переживає рестарти
});

// Запуск з human-in-the-loop
const result = await app.invoke({
  messages: [{ role: 'user', content: 'Напиши статтю про MCP' }],
});
```

**Коли обирати LangGraph:**
- Workflow з умовними переходами (if/else у графі)
- Human-in-the-loop з очікуванням (дні, тижні)
- Durable execution — стан переживає рестарти сервера
- Паралельне виконання гілок

---

## 12.6 Порівняння: Як обрати

```
Простий агент (1 LLM + tools)
→ AI SDK 6 generateText з maxSteps

Мульти-агентна система (routing, handoffs)
→ OpenAI Agents SDK

Агент що працює з комп'ютером (bash, файли, browser)
→ Claude Agent SDK

Складний workflow (branching, looping, durable state)
→ LangGraph.js
```

| Критерій | AI SDK 6 | OpenAI Agents | Claude Agent | LangGraph |
|----------|----------|--------------|-------------|-----------|
| Простота | ⭐⭐⭐⭐⭐ | ⭐⭐⭐⭐ | ⭐⭐⭐ | ⭐⭐ |
| Провайдер-агностичність | ✅ | ❌ OpenAI only | ❌ Claude only | ✅ |
| Handoffs | ❌ | ✅ | ❌ | ✅ |
| Durable state | ❌ | ❌ | ❌ | ✅ |
| Computer Use | ❌ | ❌ | ✅ | ❌ |
| Voice/Realtime | ❌ | ✅ | ❌ | ❌ |
| npm завантажень/тиждень | 2.8M | 128K | 100K+ | 795K |

---

## Перевір себе

1. Назвіть 4 TypeScript agent frameworks та їх ключову різницю
2. Що таке Handoffs і в якому фреймворку вони є?
3. Коли AI SDK 6 достатньо, а коли потрібен LangGraph.js?
4. Реалізуйте мульти-агентну систему з triage агентом який маршрутизує до 2 спеціалістів
5. Що таке durable execution і навіщо воно?

---

**Назад:** [← Модуль 11 — MCP](11-mcp.md) | **Далі:** [Модуль 13 — Production →](13-production.md)
