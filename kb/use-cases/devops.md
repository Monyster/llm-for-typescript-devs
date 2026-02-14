# Модуль 32: AI для DevOps — Log analysis, incident response, monitoring

## 🎯 Що ви отримаєте з цього модуля

Після проходження ви будете:
- Будувати AI-агента для аналізу логів та пошуку root cause
- Реалізовувати incident response agent з автоматичним діагнозом
- Використовувати LLM для аномалій в метриках
- Автоматизувати рутинні DevOps-задачі через AI

**Які задачі це дозволяє вирішувати:** 3 AM алерт → агент аналізує логи, визначає root cause, пропонує fix, а в простих випадках виправляє сам. Скорочення MTTR (Mean Time To Resolution) з годин до хвилин.

---

## 32.1 Log Analysis Agent

```typescript
import { generateObject, generateText, tool } from 'ai';
import { openai } from '@ai-sdk/openai';
import { z } from 'zod';

const logAnalysisAgent = async (alertMessage: string) => {
  const { text } = await generateText({
    model: openai('gpt-4o-mini'),
    system: `Ти — Senior SRE агент. Аналізуй логи, знаходь root cause, пропонуй рішення.
Завжди перевіряй логи перед висновками. Не вигадуй — працюй тільки з реальними даними.`,
    tools: {
      searchLogs: tool({
        description: 'Пошук у логах за запитом (Elasticsearch/Loki)',
        parameters: z.object({
          query: z.string(),
          service: z.string().optional(),
          severity: z.enum(['error', 'warn', 'info', 'all']).default('error'),
          timeRange: z.enum(['5m', '15m', '1h', '6h', '24h']).default('1h'),
        }),
        execute: async ({ query, service, severity, timeRange }) => {
          return await elasticsearch.search({
            query, service, severity, timeRange, limit: 50,
          });
        },
      }),
      getMetrics: tool({
        description: 'Отримати метрики сервісу (CPU, memory, latency, error rate)',
        parameters: z.object({
          service: z.string(),
          metric: z.enum(['cpu', 'memory', 'latency_p99', 'error_rate', 'requests_per_sec']),
          timeRange: z.enum(['5m', '15m', '1h']).default('15m'),
        }),
        execute: async ({ service, metric, timeRange }) => {
          return await prometheus.query(service, metric, timeRange);
        },
      }),
      getServiceStatus: tool({
        description: 'Статус сервісу: running, pods, recent deployments',
        parameters: z.object({ service: z.string() }),
        execute: async ({ service }) => {
          const pods = await k8s.getPods(service);
          const deploys = await k8s.getRecentDeployments(service);
          return { pods, recentDeploys: deploys };
        },
      }),
      restartService: tool({
        description: 'Рестарт сервісу (rolling restart). Використовуй ТІЛЬКИ якщо впевнений.',
        parameters: z.object({
          service: z.string(),
          reason: z.string(),
        }),
        // Без execute — human-in-the-loop (потрібне підтвердження)
      }),
    },
    maxSteps: 10,
    prompt: `Алерт: ${alertMessage}\n\nДослідж root cause та запропонуй рішення.`,
  });

  return text;
};

// Використання при алерті
const diagnosis = await logAnalysisAgent(
  'High error rate on payment-service: 500 errors increased from 0.1% to 15% in last 5 minutes'
);
```

---

## 32.2 Incident Response Agent

```typescript
const IncidentReport = z.object({
  severity: z.enum(['P1', 'P2', 'P3', 'P4']),
  affectedServices: z.array(z.string()),
  rootCause: z.string(),
  timeline: z.array(z.object({
    time: z.string(),
    event: z.string(),
  })),
  impact: z.object({
    usersAffected: z.string(),
    revenueImpact: z.string(),
  }),
  remediation: z.object({
    immediate: z.array(z.string()),
    longTerm: z.array(z.string()),
  }),
  statusPageUpdate: z.string(),
});

async function generateIncidentReport(alertData: any, logs: string, metrics: any) {
  const { object } = await generateObject({
    model: openai('gpt-4o-mini'),
    schema: IncidentReport,
    system: `Ти — Incident Commander. Створи чіткий incident report.
Будь конкретним: вказуй часи, сервіси, числа.`,
    prompt: `Alert: ${JSON.stringify(alertData)}
Logs: ${logs}
Metrics: ${JSON.stringify(metrics)}`,
  });

  // Автоматично постимо в Slack та status page
  await slack.post('#incidents', formatIncidentReport(object));
  if (object.severity === 'P1' || object.severity === 'P2') {
    await statusPage.createIncident(object.statusPageUpdate);
  }

  return object;
}
```

---

## 32.3 Anomaly Detection з LLM

```typescript
async function analyzeMetricAnomaly(
  service: string,
  metric: string,
  current: number,
  historical: number[],
) {
  const avg = historical.reduce((a, b) => a + b, 0) / historical.length;
  const stddev = Math.sqrt(historical.reduce((a, b) => a + (b - avg) ** 2, 0) / historical.length);
  const zScore = Math.abs((current - avg) / stddev);

  // Тільки якщо аномалія статистично значима
  if (zScore < 2) return null;

  const { object } = await generateObject({
    model: openai('gpt-4o-mini'),
    schema: z.object({
      isAnomaly: z.boolean(),
      possibleCauses: z.array(z.string()),
      recommendedAction: z.string(),
      urgency: z.enum(['investigate', 'monitor', 'immediate_action']),
    }),
    prompt: `Сервіс: ${service}
Метрика: ${metric}
Поточне значення: ${current}
Середнє (7 днів): ${avg.toFixed(2)}
Стандартне відхилення: ${stddev.toFixed(2)}
Z-score: ${zScore.toFixed(2)}

Це аномалія? Які можливі причини?`,
  });

  return object;
}
```

---

## 32.4 Automation Runbooks

```typescript
// AI виконує runbook — набір кроків для вирішення типових проблем
const runbooks: Record<string, string> = {
  'high_memory': `
1. Перевір яких поди споживають найбільше пам'яті
2. Перевір чи є memory leaks в логах
3. Якщо один под — рестартуй його
4. Якщо всі поди — перевір нещодавні деплої`,

  'high_latency': `
1. Перевір latency по ендпоінтах (який найповільніший)
2. Перевір базу даних (slow queries)
3. Перевір зовнішні залежності (third-party APIs)
4. Перевір CPU та connection pool`,
};

async function executeRunbook(alertType: string, service: string) {
  const runbook = runbooks[alertType];
  if (!runbook) return 'No runbook found for this alert type';

  const { text } = await generateText({
    model: openai('gpt-4o-mini'),
    system: `Виконай runbook крок за кроком. Використовуй інструменти для перевірки.
Після кожного кроку — зафіксуй результат і переходь до наступного.`,
    tools: { searchLogs, getMetrics, getServiceStatus },
    maxSteps: 15,
    prompt: `Сервіс: ${service}\nRunbook:\n${runbook}`,
  });

  return text;
}
```

---

## Перевір себе

1. Як log analysis agent знаходить root cause?
2. Навіщо tool restartService без execute (human-in-the-loop)?
3. Реалізуйте агента який аналізує логи вашого сервісу
4. Чому z-score перевірка перед LLM-аналізом важлива?
5. Спроектуйте runbook для вашого типового інциденту

---

**Назад:** [← Модуль 31 — Наскрізний проект](31-project.md) | **Далі:** [Модуль 33 — AI для e-commerce →](33-ecommerce.md)
