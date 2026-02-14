# KB: AWS Bedrock для TypeScript — Повний довідник

> 📚 Фундамент: [Course — Enterprise Providers](../../course/enterprise-providers.md)

## Коли це потрібно

- Enterprise-клієнт вимагає розгортання AI в AWS
- Потрібен managed RAG (Knowledge Bases) замість self-hosted
- Будуєте AI-агента з AgentCore/Strands
- Налаштовуєте Guardrails для compliance
- Потрібен VPC Endpoint для приватного доступу до моделей

---

## Quick Reference

```bash
npm install @ai-sdk/amazon-bedrock
```

```typescript
import { bedrock } from '@ai-sdk/amazon-bedrock';
import { generateText } from 'ai';

// Claude через Bedrock
const { text } = await generateText({
  model: bedrock('us.anthropic.claude-sonnet-4-5-20250929-v1:0'),
  prompt: 'Your prompt',
});
```

```env
# Auth варіант 1: API Key
AWS_BEARER_TOKEN_BEDROCK=your-api-key

# Auth варіант 2: SigV4 (default)
AWS_ACCESS_KEY_ID=AKIA...
AWS_SECRET_ACCESS_KEY=wJalr...
AWS_REGION=us-east-1
```

---

## Доступні моделі

### Anthropic Claude

| Model ID (Bedrock) | Модель |
|-------------------|--------|
| `us.anthropic.claude-opus-4-6-20250918-v1:0` | Claude Opus 4.6 |
| `us.anthropic.claude-sonnet-4-5-20250929-v1:0` | Claude Sonnet 4.5 |
| `us.anthropic.claude-haiku-4-5-20251001-v1:0` | Claude Haiku 4.5 |

### Amazon Nova

| Model ID | Модель | Примітка |
|----------|--------|---------|
| `amazon.nova-pro-v1:0` | Nova Pro | Потужна, reasoning |
| `amazon.nova-lite-v1:0` | Nova Lite | Швидка, дешева |
| `amazon.nova-micro-v1:0` | Nova Micro | Тільки текст, найдешевша |

### Meta Llama

| Model ID | Модель |
|----------|--------|
| `meta.llama3-1-70b-instruct-v1:0` | Llama 3.1 70B |
| `meta.llama3-1-8b-instruct-v1:0` | Llama 3.1 8B |

### Embedding моделі

| Model ID | Розмірності |
|----------|------------|
| `amazon.titan-embed-text-v2:0` | 256 / 512 / 1024 |
| `amazon.nova-embed-v1:0` | 256 / 384 / 1024 / 3072 |

```typescript
// Embeddings через Bedrock
import { bedrock } from '@ai-sdk/amazon-bedrock';
import { embed } from 'ai';

const { embedding } = await embed({
  model: bedrock.embedding('amazon.titan-embed-text-v2:0', {
    dimensions: 512,
  }),
  value: 'Your text to embed',
});
```

*Список моделей постійно оновлюється. Повний перелік: [AWS Bedrock Model Access](https://docs.aws.amazon.com/bedrock/latest/userguide/model-access.html).*

---

## Автентифікація: деталі

### API Key (Bearer Token)

Найпростіший спосіб. Підходить для розробки та простих deployments.

```typescript
import { bedrock } from '@ai-sdk/amazon-bedrock';

// Варіант 1: через env var (рекомендовано)
// AWS_BEARER_TOKEN_BEDROCK=your-key
const model = bedrock('us.anthropic.claude-sonnet-4-5-20250929-v1:0');

// Варіант 2: явно
const bedrockWithKey = bedrock.withSettings({
  apiKey: 'your-api-key',
  region: 'us-east-1',
});
```

### AWS SigV4 (Standard)

Стандартна AWS автентифікація. Підходить для production, EC2, Lambda, ECS.

```typescript
// Автоматично використовується якщо apiKey не вказано
// Потрібні env vars:
// AWS_ACCESS_KEY_ID, AWS_SECRET_ACCESS_KEY, AWS_REGION

// На EC2/Lambda/ECS — автоматично через Instance Profile / Task Role
// Ніяких env vars не потрібно — IAM role приєднується до ресурсу
```

### IAM Policy для Bedrock

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "bedrock:InvokeModel",
        "bedrock:InvokeModelWithResponseStream"
      ],
      "Resource": [
        "arn:aws:bedrock:us-east-1::foundation-model/anthropic.claude-*",
        "arn:aws:bedrock:us-east-1::foundation-model/amazon.nova-*"
      ]
    }
  ]
}
```

### VPC Endpoint

Для приватного доступу без виходу в інтернет:

```
VPC → VPC Endpoint (bedrock-runtime) → Bedrock API
↑ Дані ніколи не покидають AWS мережу
```

---

## Extended Thinking через Bedrock

Claude extended thinking працює через Bedrock з тими ж параметрами:

```typescript
import { bedrock } from '@ai-sdk/amazon-bedrock';
import { generateText } from 'ai';

