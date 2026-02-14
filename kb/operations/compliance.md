# Модуль 36: Compliance та аудит — GDPR, logging, data retention

## 🎯 Що ви отримаєте з цього модуля

Після проходження ви будете:
- Логувати LLM-взаємодії для аудиту та дебагу
- Реалізовувати data retention policies (автоматичне видалення)
- Відповідати вимогам GDPR при використанні LLM API
- Готувати документацію для SOC 2 та ISO 27001 аудитів

**Які задачі це дозволяє вирішувати:** Пройти security review у enterprise-клієнта. Відповісти на питання "де зберігаються дані, хто має доступ, як довго". Уникнути штрафів за порушення GDPR.

---

## 36.1 Audit Logging: Що логувати

Кожен LLM-запит — це потенційний audit event:

```typescript
interface AuditLog {
  id: string;
  timestamp: Date;
  userId: string;
  action: 'llm_request' | 'tool_call' | 'data_access';
  model: string;
  provider: string;

  // Input (masked)
  inputTokens: number;
  inputHash: string;        // SHA-256 hash, НЕ повний текст
  hasPII: boolean;

  // Output
  outputTokens: number;
  finishReason: string;

  // Cost
  estimatedCost: number;
  currency: string;

  // Context
  feature: string;           // Яка функція продукту
  sessionId: string;
  ipAddress?: string;
}

class AuditLogger {
  async log(entry: AuditLog) {
    // В production — append-only table або event stream
    await db.query(`
      INSERT INTO audit_logs (id, timestamp, user_id, action, model, provider,
        input_tokens, input_hash, has_pii, output_tokens, finish_reason,
        estimated_cost, feature, session_id)
      VALUES ($1, $2, $3, $4, $5, $6, $7, $8, $9, $10, $11, $12, $13, $14)
    `, [entry.id, entry.timestamp, entry.userId, entry.action, entry.model,
        entry.provider, entry.inputTokens, entry.inputHash, entry.hasPII,
        entry.outputTokens, entry.finishReason, entry.estimatedCost,
        entry.feature, entry.sessionId]);
  }
}
```

### Що НЕ логувати

- Повний текст промпту (може містити PII)
- Повну відповідь моделі (може містити згенеровані PII)
- API ключі (очевидно)

Замість повного тексту зберігайте hash + metadata.

---

## 36.2 GDPR та LLM

### Ключові вимоги

**Правова основа обробки:** Вам потрібна правова підстава для відправки даних користувача в LLM API (згода, legitimate interest, або контракт).

**Право на видалення (Article 17):** Користувач може попросити видалити свої дані — включаючи логи LLM-запитів.

**Data Processing Agreement (DPA):** Потрібен DPA з кожним LLM-провайдером.

```typescript
// Видалення даних користувача (GDPR Right to Erasure)
async function deleteUserData(userId: string) {
  // 1. Видаляємо audit logs
  await db.query('DELETE FROM audit_logs WHERE user_id = $1', [userId]);

  // 2. Видаляємо chat history
  await db.query('DELETE FROM conversations WHERE user_id = $1', [userId]);

  // 3. Видаляємо memory/facts
  await db.query('DELETE FROM memories WHERE user_id = $1', [userId]);

  // 4. Видаляємо vector embeddings
  await db.query('DELETE FROM user_embeddings WHERE user_id = $1', [userId]);

  // 5. Логуємо сам факт видалення (без PII)
  await auditLogger.log({
    action: 'data_deletion',
    userId: 'DELETED',
    metadata: { deletedAt: new Date(), requestId: generateId() },
  });
}
```

### PII Masking перед відправкою в LLM

```typescript
function maskPIIForLLM(text: string): { masked: string; mappings: Map<string, string> } {
  const mappings = new Map<string, string>();
  let masked = text;
  let counter = 0;

  // Email
  masked = masked.replace(/[\w.-]+@[\w.-]+\.\w+/g, (match) => {
    const placeholder = `[EMAIL_${++counter}]`;
    mappings.set(placeholder, match);
    return placeholder;
  });

  // Phone
  masked = masked.replace(/\+?38\s?0\d{2}\s?\d{3}\s?\d{2}\s?\d{2}/g, (match) => {
    const placeholder = `[PHONE_${++counter}]`;
    mappings.set(placeholder, match);
    return placeholder;
  });

  return { masked, mappings };
}

function unmaskPII(text: string, mappings: Map<string, string>): string {
  let unmasked = text;
  for (const [placeholder, original] of mappings) {
    unmasked = unmasked.replaceAll(placeholder, original);
  }
  return unmasked;
}

// Використання
const { masked, mappings } = maskPIIForLLM(userMessage);
const { text } = await generateText({ model, prompt: masked }); // LLM не бачить PII
const finalResponse = unmaskPII(text, mappings); // Повертаємо PII назад
```

---

## 36.3 Data Retention Policies

```typescript
// Автоматичне видалення старих даних
class RetentionManager {
  private policies: Record<string, number> = {
    'audit_logs': 365,        // 1 рік
    'conversations': 90,      // 3 місяці
    'error_logs': 30,         // 1 місяць
    'analytics_events': 730,  // 2 роки
  };

  async enforceRetention() {
    for (const [table, retentionDays] of Object.entries(this.policies)) {
      const cutoff = new Date();
      cutoff.setDate(cutoff.getDate() - retentionDays);

      const result = await db.query(
        `DELETE FROM ${table} WHERE created_at < $1`,
        [cutoff]
      );

      console.log(`[Retention] ${table}: deleted ${result.rowCount} rows older than ${retentionDays} days`);
    }
  }
}

// Запуск щоночі
cron.schedule('0 3 * * *', () => retentionManager.enforceRetention());
```

---

## 36.4 SOC 2 Checklist для AI-систем

**Access Control:** Хто має доступ до LLM API keys, audit логів, training data. Role-based access, MFA.

**Data Classification:** Які дані обробляються LLM (public, internal, confidential, restricted). Чи є PII/PHI у промптах.

**Vendor Management:** DPA з OpenAI/Anthropic/Google. Де фізично обробляються дані (EU/US). Чи використовуються дані для тренування (ні через API).

**Incident Response:** Що робити при витоку промптів, при prompt injection, при аномальній активності.

**Monitoring:** Audit logs з retention policy, алерти на аномалії (раптовий spike запитів), регулярні access reviews.

---

## Перевір себе

1. Що логувати а що НЕ логувати при LLM-запитах?
2. Як реалізувати GDPR Right to Erasure для AI-продукту?
3. Навіщо маскувати PII перед відправкою в LLM?
4. Реалізуйте data retention policy з автоматичним видаленням
5. Які документи потрібні для SOC 2 аудиту AI-системи?

---

**Назад:** [← Модуль 35 — Data pipelines](35-rag-pipelines.md) | **Далі:** [Модуль 37 — Testing patterns →](37-testing-patterns.md)
