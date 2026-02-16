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

**Задача:** Автоматичний рев'ю PR в GitHub — баги, безпека, TypeScript best practices. Коментарі прямо в PR.

**Архітектура:** GitHub Webhook → аналіз diff з reasoning → structured output → GitHub API коментарі.

### System Prompt

```typescript
function buildReviewPrompt(params: {
  projectRules: string;
  prDescription: string;
  changedFiles: string[];
}): string {
  return `
<role>
Ти — senior TypeScript code reviewer з 10+ роками досвіду.
Твоя задача — знайти реальні проблеми, а не прискіпуватись до стилю.
</role>

<review_priorities>
1. CRITICAL: Баги які зламають production (null access, race conditions, memory leaks, infinite loops)
2. CRITICAL: Вразливості безпеки (SQL injection, XSS, secret exposure, path traversal)
3. WARNING: Порушення типізації (any, type assertions без причини, missing null checks)
4. WARNING: Performance (N+1 queries, unnecessary re-renders, missing indexes)
5. SUGGESTION: Покращення читабельності та maintainability
</review_priorities>

<project_rules>
${params.projectRules}
</project_rules>

<review_guidelines>
- Коментуй ТІЛЬКИ рядки які змінились (є в diff). Не коментуй існуючий код
- Кожна проблема має мати конкретну пропозицію фіксу з кодом
- Якщо зміна виглядає правильною — не вигадуй проблему, щоб "щось написати"
- Для critical і warning — поясни ЧОМУ це проблема (яка помилка може статись)
- Не коментуй code style якщо є більш серйозні проблеми
- Максимум 10 коментарів на PR (фокус на найважливішому)
</review_guidelines>

<pr_context>
Опис PR: ${params.prDescription}
Змінені файли: ${params.changedFiles.join(', ')}
</pr_context>
`;
}
```

### Structured Output з Zod

```typescript
import { generateObject } from 'ai';
import { anthropic } from '@ai-sdk/anthropic';
import { z } from 'zod';

const ReviewIssueSchema = z.object({
  file: z.string().describe('Повний шлях файлу, наприклад src/services/auth.ts'),
  line: z.number().describe('Номер рядка де проблема (з diff)'),
  severity: z.enum(['critical', 'warning', 'suggestion']),
  category: z.enum([
    'bug', 'security', 'typing', 'performance',
    'error-handling', 'readability', 'testing',
  ]),
  description: z.string().describe('Що не так і чому це проблема (2-3 речення)'),
  suggestedFix: z.string().describe('Конкретний код-фікс який вирішує проблему'),
});

const ReviewResultSchema = z.object({
  summary: z.string().describe('1-2 речення: що робить PR і загальна якість'),
  issues: z.array(ReviewIssueSchema).describe('Масив знайдених проблем, максимум 10'),
  positives: z.array(z.string()).describe('Що зроблено добре (1-3 пункти)'),
  overallScore: z.number().min(1).max(10).describe(
    '1-3: блокуючі проблеми, 4-6: потребує змін, 7-8: minor comments, 9-10: відмінно'
  ),
  verdict: z.enum(['approve', 'request_changes', 'comment']),
});
```

### Головний обробник

```typescript
async function reviewPullRequest(pr: GitHubPR) {
  // 1. Завантажуємо diff та контекст
  const [diff, projectRules, prComments] = await Promise.all([
    github.getPRDiff(pr.owner, pr.repo, pr.number),
    loadProjectRules(pr.repo),        // .github/review-rules.md або CLAUDE.md
    github.getPRComments(pr.owner, pr.repo, pr.number),
  ]);

  // 2. Розбиваємо великий diff на файли (token budget)
  const fileDiffs = parseDiffByFile(diff);
  const changedFiles = fileDiffs.map(f => f.filename);

  // 3. Фільтруємо файли що не потребують рев'ю
  const reviewableFiles = fileDiffs.filter(f =>
    !f.filename.endsWith('.lock') &&
    !f.filename.endsWith('.snap') &&
    !f.filename.startsWith('dist/') &&
    f.additions + f.deletions > 0
  );

  // 4. Для великих PR — рев'ю по файлах, потім синтез
  if (estimateTokens(diff) > 30000) {
    return await reviewLargePR(reviewableFiles, pr, projectRules);
  }

  // 5. Рев'ю з reasoning (code review потребує deep analysis)
  const { object: review } = await generateObject({
    model: anthropic('claude-sonnet-4-5-20250929'),
    schema: ReviewResultSchema,
    system: buildReviewPrompt({
      projectRules,
      prDescription: pr.description,
      changedFiles,
    }),
    prompt: formatDiffForReview(reviewableFiles),
    maxTokens: 8000,
    providerOptions: {
      anthropic: {
        thinking: { type: 'enabled', budget_tokens: 4096 },
      },
    },
  });

  // 6. Публікуємо результат в GitHub
  await postReviewToGitHub(pr, review);

  return review;
}
```