// Adaptive thinking (Opus 4.6)
const { text, reasoning } = await generateText({
  model: bedrock('us.anthropic.claude-opus-4-6-20250918-v1:0'),
  prompt: 'Complex analysis...',
  providerOptions: {
    anthropic: { thinking: { type: 'adaptive' } },
  },
});

// Explicit budget (Sonnet 4.5)
const { text: result } = await generateText({
  model: bedrock('us.anthropic.claude-sonnet-4-5-20250929-v1:0'),
  prompt: 'Debug this code...',
  maxTokens: 16000,
  providerOptions: {
    anthropic: {
      thinking: { type: 'enabled', budget_tokens: 8000 },
    },
  },
});
```

**Interleaved thinking:** Для Claude Opus 4.6 через Bedrock — автоматично з adaptive. Для Claude 4 моделей — потрібен beta header `interleaved-thinking-2025-05-14`.

**Thinking block clearing** (beta): Header `context-management-2025-06-27`. Підтримується на Claude Sonnet 4/4.5, Haiku 4.5, Opus 4/4.1/4.5.

**Streaming обов'язковий** коли `max_tokens > 21,333`.

---

## Bedrock Knowledge Bases — Managed RAG

Замість самостійно будувати vector store + indexing pipeline:

```
Джерела даних:
  S3 bucket → ┐
  Confluence → ├→ Knowledge Base → Vector Store (OpenSearch / Aurora / Pinecone)
  SharePoint → ┘                        ↓
                               Automatic chunking + embedding
                                        ↓
                               Запит → Retrieval → Відповідь з citation
```

**Переваги:**
- Автоматична індексація при зміні джерел
- Managed vector store (не потрібен окремий Pinecone/Qdrant)
- Chunking strategies з коробки
- Citation (посилання на джерело) автоматично

**Коли використовувати:**
- Enterprise клієнт на AWS який хоче managed RAG
- Джерела: S3, Confluence, SharePoint, Web Crawler
- Не хочете підтримувати власний vector DB

**Коли НЕ використовувати:**
- Потрібен кастомний chunking/embedding pipeline
- Дані не в AWS
- Потрібна максимальна гнучкість retrieval

---

## Bedrock Guardrails

Managed content filtering і safety policies:

```
Запит → Guardrails (input filter) → Модель → Guardrails (output filter) → Відповідь
```

**Що фільтрує:**
- Content filters: hate, insults, sexual, violence
- Denied topics: теми заборонені для вашого use case
- Word filters: конкретні слова/фрази
- Sensitive information: PII detection (email, phone, SSN)
- Contextual grounding: перевірка що відповідь базується на наданому контексті

**Коли потрібно:**
- Enterprise compliance (фінанси, медицина, держсектор)
- Потрібно гарантувати що AI не обговорює певні теми
- PII protection обов'язкова

---

## Bedrock AgentCore SDK (TypeScript)

AWS випустив TypeScript SDK для побудови та деплою агентів:

### Компоненти AgentCore

| Компонент | Що робить |
|-----------|----------|
| **Runtime** | Managed server для агентів: session management, request parsing, streaming |
| **Code Interpreter** | Sandbox для виконання Python/JS/TS коду |
| **Browser** | Cloud-based веб-автоматизація |
| **Identity** | OAuth tokens, API keys management |

### Використання з AI SDK

AgentCore Tools інтегруються з AI SDK:

```typescript
// Browser tool через AgentCore + AI SDK
import { generateText } from 'ai';
import { bedrock } from '@ai-sdk/amazon-bedrock';
import { browserTool } from 'bedrock-agentcore/experimental/browser/ai-sdk';

const browser = browserTool({ region: 'us-east-1' });

