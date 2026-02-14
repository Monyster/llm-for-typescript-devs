# Модуль 37: Testing patterns — Snapshot, regression, A/B testing промптів

## 🎯 Що ви отримаєте з цього модуля

Після проходження ви будете:
- Реалізовувати snapshot testing для промптів
- Виявляти regression при зміні промптів або моделей
- Запускати A/B тести промптів з статистичною значущістю
- Інтегрувати всі ці патерни в CI/CD

**Які задачі це дозволяє вирішувати:** Модуль 16 навчив основам evaluation. Цей модуль йде глибше: як автоматично ловити регресії, як безпечно деплоїти зміни промптів, як порівнювати два варіанти промпту на реальних даних.

---

## 37.1 Snapshot Testing для промптів

Ідея: зберігаємо "знімки" відповідей і порівнюємо при кожній зміні.

```typescript
import { generateText } from 'ai';
import { openai } from '@ai-sdk/openai';
import { readFile, writeFile } from 'fs/promises';

interface Snapshot {
  input: string;
  output: string;
  model: string;
  timestamp: string;
  promptVersion: string;
}

async function runSnapshotTest(
  testCases: Array<{ id: string; input: string }>,
  systemPrompt: string,
  promptVersion: string,
) {
  const snapshotFile = `./snapshots/${promptVersion}.json`;
  let existingSnapshots: Record<string, Snapshot> = {};

  try {
    existingSnapshots = JSON.parse(await readFile(snapshotFile, 'utf-8'));
  } catch { /* Файл не існує — перший запуск */ }

  const results: Array<{ id: string; status: 'new' | 'match' | 'changed'; diff?: string }> = [];

  for (const tc of testCases) {
    const { text } = await generateText({
      model: openai('gpt-4o-mini'),
      system: systemPrompt,
      prompt: tc.input,
      temperature: 0, // Детерміністичний для повторюваності
    });

    const existing = existingSnapshots[tc.id];

    if (!existing) {
      results.push({ id: tc.id, status: 'new' });
      existingSnapshots[tc.id] = {
        input: tc.input, output: text, model: 'gpt-4o-mini',
        timestamp: new Date().toISOString(), promptVersion,
      };
    } else if (existing.output === text) {
      results.push({ id: tc.id, status: 'match' });
    } else {
      results.push({
        id: tc.id,
        status: 'changed',
        diff: `Was: "${existing.output.slice(0, 100)}..."\nNow: "${text.slice(0, 100)}..."`,
      });
      // Оновлюємо snapshot (потрібне ручне підтвердження)
    }
  }

  await writeFile(snapshotFile, JSON.stringify(existingSnapshots, null, 2));
  return results;
}
```

---

## 37.2 Regression Detection

Автоматичне виявлення погіршення якості:

```typescript
interface RegressionResult {
  passed: boolean;
  currentScore: number;
  baselineScore: number;
  degradation: number; // Відсоток погіршення
  failedCases: string[];
}

async function detectRegression(
  testCases: Array<{ id: string; input: string; minScore: number; criteria: string }>,
  currentPrompt: string,
  baselineScores: Record<string, number>,
  threshold = 0.1, // Допустиме погіршення: 10%
): Promise<RegressionResult> {
  let currentTotal = 0;
  let baselineTotal = 0;
  const failedCases: string[] = [];

  for (const tc of testCases) {
    const { text } = await generateText({
      model: openai('gpt-4o-mini'),
      system: currentPrompt,
      prompt: tc.input,
      temperature: 0,
    });

    // LLM-as-judge (з Модуля 16)
    const score = await llmJudge(tc.input, text, tc.criteria);
    const avgScore = (score.relevance + score.accuracy + score.completeness) / 3;

    currentTotal += avgScore;
    baselineTotal += (baselineScores[tc.id] ?? avgScore);

    if (baselineScores[tc.id] && avgScore < baselineScores[tc.id] * (1 - threshold)) {
      failedCases.push(`${tc.id}: ${baselineScores[tc.id].toFixed(1)} → ${avgScore.toFixed(1)}`);
    }
  }

  const currentAvg = currentTotal / testCases.length;
  const baselineAvg = baselineTotal / testCases.length;
  const degradation = (baselineAvg - currentAvg) / baselineAvg;

  return {
    passed: degradation <= threshold && failedCases.length === 0,
    currentScore: currentAvg,
    baselineScore: baselineAvg,
    degradation,
    failedCases,
  };
}

// В CI/CD
const result = await detectRegression(testCases, newPrompt, baseline);
if (!result.passed) {
  console.error(`Regression detected: ${(result.degradation * 100).toFixed(1)}% degradation`);
  console.error('Failed cases:', result.failedCases);
  process.exit(1); // Блокуємо деплой
}
```