### Рев'ю великих PR (multi-agent)

```typescript
async function reviewLargePR(
  files: FileDiff[],
  pr: GitHubPR,
  projectRules: string
): Promise<z.infer<typeof ReviewResultSchema>> {
  // Sub-agent рев'юїть кожен файл окремо (ізольований контекст)
  const fileReviews = await Promise.all(
    files.map(async (file) => {
      const { object } = await generateObject({
        model: anthropic('claude-sonnet-4-5-20250929'),
        schema: z.object({
          file: z.string(),
          issues: z.array(ReviewIssueSchema),
          fileScore: z.number().min(1).max(10),
        }),
        system: buildReviewPrompt({
          projectRules,
          prDescription: pr.description,
          changedFiles: [file.filename],
        }),
        prompt: `Файл: ${file.filename}\n\n${file.diff}`,
      });
      return object;
    })
  );

  // Lead agent синтезує результати
  const allIssues = fileReviews.flatMap(r => r.issues);
  const avgScore = fileReviews.reduce((s, r) => s + r.fileScore, 0) / fileReviews.length;

  const { object: synthesis } = await generateObject({
    model: anthropic('claude-sonnet-4-5-20250929'),
    schema: ReviewResultSchema,
    system: 'Ти — lead reviewer. Синтезуй результати рев''ю окремих файлів у фінальний звіт.',
    prompt: `PR: ${pr.description}
Результати рев'ю файлів: ${JSON.stringify(fileReviews, null, 2)}
Середній score: ${avgScore.toFixed(1)}`,
    providerOptions: {
      anthropic: { thinking: { type: 'enabled', budget_tokens: 2048 } },
    },
  });

  return synthesis;
}
```

### Публікація в GitHub

```typescript
async function postReviewToGitHub(
  pr: GitHubPR,
  review: z.infer<typeof ReviewResultSchema>
) {
  // Inline коментарі до конкретних рядків
  const comments = review.issues.map(issue => ({
    path: issue.file,
    line: issue.line,
    body: `**${issue.severity.toUpperCase()}** (${issue.category})\n\n${issue.description}\n\n**Запропонований фікс:**\n\`\`\`typescript\n${issue.suggestedFix}\n\`\`\``,
  }));

  // Summary коментар
  const summaryBody = `## 🤖 AI Code Review

${review.summary}

**Score: ${review.overallScore}/10** | **Verdict: ${review.verdict}**

${review.positives.length > 0 ? `### ✅ Що добре\n${review.positives.map(p => `- ${p}`).join('\n')}` : ''}

### 📊 Issues: ${review.issues.filter(i => i.severity === 'critical').length} critical, ${review.issues.filter(i => i.severity === 'warning').length} warnings, ${review.issues.filter(i => i.severity === 'suggestion').length} suggestions`;

  await github.createReview(pr.owner, pr.repo, pr.number, {
    event: review.verdict === 'approve' ? 'APPROVE'
         : review.verdict === 'request_changes' ? 'REQUEST_CHANGES'
         : 'COMMENT',
    body: summaryBody,
    comments,
  });
}
```

**Метрики:** ~$0.03/PR (середній PR ~500 рядків) або ~$0.15/PR (великий PR з multi-agent). Reasoning додає ~30% до вартості але різко покращує якість знаходження багів.
```

---

## 30.3 Document Processing Pipeline

**Задача:** Автоматичне витягування структурованих даних з PDF-рахунків, контрактів, актів — 500+ документів/день.

**Архітектура:** Upload → Classification → Extraction (structured output) → Validation → DB + Notification.

### Типи документів і схеми

