# Модуль 25: Multi-agent orchestration patterns

## 🎯 Що ви отримаєте з цього модуля

Після проходження ви будете:
- Знати 5 патернів оркестрації мульти-агентних систем
- Реалізовувати sequential, parallel, supervisor, debate, pipeline
- Обирати правильний патерн під задачу
- Обробляти помилки та fallback у мульти-агентних системах

**Які задачі це дозволяє вирішувати:** Коли один агент не справляється — розбити задачу між спеціалізованими агентами. Customer support з маршрутизацією, content pipeline з рев'ю, дослідницький агент з кількома джерелами.

---

## 25.1 П'ять патернів оркестрації

```
1. Sequential:   A → B → C (конвеєр)
2. Parallel:     A, B, C одночасно → merge
3. Supervisor:   Boss → вибирає A або B або C
4. Debate:       A та B сперечаються → Judge вирішує
5. Pipeline:     A → [review] → A (з циклом доопрацювання)
```

---

## 25.2 Sequential — Конвеєр агентів

Кожен агент обробляє результат попереднього:

```typescript
import { generateText } from 'ai';
import { openai } from '@ai-sdk/openai';

interface AgentConfig {
  name: string;
  system: string;
  model?: any;
}

async function sequentialPipeline(agents: AgentConfig[], input: string): Promise<string> {
  let currentInput = input;

  for (const agent of agents) {
    console.log(`[${agent.name}] Обробляю...`);
    const { text } = await generateText({
      model: agent.model ?? openai('gpt-4o-mini'),
      system: agent.system,
      prompt: currentInput,
    });
    currentInput = text;
    console.log(`[${agent.name}] Готово: ${text.slice(0, 80)}...`);
  }

  return currentInput;
}

// Приклад: Content Pipeline
const article = await sequentialPipeline([
  {
    name: 'Researcher',
    system: 'Збери ключові факти та тези по темі. Формат: список тез з джерелами.',
  },
  {
    name: 'Writer',
    system: 'Напиши статтю на основі наданих тез. Стиль: професійний блог.',
  },
  {
    name: 'Editor',
    system: 'Відредагуй статтю: виправ помилки, покращ структуру, додай підзаголовки.',
  },
  {
    name: 'SEO Optimizer',
    system: 'Оптимізуй текст для SEO: додай ключові слова, мета-опис, alt-тексти.',
  },
], 'TypeScript у 2026: що нового');
```

---

## 25.3 Parallel — Одночасне виконання

Незалежні задачі виконуються паралельно:

```typescript
async function parallelAgents(
  agents: Array<AgentConfig & { task: string }>,
): Promise<Record<string, string>> {
  const results = await Promise.all(
    agents.map(async (agent) => {
      const { text } = await generateText({
        model: agent.model ?? openai('gpt-4o-mini'),
        system: agent.system,
        prompt: agent.task,
      });
      return { name: agent.name, result: text };
    })
  );

  return Object.fromEntries(results.map(r => [r.name, r.result]));
}

// Приклад: Мульти-аспектний аналіз
const analysis = await parallelAgents([
  {
    name: 'technical',
    system: 'Проаналізуй технічну сторону пропозиції.',
    task: proposalText,
  },
  {
    name: 'financial',
    system: 'Проаналізуй фінансову сторону та ROI.',
    task: proposalText,
  },
  {
    name: 'risk',
    system: 'Визнач ризики та способи їх мітигації.',
    task: proposalText,
  },
]);

// Мердж результатів
const { text: summary } = await generateText({
  model: openai('gpt-4o-mini'),
  system: 'Об\'єднай результати трьох аналізів у єдиний звіт.',
  prompt: `Технічний аналіз: ${analysis.technical}
Фінансовий аналіз: ${analysis.financial}
Аналіз ризиків: ${analysis.risk}`,
});
```

---

## 25.4 Supervisor — Розумний маршрутизатор

Один агент-супервізор вирішує кому делегувати задачу:

```typescript
import { generateObject, generateText, tool } from 'ai';
import { z } from 'zod';

// Спеціалізовані агенти
const specialists: Record<string, AgentConfig> = {
  billing: {
    name: 'Billing Agent',
    system: 'Ти вирішуєш питання з оплатою, підписками, рахунками.',
  },
  technical: {
    name: 'Technical Agent',
    system: 'Ти вирішуєш технічні проблеми: баги, інтеграції, API.',
  },
  sales: {
    name: 'Sales Agent',
    system: 'Ти відповідаєш на питання про продукт, ціни, функції.',
  },
};

async function supervisorAgent(message: string): Promise<string> {
  // Крок 1: Supervisor визначає кому делегувати
  const { object: routing } = await generateObject({
    model: openai('gpt-4o-mini'),
    schema: z.object({
      agent: z.enum(['billing', 'technical', 'sales']),
      confidence: z.number().min(0).max(1),
      reasoning: z.string(),
    }),
    temperature: 0,
    prompt: `Визнач якому агенту передати запит:
