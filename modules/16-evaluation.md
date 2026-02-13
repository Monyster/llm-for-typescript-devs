# Модуль 16: Evaluation та тестування LLM-застосунків

## 🎯 Що ви отримаєте з цього модуля

Після проходження ви будете:
- Розуміти чому тестування LLM відрізняється від звичайного тестування
- Використовувати LLM-as-judge для автоматичної оцінки якості
- Налаштовувати Promptfoo для CI/CD тестування промптів
- Будувати evaluation pipeline для порівняння моделей

**Які задачі це дозволяє вирішувати:** Впевнено деплоїти зміни в промптах (знати що нічого не зламали). Порівнювати моделі на ваших задачах. Знаходити регресії до того як їх помітять користувачі.

---

## 16.1 Чому тестування LLM — це особлива задача

Звичайний код: `assertEqual(add(2, 3), 5)` — результат детермінований.

LLM: один і той же промпт може дати різні відповіді. "Правильна" відповідь — це спектр, а не одне значення.

```typescript
// Звичайний тест — працює/не працює
expect(add(2, 3)).toBe(5);  // ✅ або ❌

// LLM тест — "наскільки добре?"
// Питання: "Поясни async/await"
// Як оцінити що відповідь "достатньо хороша"?
// - Чи згадано Promise? 
// - Чи є приклад коду?
// - Чи коректний приклад?
// - Чи зрозуміло для цільової аудиторії?
```

### Три підходи до оцінки

| Підхід | Коли використовувати | Точність | Масштабованість |
|--------|---------------------|----------|----------------|
| **Ручна оцінка** | Перші ітерації, складні задачі | Висока | Низька |
| **Автоматичні метрики** | Класифікація, витягування даних | Середня | Висока |
| **LLM-as-judge** | Генерація тексту, складні задачі | Висока | Висока |

---

## 16.2 Автоматичні метрики

Для задач з чіткою "правильною відповіддю":

```typescript
interface EvalResult {
  testCase: string;
  passed: boolean;
  score: number;
  details: string;
}

// Exact match — для класифікації
function evalExactMatch(expected: string, actual: string): EvalResult {
  const passed = expected.toLowerCase().trim() === actual.toLowerCase().trim();
  return {
    testCase: `Expected: "${expected}"`,
    passed,
    score: passed ? 1 : 0,
    details: passed ? 'Match' : `Got: "${actual}"`,
  };
}

// Contains — відповідь містить ключові елементи
function evalContains(required: string[], actual: string): EvalResult {
  const found = required.filter(r => actual.toLowerCase().includes(r.toLowerCase()));
  const score = found.length / required.length;
  return {
    testCase: `Must contain: ${required.join(', ')}`,
    passed: score >= 0.8,
    score,
    details: `Found ${found.length}/${required.length}: ${found.join(', ')}`,
  };
}

// JSON schema validation — для structured output
function evalSchema(schema: z.ZodSchema, actual: string): EvalResult {
  try {
    const parsed = JSON.parse(actual);
    const result = schema.safeParse(parsed);
    return {
      testCase: 'Schema validation',
      passed: result.success,
      score: result.success ? 1 : 0,
      details: result.success ? 'Valid' : result.error.message,
    };
  } catch {
    return { testCase: 'Schema validation', passed: false, score: 0, details: 'Invalid JSON' };
  }
}
```

---

## 16.3 LLM-as-Judge — AI оцінює AI

Використовуємо потужну модель для оцінки відповідей менш потужної:

```typescript
import { generateObject } from 'ai';
import { openai } from '@ai-sdk/openai';
import { z } from 'zod';

const JudgeResult = z.object({
  relevance: z.number().min(1).max(5).describe('Наскільки відповідь релевантна питанню'),
  accuracy: z.number().min(1).max(5).describe('Наскільки відповідь фактично коректна'),
  completeness: z.number().min(1).max(5).describe('Наскільки повна відповідь'),
  clarity: z.number().min(1).max(5).describe('Наскільки зрозуміла відповідь'),
  reasoning: z.string().describe('Обґрунтування оцінки'),
});

async function llmJudge(
  question: string,
  answer: string,
  criteria?: string
): Promise<z.infer<typeof JudgeResult>> {
  const { object } = await generateObject({
    model: openai('gpt-5'),  // Потужна модель як суддя
    schema: JudgeResult,
    temperature: 0,
    system: `Ти — експерт-оцінювач якості AI-відповідей.
Оцінюй об'єктивно за шкалою 1-5.
${criteria ? `Додаткові критерії: ${criteria}` : ''}`,
    prompt: `Питання: ${question}

Відповідь: ${answer}