```typescript
import { generateObject, generateText } from 'ai';
import { anthropic } from '@ai-sdk/anthropic';
import { z } from 'zod';

// --- Схеми для різних типів документів ---

const InvoiceSchema = z.object({
  documentType: z.literal('invoice'),
  invoiceNumber: z.string(),
  date: z.string().describe('Дата у форматі YYYY-MM-DD'),
  dueDate: z.string().nullable().describe('Дата оплати, якщо вказана'),
  vendor: z.object({
    name: z.string(),
    taxId: z.string().nullable().describe('ЄДРПОУ або ІПН'),
    address: z.string().nullable(),
  }),
  buyer: z.object({
    name: z.string(),
    taxId: z.string().nullable(),
  }),
  items: z.array(z.object({
    description: z.string(),
    quantity: z.number(),
    unit: z.string().nullable().describe('шт, кг, послуга, тощо'),
    unitPrice: z.number(),
    total: z.number(),
    vatRate: z.number().nullable().describe('Ставка ПДВ у відсотках'),
  })),
  subtotal: z.number(),
  vatAmount: z.number().nullable(),
  totalAmount: z.number(),
  currency: z.string().describe('UAH, USD, EUR'),
  paymentDetails: z.object({
    iban: z.string().nullable(),
    bankName: z.string().nullable(),
  }).nullable(),
});

const ContractSchema = z.object({
  documentType: z.literal('contract'),
  contractNumber: z.string(),
  date: z.string(),
  parties: z.array(z.object({
    role: z.enum(['замовник', 'виконавець', 'постачальник', 'покупець', 'інше']),
    name: z.string(),
    taxId: z.string().nullable(),
  })),
  subject: z.string().describe('Предмет договору — 1-2 речення'),
  totalAmount: z.number().nullable(),
  currency: z.string().nullable(),
  startDate: z.string().nullable(),
  endDate: z.string().nullable(),
  keyTerms: z.array(z.string()).describe('Ключові умови: штрафи, гарантії, SLA (максимум 5)'),
  terminationClause: z.string().nullable().describe('Умови розірвання, якщо вказані'),
});

type DocumentSchema = z.infer<typeof InvoiceSchema> | z.infer<typeof ContractSchema>;
```

### System Prompt для витягування

```typescript
function buildExtractionPrompt(documentType: string): string {
  return `
<role>
Ти — спеціаліст з обробки документів. Твоя задача — точно витягти структуровані дані.
</role>

<rules>
- Витягуй ТІЛЬКИ те що явно написано в документі
- Якщо поле не знайдено — повертай null, НІКОЛИ не вигадуй
- Числа: витягуй як числа, не рядки. "1 500,00 грн" → 1500.00
- Дати: конвертуй у формат YYYY-MM-DD. "15 січня 2026" → "2026-01-15"
- ЄДРПОУ: 8 цифр. ІПН: 10 або 12 цифр. Перевіряй формат
- Якщо документ нечитабельний або пошкоджений — вкажи це в description
- Для сум: перевіряй що subtotal + VAT = totalAmount (якщо дані є)
</rules>

<document_type>${documentType}</document_type>

<quality_checks>
Після витягування перевір:
1. Чи сума items.total збігається з subtotal?
2. Чи формат дат консистентний?
3. Чи ЄДРПОУ/ІПН мають правильну кількість цифр?
Якщо є розбіжності — витягуй як є в документі, не коригуй.
</quality_checks>
`;
}
```

### Крок 1: Класифікація документу

```typescript
const DocumentClassification = z.object({
  documentType: z.enum(['invoice', 'contract', 'act', 'unknown']),
  language: z.enum(['uk', 'en', 'ru', 'other']),
  pageCount: z.number(),
  quality: z.enum(['good', 'readable', 'poor']).describe(
    'good: чіткий текст. readable: є артефакти але читається. poor: важко розібрати'
  ),
  confidence: z.number().min(0).max(1),
});

async function classifyDocument(pdfBase64: string) {
  const { object } = await generateObject({
    model: anthropic('claude-haiku-4-5-20251001'), // Дешева модель для класифікації
    schema: DocumentClassification,
    messages: [{
      role: 'user',
      content: [
        {
          type: 'document',
          source: { type: 'base64', media_type: 'application/pdf', data: pdfBase64 },
        },
        { type: 'text', text: 'Визнач тип документу, мову, якість скану.' },
      ],
    }],
  });

  return object;
}
```