"${message}"

Агенти: billing (оплата), technical (техпідтримка), sales (продажі)`,
  });

  console.log(`[Supervisor] → ${routing.agent} (${(routing.confidence * 100).toFixed(0)}%)`);

  // Крок 2: Делегуємо спеціалісту
  const specialist = specialists[routing.agent];
  const { text } = await generateText({
    model: openai('gpt-4o-mini'),
    system: specialist.system,
    prompt: message,
  });

  return text;
}
```

---

## 25.5 Debate — Агенти сперечаються

Два агенти з різними позиціями обговорюють тему, третій суддя обирає найкращу відповідь:

```typescript
async function debatePattern(
  question: string,
  rounds = 2
): Promise<string> {
  let proArguments = '';
  let conArguments = '';

  for (let round = 0; round < rounds; round++) {
    // Pro-агент
    const { text: proResponse } = await generateText({
      model: openai('gpt-4o-mini'),
      system: 'Ти аргументуєш ЗА пропозицію. Будь переконливим.',
      prompt: `Питання: ${question}
${conArguments ? `Аргументи проти: ${conArguments}\nСпростуй їх і додай нові аргументи ЗА.` : 'Наведи аргументи ЗА.'}`,
    });
    proArguments = proResponse;

    // Con-агент
    const { text: conResponse } = await generateText({
      model: openai('gpt-4o-mini'),
      system: 'Ти аргументуєш ПРОТИ пропозиції. Будь критичним.',
      prompt: `Питання: ${question}
Аргументи за: ${proArguments}
Спростуй їх і наведи аргументи ПРОТИ.`,
    });
    conArguments = conResponse;
  }

  // Judge — незалежний суддя
  const { text: verdict } = await generateText({
    model: openai('gpt-5'),  // Найрозумніша модель як суддя
    system: 'Ти — неупереджений суддя. Оціни обидві сторони та дай зважену відповідь.',
    prompt: `Питання: ${question}

Аргументи ЗА:
${proArguments}

Аргументи ПРОТИ:
${conArguments}

Дай зважену відповідь з урахуванням обох сторін.`,
  });

  return verdict;
}

// Приклад
const decision = await debatePattern(
  'Чи варто мігрувати з монолітної архітектури на мікросервіси?'
);
```

---

## 25.6 Обробка помилок у мульти-агентних системах

```typescript
interface AgentResult {
  agent: string;
  success: boolean;
  result?: string;
  error?: string;
  fallback?: string;
}

async function resilientOrchestration(
  agents: AgentConfig[],
  input: string
): Promise<AgentResult[]> {
  const results: AgentResult[] = [];

  for (const agent of agents) {
    try {
      const { text } = await generateText({
        model: agent.model ?? openai('gpt-4o-mini'),
        system: agent.system,
        prompt: input,
        maxRetries: 2,
      });
      results.push({ agent: agent.name, success: true, result: text });
    } catch (error) {
      console.error(`[${agent.name}] Failed: ${error.message}`);

      // Fallback на дешевшу/швидшу модель
      try {
        const { text } = await generateText({
          model: openai('gpt-4o-mini'),
          system: `${agent.system}\nNote: working in fallback mode, be concise.`,
          prompt: input,
        });
        results.push({ agent: agent.name, success: true, result: text, fallback: 'gpt-4o-mini' });
      } catch {
        results.push({ agent: agent.name, success: false, error: error.message });
      }
    }
  }

  return results;
}
```

---

## Перевір себе

1. Назвіть 5 патернів оркестрації і коли кожен використовувати
2. Чим Supervisor відрізняється від Sequential? Переваги кожного?
3. Коли Debate pattern дає кращі результати ніж один агент?
4. Реалізуйте Parallel pattern для аналізу тексту з 3 різних аспектів
5. Як обробляти помилку коли один з агентів у pipeline відмовив?

---

**Назад:** [← Модуль 24 — OAuth для MCP](24-mcp-oauth.md) | **Далі:** [Модуль 26 — Long-running agents →](26-long-running-agents.md)