const { text } = await generateText({
  model: bedrock('us.anthropic.claude-sonnet-4-5-20250929-v1:0'),
  prompt: 'Go to https://example.com and extract the main heading',
  tools: { browser },
  maxSteps: 5,
});
```

```typescript
// Code Interpreter tool
import { codeInterpreterTool } from 'bedrock-agentcore/experimental/code-interpreter/ai-sdk';

const codeInterpreter = codeInterpreterTool({ region: 'us-east-1' });

const { text } = await generateText({
  model: bedrock('us.anthropic.claude-sonnet-4-5-20250929-v1:0'),
  prompt: 'Calculate the compound interest on $10,000 at 5% for 10 years',
  tools: { codeInterpreter },
  maxSteps: 3,
});
```

### Використання зі Strands Agents

Strands — AWS-native agent framework:

```typescript
import { Agent, BedrockModel } from '@strands-agents/sdk';
import { z } from 'zod';

const agent = new Agent({
  model: new BedrockModel({
    modelId: 'us.anthropic.claude-sonnet-4-5-20250929-v1:0',
  }),
  tools: [{
    name: 'get_weather',
    description: 'Get current weather for a city',
    schema: z.object({ city: z.string() }),
    handler: async ({ city }) => {
      return await fetchWeather(city);
    },
  }],
});

const response = await agent.run('What is the weather in Kyiv?');
```

---

## Pricing: Bedrock vs Direct API

Bedrock зазвичай дорожчий за direct API, але включає enterprise features:

| Модель | Direct API ($/M input) | Bedrock ($/M input) | Overhead |
|--------|----------------------|--------------------|---------| 
| Claude Sonnet 4.5 | $3 | ~$3 | ~0% (ціни вирівнялись) |
| Claude Opus 4.6 | $15 | ~$15 | ~0% |
| Claude Haiku 4.5 | $0.80 | ~$0.80 | ~0% |

*Примітка: Bedrock ціни для on-demand. Provisioned Throughput і Batch Inference мають інші тарифи. Перевіряйте [актуальні ціни](https://aws.amazon.com/bedrock/pricing/).*

**Що ви отримуєте за ці гроші:**
- VPC Endpoints (приватний доступ)
- IAM-based access control
- CloudTrail audit logging
- Guardrails (content filtering)
- Knowledge Bases (managed RAG)
- Єдиний AWS білінг
- Enterprise SLA

---

## Production Setup Checklist

### Мінімальний production setup

- [ ] IAM Role з мінімальними правами (InvokeModel тільки для потрібних моделей)
- [ ] Environment variables для credentials (не хардкодити)
- [ ] VPC Endpoint якщо дані чутливі
- [ ] CloudTrail logging увімкнений
- [ ] Request access для потрібних моделей в Console

### Enterprise setup

- [ ] Все з мінімального +
- [ ] Guardrails налаштовані під use case
- [ ] Knowledge Bases для RAG (якщо потрібно)
- [ ] Provisioned Throughput якщо потрібна гарантована latency
- [ ] Cross-region inference для high availability
- [ ] Cost allocation tags для білінгу по проектах
- [ ] AWS Organizations SCP для обмеження моделей

---

## Пов'язане

- **Курс:** [AI SDK](../../course/03-ai-sdk.md) — єдиний інтерфейс до всіх провайдерів
- **Курс:** [Enterprise Providers](../../course/enterprise-providers.md) — порівняння Bedrock/Azure/Vertex
- **Курс:** [Production](../../course/13-production.md) — caching, routing, observability
- **Курс:** [Reasoning Models](../../course/reasoning-models.md) — extended thinking через Bedrock
- **KB:** [RAG Pipelines](../rag/rag-pipelines.md) — self-hosted альтернатива Knowledge Bases
- **KB:** [Compliance](../operations/compliance.md) — GDPR, audit logging

## Джерела

- [AI SDK — Amazon Bedrock Provider](https://ai-sdk.dev/providers/ai-sdk-providers/amazon-bedrock)
- [AWS — Bedrock Documentation](https://docs.aws.amazon.com/bedrock/)
- [AWS — Bedrock AgentCore SDK TypeScript](https://github.com/aws/bedrock-agentcore-sdk-typescript)
- [AWS — Strands Agents SDK TypeScript](https://github.com/strands-agents/sdk-typescript)
- [AI SDK — Bedrock AgentCore Tools](https://ai-sdk.dev/tools-registry/bedrock-agentcore)
- [AWS — Bedrock Pricing](https://aws.amazon.com/bedrock/pricing/)
