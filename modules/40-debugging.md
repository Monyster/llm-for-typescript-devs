# Модуль 40: Debugging LLM apps — Галюцинації, tool failures, непередбачувана поведінка

## 🎯 Що ви отримаєте з цього модуля

Після проходження ви будете:
- Діагностувати та виправляти галюцинації LLM
- Дебажити tool call failures та зациклювання агентів
- Виявляти причини непередбачуваної поведінки
- Будувати debug tooling для LLM-застосунків

**Які задачі це дозволяє вирішувати:** LLM-застосунки ламаються по-іншому ніж звичайний код. Немає stack trace для "модель відповіла нісенітницю". Цей модуль вчить системному підходу до дебагу AI-систем.

---

## 40.1 Типи проблем LLM-застосунків

```
Звичайний код:  Crashує → Stack trace → Виправляємо
LLM-застосунок: Працює... але відповідь неправильна. Або правильна,
                але лише у 80% випадків. Або правильна сьогодні,
                але завтра провайдер оновив модель і зламалось.
```

| Проблема | Симптом | Де шукати причину |
|----------|--------|-------------------|
| Галюцинація | Модель вигадує факти | Промпт, контекст, temperature |
| Wrong tool | Агент обирає неправильний tool | Tool descriptions, system prompt |
| Loop | Агент зациклюється | maxSteps, exit conditions |
| Format error | Відповідь не парситься | Schema, Zod, output format |
| Inconsistency | Різні відповіді на те саме | Temperature, model version |
| Slow response | >10 секунд відповідь | Токени, контекст, кількість tools |

---

## 40.2 Debug галюцинацій

### Крок 1: Визначити тип галюцинації

```typescript
// Faithfulness check: чи відповідь базується на наданому контексті?
const { object: check } = await generateObject({
  model: openai('gpt-4o-mini'),
  schema: z.object({
    claims: z.array(z.object({
      claim: z.string(),
      supportedByContext: z.boolean(),
      evidence: z.string().nullable(),
    })),
    hallucinationRate: z.number().min(0).max(1),
  }),
  temperature: 0,
  prompt: `Контекст (що модель знала):
${providedContext}

Відповідь моделі:
${modelResponse}

Для кожного факту у відповіді — перевір чи він є в контексті.`,
});

if (check.hallucinationRate > 0.3) {
  console.warn(`⚠️ Hallucination rate: ${(check.hallucinationRate * 100).toFixed(0)}%`);
  check.claims.filter(c => !c.supportedByContext).forEach(c => {
    console.warn(`  Unsupported: "${c.claim}"`);
  });
}
```

### Крок 2: Типові причини та рішення

**Причина: Недостатній контекст у RAG**
- Перевірте: чи знайдені chunks релевантні?
- Рішення: покращіть chunking, додайте re-ranking

**Причина: Промпт не вказує обмеження**
- Перевірте: чи є інструкція "відповідай тільки на основі контексту"?
- Рішення: додайте explicit constraints

**Причина: Temperature > 0**
- Перевірте: яка temperature?
- Рішення: для фактичних відповідей — temperature: 0

---

## 40.3 Debug Tool Calls

### Агент обирає неправильний tool

```typescript
// Логування tool selection process
const { text, steps } = await generateText({
  model: openai('gpt-4o-mini'),
  system: systemPrompt,
  tools: myTools,
  maxSteps: 5,
  prompt: userMessage,
});

// Детальний лог кожного кроку
for (const [i, step] of steps.entries()) {
  console.log(`\n--- Step ${i + 1} ---`);
  if (step.toolCalls?.length) {
    for (const call of step.toolCalls) {
      console.log(`Tool: ${call.toolName}`);
      console.log(`Args: ${JSON.stringify(call.args)}`);
    }
  }
  if (step.toolResults?.length) {
    for (const result of step.toolResults) {
      console.log(`Result: ${JSON.stringify(result.result).slice(0, 200)}`);
    }
  }
  if (step.text) {
    console.log(`Text: ${step.text.slice(0, 200)}`);
  }
}
```

### Типові проблеми з tools