### Крок 2: Витягування даних

```typescript
async function extractDocumentData(
  pdfBase64: string,
  classification: z.infer<typeof DocumentClassification>
): Promise<DocumentSchema> {
  // Обираємо схему і модель залежно від типу і якості
  const config = {
    invoice: { schema: InvoiceSchema, model: 'claude-sonnet-4-5-20250929' as const },
    contract: { schema: ContractSchema, model: 'claude-sonnet-4-5-20250929' as const },
    act: { schema: InvoiceSchema, model: 'claude-haiku-4-5-20251001' as const }, // акти простіші
    unknown: { schema: InvoiceSchema, model: 'claude-sonnet-4-5-20250929' as const },
  }[classification.documentType];

  // Reasoning для поганої якості або складних документів
  const needsReasoning = classification.quality === 'poor'
    || classification.documentType === 'contract';

  const { object } = await generateObject({
    model: anthropic(config.model),
    schema: config.schema,
    system: buildExtractionPrompt(classification.documentType),
    messages: [{
      role: 'user',
      content: [
        {
          type: 'document',
          source: { type: 'base64', media_type: 'application/pdf', data: pdfBase64 },
        },
        { type: 'text', text: 'Витягни всі дані з цього документу відповідно до схеми.' },
      ],
    }],
    ...(needsReasoning && {
      maxTokens: 8000,
      providerOptions: {
        anthropic: {
          thinking: { type: 'enabled', budget_tokens: 2048 },
        },
      },
    }),
  });

  return object;
}
```

### Крок 3: Валідація

```typescript
interface ValidationResult {
  valid: boolean;
  errors: string[];
  warnings: string[];
}

function validateExtraction(data: DocumentSchema): ValidationResult {
  const errors: string[] = [];
  const warnings: string[] = [];

  if (data.documentType === 'invoice') {
    const invoice = data as z.infer<typeof InvoiceSchema>;

    // Перевірка суми items
    const calculatedSubtotal = invoice.items.reduce((sum, item) => sum + item.total, 0);
    if (Math.abs(calculatedSubtotal - invoice.subtotal) > 1) {
      warnings.push(
        `Сума items (${calculatedSubtotal}) ≠ subtotal (${invoice.subtotal})`
      );
    }

    // Перевірка total = subtotal + VAT
    if (invoice.vatAmount !== null) {
      const calculatedTotal = invoice.subtotal + invoice.vatAmount;
      if (Math.abs(calculatedTotal - invoice.totalAmount) > 1) {
        warnings.push(
          `subtotal + VAT (${calculatedTotal}) ≠ totalAmount (${invoice.totalAmount})`
        );
      }
    }

    // Перевірка ЄДРПОУ формату
    if (invoice.vendor.taxId && !/^\d{8}$|^\d{10}$|^\d{12}$/.test(invoice.vendor.taxId)) {
      errors.push(`Невалідний ЄДРПОУ/ІПН: ${invoice.vendor.taxId}`);
    }

    // Перевірка дати
    if (invoice.date && isNaN(Date.parse(invoice.date))) {
      errors.push(`Невалідна дата: ${invoice.date}`);
    }
  }

  return { valid: errors.length === 0, errors, warnings };
}
```

### Повний pipeline

```typescript
interface ProcessingResult {
  id: string;
  classification: z.infer<typeof DocumentClassification>;
  data: DocumentSchema;
  validation: ValidationResult;
  cost: number;
  processingTime: number;
}

async function processDocument(pdfBuffer: Buffer): Promise<ProcessingResult> {
  const startTime = Date.now();
  const pdfBase64 = pdfBuffer.toString('base64');

  // 1. Класифікація (Haiku — дешево)
  const classification = await classifyDocument(pdfBase64);

  if (classification.quality === 'poor' && classification.confidence < 0.5) {
    throw new DocumentQualityError('Документ занадто поганої якості для обробки');
  }

  // 2. Витягування (Sonnet + reasoning для складних)
  const data = await extractDocumentData(pdfBase64, classification);

  // 3. Валідація (код, без LLM — безкоштовно)
  const validation = validateExtraction(data);

  // 4. Збереження
  const id = await db.documents.create({
    data,
    classification,
    validation,
    rawPdf: pdfBase64,
    processedAt: new Date(),
  });

  // 5. Нотифікація якщо є проблеми
  if (!validation.valid) {
    await notifyReviewTeam(id, validation.errors);
  }

  return {
    id,
    classification,
    data,
    validation,
    cost: estimateCost(classification),
    processingTime: Date.now() - startTime,
  };
}
```

