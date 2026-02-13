# Модуль 23: Mastra та інші нові фреймворки

## 🎯 Що ви отримаєте з цього модуля

Після проходження ви будете:
- Знати Mastra — TypeScript-first AI фреймворк (YC W25)
- Розуміти VoltAgent та його підхід до agent development
- Порівнювати нові фреймворки з AI SDK та LangGraph
- Обирати правильний інструмент під розмір та тип проекту

**Які задачі це дозволяє вирішувати:** Розуміти повний ландшафт TypeScript AI фреймворків і обирати інструмент який найкраще підходить саме для вашого проекту — від простих інтеграцій до складних enterprise-систем.

---

## 23.1 Mastra — TypeScript-first AI Framework

### Що це

Mastra — AI фреймворк від засновника Gatsby (Kyle Mathews). YC W25 batch, $13M фінансування. Фокус: агенти + workflows + RAG + evaluation в одному пакеті.

### Ключова ідея

Mastra об'єднує те, що зазвичай потребує 3-4 окремих бібліотеки:
- Агенти (як AI SDK)
- Workflows (як LangGraph)
- RAG (як LlamaIndex)
- Evals (як Promptfoo)

### Базовий приклад

```typescript
import { Mastra } from '@mastra/core';
import { Agent } from '@mastra/core/agent';
import { createTool } from '@mastra/core/tools';
import { z } from 'zod';

// Визначаємо інструмент
const weatherTool = createTool({
  id: 'get-weather',
  description: 'Отримати погоду для міста',
  inputSchema: z.object({
    city: z.string(),
  }),
  execute: async ({ context }) => {
    const data = await fetchWeather(context.city);
    return { temperature: data.temp, condition: data.condition };
  },
});

// Визначаємо агента
const supportAgent = new Agent({
  name: 'Support Agent',
  instructions: 'Ти — асистент підтримки. Відповідай українською.',
  model: {
    provider: 'ANTHROPIC',
    name: 'claude-sonnet-4-5-20250929',
  },
  tools: { weatherTool },
});

// Ініціалізуємо Mastra
const mastra = new Mastra({
  agents: { supportAgent },
});

// Використання
const agent = mastra.getAgent('supportAgent');
const response = await agent.generate('Яка погода в Києві?');
console.log(response.text);
```

### Mastra Workflows

Mastra має вбудовану систему workflows зі step-based виконанням:

```typescript
import { Workflow, Step } from '@mastra/core/workflow';
import { z } from 'zod';

const classifyStep = new Step({
  id: 'classify',
  inputSchema: z.object({ message: z.string() }),
  outputSchema: z.object({ category: z.string(), priority: z.string() }),
  execute: async ({ context }) => {
    // LLM класифікує повідомлення
    const result = await agent.generate(`Класифікуй: "${context.message}"`);
    return { category: result.category, priority: result.priority };
  },
});

const routeStep = new Step({
  id: 'route',
  execute: async ({ context }) => {
    // Маршрутизація на основі класифікації
    if (context.category === 'billing') return { nextAgent: 'billingAgent' };
    if (context.category === 'technical') return { nextAgent: 'technicalAgent' };
    return { nextAgent: 'generalAgent' };
  },
});

const supportWorkflow = new Workflow({
  name: 'support-pipeline',
  steps: [classifyStep, routeStep],
});

// Запуск
const result = await supportWorkflow.execute({
  message: 'Не можу оплатити підписку',
});
```

### Mastra RAG

```typescript
import { RAG } from '@mastra/rag';

const rag = new RAG({
  provider: {
    type: 'PINECONE',
    apiKey: process.env.PINECONE_API_KEY,
    indexName: 'knowledge-base',
  },
  embedding: {
    provider: 'OPEN_AI',
    model: 'text-embedding-3-small',
  },
});

// Індексація
await rag.ingest({
  documents: [
    { content: '...', metadata: { source: 'faq.md' } },
  ],
  chunkSize: 500,
  chunkOverlap: 100,
});

// Пошук
const results = await rag.query('Як повернути товар?', { topK: 5 });
```

---

## 23.2 VoltAgent — Open-source AI Agent Framework

### Що це

VoltAgent — open-source TypeScript фреймворк для побудови AI-агентів. 2,400+ GitHub stars. Фокус на простоті та observability.