Оціни якість цієї відповіді.`,
  });

  return object;
}

// Використання
const score = await llmJudge(
  'Як працює async/await в TypeScript?',
  generatedAnswer,
  'Відповідь має містити приклад коду та згадку про Promise'
);

console.log(`Релевантність: ${score.relevance}/5`);
console.log(`Точність: ${score.accuracy}/5`);
console.log(`Обґрунтування: ${score.reasoning}`);
```

---

## 16.4 Promptfoo — CLI для тестування промптів

Promptfoo — open-source інструмент для systematic evaluation:

```bash
# Встановлення
npx promptfoo@latest init
```

### Конфігурація

```yaml
# promptfooconfig.yaml
description: "Тест класифікації тікетів"

prompts:
  - id: classify-v1
    raw: |
      Класифікуй тікет підтримки в одну з категорій: 
      billing, technical, feature_request, complaint, other.
      Відповідай одним словом.
      
      Тікет: {{message}}
  
  - id: classify-v2
    raw: |
      Ти — експерт класифікації тікетів підтримки.
      Категорії: billing, technical, feature_request, complaint, other.
      
      Правила:
      - Якщо згадуються гроші, оплата, рахунок → billing
      - Якщо щось не працює, помилка → technical
      - Якщо "було б добре", "хочу функцію" → feature_request
      
      Тікет: {{message}}
      Категорія:

providers:
  - id: openai:gpt-4o-mini
  - id: anthropic:messages:claude-haiku-4-5-20251001

tests:
  - vars:
      message: "Не можу оплатити підписку"
    assert:
      - type: equals
        value: "billing"
  
  - vars:
      message: "Сайт видає 500 помилку при авторизації"
    assert:
      - type: equals
        value: "technical"
  
  - vars:
      message: "Було б класно мати dark mode"
    assert:
      - type: equals
        value: "feature_request"
  
  - vars:
      message: "Ваш сервіс жахливий, хочу повернення"
    assert:
      - type: equals
        value: "complaint"
```

```bash
# Запуск тестів
npx promptfoo eval

# Переглянути результати у браузері
npx promptfoo view
```

### Що показує Promptfoo

```
┌─────────────────────────┬────────────┬──────────────┬───────────┐
│ Test Case               │ classify-v1│ classify-v2  │ Expected  │
├─────────────────────────┼────────────┼──────────────┼───────────┤
│ Не можу оплатити...     │ ✅ billing │ ✅ billing   │ billing   │
│ Сайт видає 500...       │ ✅ technical│ ✅ technical │ technical │
│ Було б класно dark mode │ ❌ other   │ ✅ feature_r │ feature_r │
│ Ваш сервіс жахливий... │ ✅ complaint│ ✅ complaint │ complaint │
├─────────────────────────┼────────────┼──────────────┼───────────┤
│ Pass rate               │ 75%        │ 100%         │           │
│ Avg latency             │ 0.8s       │ 1.1s         │           │
│ Avg cost                │ $0.0001    │ $0.0002      │           │
└─────────────────────────┴────────────┴──────────────┴───────────┘
```

---

## 16.5 Evaluation Pipeline для CI/CD

```typescript
// eval/run-eval.ts — запуск перед кожним деплоєм
async function runEvaluation(): Promise<{ passed: boolean; score: number }> {
  const testCases = await loadTestCases('./eval/test-cases.json');
  let totalScore = 0;
  let passed = 0;

  for (const tc of testCases) {
    const { text } = await generateText({
      model: openai('gpt-4o-mini'),
      system: currentSystemPrompt,
      prompt: tc.input,
      temperature: 0,
    });

    const score = await llmJudge(tc.input, text, tc.criteria);
    const avgScore = (score.relevance + score.accuracy + score.completeness + score.clarity) / 4;

    totalScore += avgScore;
    if (avgScore >= tc.minScore) passed++;

    console.log(`${avgScore >= tc.minScore ? '✅' : '❌'} ${tc.name}: ${avgScore.toFixed(1)}/5`);
  }

  const overallScore = totalScore / testCases.length;
  const passRate = passed / testCases.length;

  console.log(`\nOverall: ${overallScore.toFixed(1)}/5 | Pass rate: ${(passRate * 100).toFixed(0)}%`);

  return {
    passed: passRate >= 0.9, // 90% тестів мають пройти
    score: overallScore,
  };
}

// В CI/CD
const result = await runEvaluation();
if (!result.passed) {
  process.exit(1); // Блокуємо деплой
}
```

---

## Перевір себе

1. Чому тестування LLM відрізняється від тестування звичайного коду?
2. Назвіть 3 підходи до оцінки та коли кожен використовувати
3. Що таке LLM-as-judge? Яку модель використовувати як суддю?
4. Налаштуйте Promptfoo для тестування свого промпту з 5 тест-кейсами
5. Як інтегрувати evaluation в CI/CD pipeline?

---

**Назад:** [← Модуль 15 — Безпека](15-security.md) | **Далі:** [Модуль 17 — Передові техніки →](17-advanced.md)