### Batch Processing — 500 документів/день

```typescript
import pLimit from 'p-limit';

async function processBatch(pdfBuffers: Buffer[]) {
  const limit = pLimit(5); // Максимум 5 паралельних запитів (rate limit)

  const results = await Promise.allSettled(
    pdfBuffers.map((pdf, index) =>
      limit(async () => {
        try {
          const result = await processDocument(pdf);
          console.log(`[${index + 1}/${pdfBuffers.length}] ✅ ${result.data.documentType}`);
          return result;
        } catch (error) {
          console.error(`[${index + 1}/${pdfBuffers.length}] ❌ ${error.message}`);
          throw error;
        }
      })
    )
  );

  const successful = results.filter(r => r.status === 'fulfilled').length;
  const failed = results.filter(r => r.status === 'rejected').length;

  console.log(`\nBatch complete: ${successful} ✅ / ${failed} ❌`);
  return results;
}
```

**Метрики:**

| Тип документу | Модель | Вартість | Accuracy | Час |
|---------------|--------|---------|---------|-----|
| Рахунок (invoice) | Haiku (class) + Sonnet (extract) | ~$0.04 | 96% | ~4с |
| Контракт | Haiku + Sonnet + reasoning | ~$0.08 | 92% | ~7с |
| Поганий скан | Haiku + Sonnet + reasoning | ~$0.08 | 85% | ~8с |
| 500 документів/день | mix | ~$25/день | - | ~30 хв |

---

## 30.4 Email Automation

**Задача:** Автоматична обробка вхідних email: класифікація → routing → auto-reply або ескалація. 200+ листів/день.

**Архітектура:** IMAP polling → Classification → Intent extraction → Template selection → Draft generation → Human approval (optional) → Send.

### Крок 1: Класифікація + Intent Extraction (один виклик)

Замість двох окремих викликів — один structured output з повною інформацією:

```typescript
import { generateObject, generateText } from 'ai';
import { anthropic } from '@ai-sdk/anthropic';
import { openai } from '@ai-sdk/openai';
import { z } from 'zod';

const EmailAnalysisSchema = z.object({
  category: z.enum([
    'order_status',     // Де моє замовлення?
    'product_inquiry',  // Питання про товар
    'complaint',        // Скарга
    'return_request',   // Повернення
    'partnership',      // B2B пропозиція
    'spam',             // Спам / нерелевантне
    'other',
  ]),
  urgency: z.enum(['low', 'medium', 'high']).describe(
    'low: загальне питання. medium: клієнт чекає. high: скарга, загроза відписки, юрист'
  ),
  sentiment: z.enum(['positive', 'neutral', 'negative', 'angry']),
  language: z.enum(['uk', 'en', 'ru']),
  extractedEntities: z.object({
    orderNumber: z.string().nullable().describe('ORD-XXXXX якщо згадується'),
    productName: z.string().nullable(),
    customerName: z.string().nullable(),
    phoneNumber: z.string().nullable(),
  }),
  keyPoints: z.array(z.string()).describe('Головні питання/вимоги клієнта (1-3 пункти)'),
  autoReplyPossible: z.boolean().describe(
    'true: є шаблонна відповідь. false: потребує людини (складне питання, скарга, B2B)'
  ),
  suggestedAction: z.enum([
    'auto_reply',         // Автоматична відповідь
    'draft_for_review',   // Чернетка для оператора
    'forward_to_sales',   // Переслати в sales
    'forward_to_support', // Переслати в підтримку
    'archive',            // Спам — архівувати
  ]),
});

async function analyzeEmail(email: IncomingEmail) {
  const { object } = await generateObject({
    model: openai('gpt-4o-mini'), // Дешева модель для класифікації
    schema: EmailAnalysisSchema,
    system: `
<role>
Ти — email-аналітик інтернет-магазину TechShop.
</role>

<rules>
- Класифікуй email за категорією, терміновістю і настроєм
- Витягни всі згадані сутності (номери замовлень, товари, телефони)
- Визнач чи можлива автовідповідь
- autoReplyPossible = false для: скарг, повернень, B2B, складних технічних питань
- Якщо клієнт роздратований (angry) — завжди suggestedAction: draft_for_review
</rules>`,
    prompt: `From: ${email.from}