---

## 37.3 A/B Testing промптів

```typescript
interface ABTestConfig {
  name: string;
  variants: {
    control: string;   // Поточний промпт
    treatment: string; // Новий промпт
  };
  trafficSplit: number; // 0-1, частка treatment
  minSamples: number;   // Мінімум зразків для висновку
}

class PromptABTest {
  private results: Record<'control' | 'treatment', number[]> = {
    control: [],
    treatment: [],
  };

  constructor(private config: ABTestConfig) {}

  // Обирає варіант для користувача (sticky — один юзер завжди бачить той самий)
  getVariant(userId: string): 'control' | 'treatment' {
    const hash = createHash('md5').update(`${this.config.name}:${userId}`).digest('hex');
    const value = parseInt(hash.slice(0, 8), 16) / 0xffffffff;
    return value < this.config.trafficSplit ? 'treatment' : 'control';
  }

  getPrompt(userId: string): string {
    const variant = this.getVariant(userId);
    return this.config.variants[variant];
  }

  recordScore(variant: 'control' | 'treatment', score: number) {
    this.results[variant].push(score);
  }

  // Статистична значущість (Z-test)
  getResults(): {
    control: { avg: number; n: number };
    treatment: { avg: number; n: number };
    significant: boolean;
    winner: 'control' | 'treatment' | 'inconclusive';
  } {
    const controlAvg = mean(this.results.control);
    const treatmentAvg = mean(this.results.treatment);
    const controlStd = stdDev(this.results.control);
    const treatmentStd = stdDev(this.results.treatment);

    const n1 = this.results.control.length;
    const n2 = this.results.treatment.length;

    if (n1 < this.config.minSamples || n2 < this.config.minSamples) {
      return {
        control: { avg: controlAvg, n: n1 },
        treatment: { avg: treatmentAvg, n: n2 },
        significant: false,
        winner: 'inconclusive',
      };
    }

    // Z-score
    const se = Math.sqrt((controlStd ** 2 / n1) + (treatmentStd ** 2 / n2));
    const zScore = (treatmentAvg - controlAvg) / se;
    const significant = Math.abs(zScore) > 1.96; // 95% confidence

    return {
      control: { avg: controlAvg, n: n1 },
      treatment: { avg: treatmentAvg, n: n2 },
      significant,
      winner: significant
        ? (treatmentAvg > controlAvg ? 'treatment' : 'control')
        : 'inconclusive',
    };
  }
}
```

---

## 37.4 CI/CD Integration

```yaml
# .github/workflows/prompt-tests.yml
name: Prompt Quality Tests

on:
  pull_request:
    paths:
      - 'prompts/**'
      - 'src/ai/**'

jobs:
  eval:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Run snapshot tests
        run: npm run test:snapshots
        env:
          OPENAI_API_KEY: ${{ secrets.OPENAI_API_KEY }}

      - name: Run regression tests
        run: npm run test:regression

      - name: Comment PR with results
        uses: actions/github-script@v7
        with:
          script: |
            const results = require('./eval-results.json');
            github.rest.issues.createComment({
              issue_number: context.issue.number,
              body: `## Prompt Eval Results\n${results.summary}`
            });
```

---

## Перевір себе

1. Чим snapshot testing для промптів відрізняється від snapshot testing в Jest?
2. Як визначити "допустимий" threshold для regression?
3. Чому A/B тест промптів потребує sticky assignment (один юзер = один варіант)?
4. Реалізуйте regression test для 5 тест-кейсів свого промпту
5. Як інтегрувати prompt eval у GitHub Actions?

---

**Назад:** [← Модуль 36 — Compliance](36-compliance.md) | **Далі:** [Модуль 38 — Claude Code та Cursor →](38-ai-dev-tools.md)