### Ключова особливість

VoltAgent Console — вбудований UI для моніторингу агентів у реальному часі.

```typescript
import { VoltAgent, Agent } from '@voltagent/core';
import { VercelAIProvider } from '@voltagent/vercel-ai';
import { openai } from '@ai-sdk/openai';

// VoltAgent використовує AI SDK як провайдер
const agent = new Agent({
  name: 'Research Agent',
  description: 'Досліджує теми та збирає інформацію',
  llm: new VercelAIProvider(),
  model: openai('gpt-4o-mini'),
  tools: [searchTool, summarizeTool],
});

// Sub-agents
const writerAgent = new Agent({
  name: 'Writer Agent',
  description: 'Пише статті на основі досліджень',
  llm: new VercelAIProvider(),
  model: openai('gpt-4o-mini'),
  subAgents: [agent], // Research Agent як sub-agent
});

new VoltAgent({
  agents: { writerAgent },
  port: 3141, // VoltAgent Console на localhost:3141
});
```

---

## 23.3 Google ADK (Agent Development Kit)

### Що це

Google's TypeScript-first фреймворк для мульти-агентних систем. Вбудована підтримка Gemini та Agent-to-Agent.

```typescript
import { Agent, SequentialAgent, ParallelAgent, LoopAgent } from '@google/adk';

// Sequential — агенти працюють послідовно
const pipeline = new SequentialAgent({
  name: 'Document Pipeline',
  subAgents: [classifierAgent, extractorAgent, validatorAgent],
});

// Parallel — агенти працюють одночасно
const researcher = new ParallelAgent({
  name: 'Multi-Source Research',
  subAgents: [webSearchAgent, dbSearchAgent, apiSearchAgent],
});

// Loop — агент повторює до досягнення результату
const refiner = new LoopAgent({
  name: 'Quality Refiner',
  subAgent: writerAgent,
  maxIterations: 3,
  exitCondition: async (result) => result.qualityScore > 0.8,
});
```

---

## 23.4 Порівняння всіх фреймворків

| Критерій | AI SDK 6 | Mastra | VoltAgent | LangGraph | OpenAI Agents | Google ADK |
|----------|----------|--------|-----------|-----------|--------------|------------|
| Складність | Мінімальна | Середня | Мінімальна | Висока | Середня | Середня |
| Agents | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ |
| Workflows | ❌ | ✅ | ❌ | ✅ | ❌ | ✅ |
| RAG | Базовий | ✅ | ❌ | ❌ | ❌ | ❌ |
| Evals | ❌ | ✅ | ❌ | ❌ | ❌ | ❌ |
| Observability | DevTools | CLI | Console UI | LangSmith | Tracing | Cloud Trace |
| Провайдери | 25+ | 10+ | Через AI SDK | 10+ | OpenAI only | Google only |
| Multi-agent | Ручний | Workflows | SubAgents | Графи | Handoffs | Seq/Par/Loop |
| MCP | ✅ | ✅ | ❌ | ❌ | ❌ | ❌ |

### Коли який обирати

```
"Мені потрібен простий agent з tools"
→ AI SDK 6 (найменше коду, найбільше провайдерів)

"Мені потрібен agent + workflow + RAG + evals в одному"
→ Mastra (all-in-one)

"Мені потрібен простий agent з гарним моніторингом"
→ VoltAgent (Console UI з коробки)

"Мені потрібні складні stateful workflows з branching"
→ LangGraph.js (найгнучкіший)

"Мій стек — OpenAI, потрібні handoffs та voice"
→ OpenAI Agents SDK

"Мій стек — Google Cloud, потрібні multi-agent patterns"
→ Google ADK
```

---

## Перевір себе

1. Що робить Mastra і чим відрізняється від AI SDK?
2. Коли Mastra краще за AI SDK + окремий RAG + окремий eval?
3. Яку перевагу дає VoltAgent Console?
4. Створіть простого агента на Mastra з одним інструментом
5. Який фреймворк ви б обрали для свого проекту і чому?

---

**Назад:** [← Модуль 22 — AI SDK UI](22-ai-sdk-ui.md) | **Далі:** [Модуль 24 — OAuth для MCP →](24-mcp-oauth.md)