| Проблема | Причина | Рішення |
|----------|---------|---------|
| Модель не викликає tool | Погане `description` | Переписати description чіткіше |
| Модель викликає неправильний tool | Схожі описи tools | Зробити descriptions більш різними |
| Неправильні аргументи | Нечіткі `.describe()` в Zod | Додати приклади в describe |
| Зациклювання | Немає exit condition | Додати maxSteps, перевірити tool results |

---

## 40.4 Debug зациклювання агентів

```typescript
// Детектор зациклювань
function detectLoop(steps: any[]): { isLooping: boolean; pattern?: string } {
  if (steps.length < 4) return { isLooping: false };

  // Перевіряємо чи останні N кроків повторюються
  const recentTools = steps.slice(-6).map(s =>
    s.toolCalls?.map((c: any) => c.toolName).join(',') ?? 'text'
  );

  // Патерн: A,B,A,B,A,B
  for (let patternLen = 1; patternLen <= 3; patternLen++) {
    const pattern = recentTools.slice(-patternLen);
    const prev = recentTools.slice(-patternLen * 2, -patternLen);
    if (JSON.stringify(pattern) === JSON.stringify(prev)) {
      return { isLooping: true, pattern: pattern.join(' → ') };
    }
  }

  return { isLooping: false };
}
```

---

## 40.5 Debug Toolkit

```typescript
class LLMDebugger {
  private logs: any[] = [];

  // Wrapper що логує все
  async tracedGenerate(params: Parameters<typeof generateText>[0] & { debugTag: string }) {
    const start = Date.now();
    const { debugTag, ...generateParams } = params;

    try {
      const result = await generateText(generateParams);

      this.logs.push({
        tag: debugTag,
        timestamp: new Date(),
        latencyMs: Date.now() - start,
        inputTokens: result.usage.promptTokens,
        outputTokens: result.usage.completionTokens,
        steps: result.steps?.length ?? 1,
        finishReason: result.finishReason,
        success: true,
      });

      return result;
    } catch (error: any) {
      this.logs.push({
        tag: debugTag,
        timestamp: new Date(),
        latencyMs: Date.now() - start,
        error: error.message,
        status: error.status,
        success: false,
      });
      throw error;
    }
  }

  // Підсумок для дебагу
  summary() {
    const total = this.logs.length;
    const errors = this.logs.filter(l => !l.success).length;
    const avgLatency = this.logs.reduce((s, l) => s + (l.latencyMs ?? 0), 0) / total;
    const totalTokens = this.logs.reduce((s, l) => s + (l.inputTokens ?? 0) + (l.outputTokens ?? 0), 0);

    console.log(`
Debug Summary:
  Total calls: ${total}
  Errors: ${errors} (${((errors/total)*100).toFixed(1)}%)
  Avg latency: ${avgLatency.toFixed(0)}ms
  Total tokens: ${totalTokens}
  Logs by tag: ${JSON.stringify(
    this.logs.reduce((acc, l) => { acc[l.tag] = (acc[l.tag] ?? 0) + 1; return acc; }, {})
  )}
    `);
  }
}
```

---

## 40.6 Чекліст: Що перевіряти при проблемах

**Відповідь неправильна:** temperature (0 для фактів), system prompt (чи є обмеження), контекст (чи знайдено правильні chunks), модель (чи достатньо потужна).

**Агент не працює як очікувалося:** tool descriptions (чіткі?), maxSteps (достатньо?), logуємо steps (де ламається?), exit conditions (як агент завершує?).

**Performance проблеми:** кількість токенів у промпті, кількість tools (менше = швидше), стрімінг увімкнений?, кешування (prompt caching).

**Inconsistency:** temperature > 0?, модель змінилась (provider update)?, prompt injection від користувачів?

---

## Перевір себе

1. Назвіть 6 типів проблем LLM-застосунків і як їх діагностувати
2. Як виявити галюцинації автоматично (faithfulness check)?
3. Чому агент може зациклюватись і як це детектувати?
4. Побудуйте debug wrapper для своїх LLM-запитів
5. Що перевіряти першим коли "модель відповідає неправильно"?

---

**Назад:** [← Модуль 39 — Prompt Management](39-prompt-management.md)

---

## 🏁 Курс завершено!

41 модуль (00-40) покривають весь шлях від основ до production-grade AI-систем. Використовуйте ці знання щоб будувати, монетизувати та масштабувати AI-продукти.