Subject: ${email.subject}
Date: ${email.date}

${email.body}`,
  });

  return object;
}
```

### Крок 2: Генерація відповіді з шаблоном

```typescript
// Шаблони для різних категорій (завантажуються з бази/файлів)
const REPLY_TEMPLATES: Record<string, string> = {
  order_status: `
<template>
Тема: Re: {{subject}}

Вітаю, {{customerName}}!

{{order_info}}

Якщо маєте додаткові питання — пишіть, завжди раді допомогти.

З повагою,
TechShop Support
</template>`,

  product_inquiry: `
<template>
Тема: Re: {{subject}}

Вітаю!

{{product_info}}

{{availability_info}}

Залишились питання? Напишіть або зателефонуйте: 0 800 123 456.

З повагою,
TechShop Support
</template>`,
};

async function generateReply(
  email: IncomingEmail,
  analysis: z.infer<typeof EmailAnalysisSchema>
): Promise<{ subject: string; body: string; confidence: number }> {
  // Збираємо додатковий контекст залежно від категорії
  let additionalContext = '';

  if (analysis.extractedEntities.orderNumber) {
    const order = await orderService.getStatus(analysis.extractedEntities.orderNumber);
    additionalContext += `\n<order_data>\n${JSON.stringify(order, null, 2)}\n</order_data>`;
  }

  if (analysis.extractedEntities.productName) {
    const products = await catalogService.search(analysis.extractedEntities.productName);
    additionalContext += `\n<product_data>\n${JSON.stringify(products.slice(0, 3), null, 2)}\n</product_data>`;
  }

  const template = REPLY_TEMPLATES[analysis.category] || '';

  const { text } = await generateText({
    model: openai('gpt-4o-mini'),
    system: `
<role>
Ти — автор email-відповідей для TechShop. Пишеш ввічливо, конкретно, коротко.
</role>

<rules>
- Пиши ${analysis.language === 'uk' ? 'українською' : analysis.language === 'en' ? 'англійською' : 'мовою клієнта'}
- Відповідай на ВСІ питання/пункти клієнта з <key_points>
- Використовуй дані з <order_data> і <product_data> — не вигадуй
- Максимум 5-7 речень (не рахуючи привітання і підпис)
- Тон: дружній, але професійний
- Формат: готовий email з темою і підписом
</rules>

<key_points>
${analysis.keyPoints.map((p, i) => `${i + 1}. ${p}`).join('\n')}
</key_points>

${template ? `<template_reference>\nВикористай цей шаблон як основу:\n${template}\n</template_reference>` : ''}

${additionalContext}`,
    prompt: `Напиши відповідь на цей email:

From: ${email.from}
Subject: ${email.subject}

${email.body}`,
  });

  // Парсимо subject і body з відповіді
  const subjectMatch = text.match(/Тема:\s*(.+)/i) || text.match(/Subject:\s*(.+)/i);
  const subject = subjectMatch ? subjectMatch[1].trim() : `Re: ${email.subject}`;
  const body = text.replace(/^Тема:.+\n?/im, '').replace(/^Subject:.+\n?/im, '').trim();

  return { subject, body, confidence: analysis.autoReplyPossible ? 0.85 : 0.5 };
}
```

### Крок 3: Повний pipeline

```typescript
interface EmailAction {
  action: 'sent' | 'draft_created' | 'forwarded' | 'archived';
  emailId: string;
  analysis: z.infer<typeof EmailAnalysisSchema>;
  reply?: { subject: string; body: string };
  forwardedTo?: string;
}

async function processIncomingEmail(email: IncomingEmail): Promise<EmailAction> {
  // 1. Аналіз
  const analysis = await analyzeEmail(email);

  // 2. Routing
  switch (analysis.suggestedAction) {
    case 'archive':
      await emailService.archive(email.id);
      return { action: 'archived', emailId: email.id, analysis };

    case 'forward_to_sales':
      await emailService.forward(email.id, 'sales@techshop.ua', {
        note: `[AI] Category: ${analysis.category} | ${analysis.keyPoints.join('; ')}`,
      });
      return { action: 'forwarded', emailId: email.id, analysis, forwardedTo: 'sales' };

    case 'forward_to_support':
      await emailService.forward(email.id, 'support@techshop.ua', {
        note: `[AI] Priority: ${analysis.urgency} | ${analysis.keyPoints.join('; ')}`,
      });
      return { action: 'forwarded', emailId: email.id, analysis, forwardedTo: 'support' };

    case 'auto_reply': {
      const reply = await generateReply(email, analysis);

      if (reply.confidence > 0.8 && analysis.sentiment !== 'angry') {
        // Високий confidence — відправляємо автоматично
        await emailService.send({
          to: email.from,
          subject: reply.subject,
          body: reply.body,
          inReplyTo: email.id,
        });
        return { action: 'sent', emailId: email.id, analysis, reply };
      }

      // Низький confidence — створюємо чернетку для оператора
      await emailService.createDraft({
        to: email.from,
        subject: reply.subject,
        body: reply.body,
        inReplyTo: email.id,
        metadata: { aiAnalysis: analysis, confidence: reply.confidence },
      });
      return { action: 'draft_created', emailId: email.id, analysis, reply };
    }

    case 'draft_for_review': {
      const reply = await generateReply(email, analysis);
      await emailService.createDraft({
        to: email.from,
        subject: reply.subject,
        body: reply.body,
        inReplyTo: email.id,
        metadata: { aiAnalysis: analysis, confidence: reply.confidence },
      });
      // Нотифікація оператору для urgent
      if (analysis.urgency === 'high') {
        await notifyOperator(email.id, analysis);
      }
      return { action: 'draft_created', emailId: email.id, analysis, reply };
    }
  }
}
```

### Batch Processing з метриками

```typescript
async function processEmailBatch() {
  const unprocessed = await emailService.getUnprocessed({ limit: 50 });

  const results = await Promise.allSettled(
    unprocessed.map(email => processIncomingEmail(email))
  );

  // Збираємо метрики
  const metrics = {
    total: results.length,
    sent: results.filter(r => r.status === 'fulfilled' && r.value.action === 'sent').length,
    drafted: results.filter(r => r.status === 'fulfilled' && r.value.action === 'draft_created').length,
    forwarded: results.filter(r => r.status === 'fulfilled' && r.value.action === 'forwarded').length,
    archived: results.filter(r => r.status === 'fulfilled' && r.value.action === 'archived').length,
    failed: results.filter(r => r.status === 'rejected').length,
  };

  console.log(`📧 Batch: ${metrics.total} emails processed`);
  console.log(`   ✅ Auto-sent: ${metrics.sent}`);
  console.log(`   📝 Drafts: ${metrics.drafted}`);
  console.log(`   ↪️  Forwarded: ${metrics.forwarded}`);
  console.log(`   🗑️  Archived: ${metrics.archived}`);
  console.log(`   ❌ Failed: ${metrics.failed}`);

  return metrics;
}
```

**Метрики:**

| Крок | Модель | Вартість/email | Час |
|------|--------|---------------|-----|
| Класифікація | GPT-4o-mini | ~$0.0003 | <1с |
| Генерація відповіді | GPT-4o-mini | ~$0.0008 | ~2с |
| Разом | - | ~$0.001 | ~3с |
| 200 emails/день | - | ~$0.20/день | ~10 хв |

При 200 листах/день: 40-50% auto-sent, 30% drafts для оператора, 10% forwarded, 10% archived. Оператор замість 4 годин/день витрачає 1 годину на перегляд чернеток.
```

---

## Перевір себе

1. Чому XML-теги (`<rules>`, `<knowledge_base>`) працюють краще за plain text у system prompt? Як це допомагає з prompt caching?
2. У Code Review Assistant — навіщо multi-agent підхід для великих PR? Яку проблему вирішує розділення на file-level sub-agents?
3. Document Processing Pipeline: чому класифікація на Haiku а витягування на Sonnet? Порахуйте різницю у вартості при 500 документів/день
4. Email Automation: яка різниця між `auto_reply` і `draft_for_review`? Коли що використовується?
5. Реалізуйте один з кейсів для свого проекту. Порахуйте місячну вартість при очікуваному навантаженні

---

**Назад:** [← Модуль 29 — Edge AI](29-edge-local.md) | **Далі:** [Модуль 31 — Наскрізний проект →](31-project.md)
