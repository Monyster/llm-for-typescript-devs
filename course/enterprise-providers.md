# Enterprise Providers — Bedrock, Azure, Vertex

> ⏱ ~2 години | 🟠 Middle-Senior | Потрібен: [Модуль 03 — AI SDK](03-ai-sdk.md)
>
> 📖 Deep dive: [KB: AWS Bedrock для TypeScript](../kb/enterprise/aws-bedrock.md)

## 💰 Бізнес-цінність

**Проблема клієнта:** "Ми не можемо використовувати OpenAI API напряму — наш compliance вимагає щоб дані не покидали AWS/Azure."  
**Рішення з цього модуля:** Ті ж самі моделі (Claude, GPT, Llama) через enterprise cloud з VPC, IAM, audit logging.  
**Результат:** Enterprise-клієнти платять на 15–30% більше за managed сервіс, але отримують compliance, SLA, і єдиний білінг через cloud provider.  
**Як продати:** Знання Bedrock/Azure = enterprise-проекти = контракти від $50K+. "Розгортання AI-рішення в вашій AWS інфраструктурі" — аргумент який відкриває двері до корпоратів.

---

## Навіщо enterprise providers

Коли ви викликаєте `openai('gpt-5')` або `anthropic('claude-sonnet-4-5')` — ваші дані йдуть на сервери OpenAI/Anthropic. Для стартапу це нормально. Для банку, страхової чи держоргану — ні.

Enterprise providers — це ті ж самі моделі, але розгорнуті в хмарі клієнта:

```
Direct API:
  Ваш код → api.openai.com → GPT-5 → відповідь
  ✅ Просто      ❌ Дані на серверах OpenAI

AWS Bedrock:
  Ваш код → bedrock.us-east-1.amazonaws.com → GPT-5 / Claude → відповідь
  ✅ Дані в AWS клієнта  ✅ VPC  ✅ IAM  ✅ CloudTrail  ✅ Єдиний білінг AWS

Azure OpenAI:
  Ваш код → your-resource.openai.azure.com → GPT-5 → відповідь
  ✅ Дані в Azure клієнта  ✅ Entra ID  ✅ VNET  ✅ Compliance
```

### Коли обирати enterprise provider

| Вимога | Direct API | Enterprise Provider |
|--------|-----------|-------------------|
| Швидкий старт, прототип | ✅ | Зайве |
| Стартап, MVP | ✅ | Зайве |
| Enterprise клієнт з compliance | ❌ | ✅ Обов'язково |
| Дані не покидають регіон (GDPR) | ❌ | ✅ |
| Єдиний білінг через cloud | ❌ | ✅ |
| VPC / Private Link | ❌ | ✅ |
| IAM / RBAC для доступу до AI | ❌ | ✅ |
| Audit logging | Обмежено | ✅ CloudTrail / Azure Monitor |
| SLA гарантований | Залежить | ✅ Enterprise SLA |

---

## AWS Bedrock

Amazon Bedrock — найпопулярніший enterprise AI gateway. Доступ до Claude, Llama, Mistral, Amazon Nova та інших моделей через AWS API.

### Встановлення

```bash
npm install @ai-sdk/amazon-bedrock
```

### Автентифікація

Два способи:

```typescript
import { bedrock } from '@ai-sdk/amazon-bedrock';

// Спосіб 1: API Key (простіший)
// Встановіть AWS_BEARER_TOKEN_BEDROCK env var
// або передайте напряму:
const bedrockWithKey = bedrock.withSettings({
  apiKey: process.env.AWS_BEARER_TOKEN_BEDROCK,
  region: 'us-east-1',
});

// Спосіб 2: AWS SigV4 (стандартний для production)
// Використовує AWS_ACCESS_KEY_ID + AWS_SECRET_ACCESS_KEY env vars
// Якщо apiKey не вказано — автоматично SigV4
```

```env
# .env для SigV4
AWS_ACCESS_KEY_ID=AKIA...
AWS_SECRET_ACCESS_KEY=wJalr...
AWS_REGION=us-east-1
```

### Базове використання

```typescript
import { generateText } from 'ai';
import { bedrock } from '@ai-sdk/amazon-bedrock';

// Claude через Bedrock
const { text } = await generateText({
  model: bedrock('us.anthropic.claude-sonnet-4-5-20250929-v1:0'),
  prompt: 'Проаналізуй цей документ...',
});

// Amazon Nova (власна модель AWS)
const { text: novaText } = await generateText({
  model: bedrock('amazon.nova-pro-v1:0'),
  prompt: 'Summarize this report...',
});

// Llama через Bedrock
const { text: llamaText } = await generateText({
  model: bedrock('meta.llama3-1-70b-instruct-v1:0'),
  prompt: 'Explain this code...',
});
```

### Extended Thinking через Bedrock

```typescript
const { text, reasoning } = await generateText({
  model: bedrock('us.anthropic.claude-sonnet-4-5-20250929-v1:0'),
  prompt: 'Знайди помилку в цьому коді...',
  providerOptions: {
    anthropic: {
      thinking: { type: 'enabled', budget_tokens: 4096 },
    },
  },
  maxTokens: 16000,
});
```

### Bedrock AgentCore + Strands

AWS нещодавно випустив TypeScript SDK для побудови агентів:

```typescript
import { BedrockAgentCoreApp } from 'bedrock-agentcore/runtime';
import { Agent, BedrockModel } from '@strands-agents/sdk';
import { z } from 'zod';

const agent = new Agent({
  model: new BedrockModel({
    modelId: 'global.amazon.nova-2-lite-v1:0',
  }),
});

// AgentCore Runtime — managed server для агентів
const app = new BedrockAgentCoreApp({
  invocationHandler: {
    requestSchema: z.object({ prompt: z.string() }),
    process: async function* (request) {
      for await (const event of agent.stream(request.prompt)) {
        if (event.type === 'modelContentBlockDeltaEvent' 
            && event.delta?.type === 'textDelta') {
          yield { event: 'message', data: { text: event.delta.text } };
        }
      }
    },
  },
});

app.run();
```

AgentCore надає managed інфраструктуру: Browser tool (веб-автоматизація), Code Interpreter (sandbox для JS/TS/Python), Identity (OAuth, API keys), та Runtime (session-isolated compute).

### Bedrock Knowledge Bases — Managed RAG

Замість самостійно будувати RAG pipeline, Bedrock пропонує managed сервіс:

```
Ваші документи → S3 → Bedrock Knowledge Base → Автоматична індексація
                                                  ↓
Запит → Bedrock Agent → Knowledge Base retrieval → Відповідь з джерелами
```

Підтримує: S3, Confluence, SharePoint, Salesforce, Web Crawler як джерела даних.

---

## Azure OpenAI

Microsoft Azure OpenAI Service — доступ до моделей OpenAI (GPT-5, o3, DALL-E) через Azure інфраструктуру.

### Встановлення

```bash
npm install @ai-sdk/azure
```

### Налаштування

```env
# .env
AZURE_RESOURCE_NAME=your-resource-name
AZURE_API_KEY=your-api-key
```

```typescript
import { azure } from '@ai-sdk/azure';
import { generateText } from 'ai';

// Azure використовує deployment names замість model IDs
const { text } = await generateText({
  model: azure('your-gpt5-deployment'),
  prompt: 'Проаналізуй...',
});
```

### Кастомна конфігурація

```typescript
import { createAzure } from '@ai-sdk/azure';

const azure = createAzure({
  resourceName: 'my-company-openai',
  apiKey: process.env.AZURE_API_KEY,
  // або baseURL для proxy/custom endpoint:
  // baseURL: 'https://my-proxy.company.com/openai/v1',
});

const { text } = await generateText({
  model: azure('gpt5-east-us'),
  prompt: 'Summarize...',
});
```

### Ключова різниця від Direct OpenAI

- **Deployment-based:** Ви створюєте "deployment" для кожної моделі в Azure Portal. Використовуєте deployment name замість model ID
- **Регіони:** Моделі доступні не у всіх регіонах Azure. Перевіряйте availability
- **Квоти:** Azure має rate limits на рівні deployment, які ви контролюєте
- **Responses API:** Azure використовує Responses API за замовчуванням в AI SDK 6. Provider options використовують ключ `azure` замість `openai`

---

## Google Vertex AI

Google Cloud Vertex AI — доступ до Gemini моделей через Google Cloud інфраструктуру.

### Встановлення

```bash
npm install @ai-sdk/google-vertex
```

### Налаштування

```env
# .env
GOOGLE_VERTEX_PROJECT=your-gcp-project-id
GOOGLE_VERTEX_LOCATION=us-central1
```

```typescript
import { vertex } from '@ai-sdk/google-vertex';
import { generateText } from 'ai';

const { text } = await generateText({
  model: vertex('gemini-2.5-pro'),
  prompt: 'Analyze this dataset...',
});
```

Vertex AI також надає доступ до Claude моделей через Model Garden.

---

## Сила AI SDK: одна зміна — інший провайдер

Ось навіщо ми вчили AI SDK (Модуль 03). Бізнес-логіка не змінюється — змінюється один рядок:

```typescript
import { generateText } from 'ai';

// Розробка — пряме API (дешево, швидко)
import { openai } from '@ai-sdk/openai';
const model = openai('gpt-4o-mini');

// Staging — Azure (тестуємо enterprise setup)
import { azure } from '@ai-sdk/azure';
const model = azure('gpt4o-mini-staging');

// Production — Bedrock (enterprise клієнт на AWS)
import { bedrock } from '@ai-sdk/amazon-bedrock';
const model = bedrock('us.anthropic.claude-sonnet-4-5-20250929-v1:0');

// Код залишається ІДЕНТИЧНИМ:
const { text } = await generateText({
  model,
  prompt: userQuery,
  tools: myTools,
  maxSteps: 5,
});
```

### Provider Fallback

```typescript
// Якщо основний провайдер недоступний — fallback
async function generateWithFallback(prompt: string) {
  try {
    return await generateText({
      model: bedrock('us.anthropic.claude-sonnet-4-5-20250929-v1:0'),
      prompt,
    });
  } catch (error) {
    console.warn('Bedrock unavailable, falling back to direct API');
    return await generateText({
      model: anthropic('claude-sonnet-4-5-20250929'),
      prompt,
    });
  }
}
```

---

## Порівняння: коли який обрати

| Критерій | AWS Bedrock | Azure OpenAI | Google Vertex |
|----------|-------------|-------------|---------------|
| **Клієнт вже на** | AWS | Azure | GCP |
| **Найкращі моделі** | Claude, Llama, Nova | GPT-5, o3 | Gemini 2.5 |
| **Managed RAG** | Knowledge Bases | AI Search | Vertex AI Search |
| **Agent framework** | AgentCore + Strands SDK | Foundry Agents | Vertex AI Agents |
| **Pricing overhead** | ~15-20% vs direct | ~10-20% vs direct | ~10-15% vs direct |
| **TypeScript SDK** | @ai-sdk/amazon-bedrock | @ai-sdk/azure | @ai-sdk/google-vertex |
| **Auth** | SigV4 / API Key | API Key / Entra ID | Google ADC |

**Просте правило:** Дивись де вже живе інфраструктура клієнта. AWS → Bedrock, Azure → Azure OpenAI, GCP → Vertex.

---

## Чеклист: Enterprise AI проект

1. **Де інфраструктура клієнта?** AWS / Azure / GCP → відповідний провайдер
2. **Які моделі потрібні?** Claude → Bedrock або Vertex. GPT → Azure. Gemini → Vertex
3. **Compliance вимоги?** GDPR, SOC 2, HIPAA → Enterprise provider обов'язково
4. **Auth модель?** IAM roles (AWS), Entra ID (Azure), Service Accounts (GCP)
5. **Networking?** VPC Endpoints (AWS), Private Link (Azure), VPC SC (GCP)
6. **Billing?** Клієнт хоче єдиний рахунок через cloud provider

---

## Що далі

Тепер ви знаєте як розгортати AI через enterprise cloud. Далі — як монетизувати всі ці знання: [Модуль 18: Бізнес](18-business.md).

Для глибшого занурення в AWS Bedrock (Knowledge Bases, Guardrails, AgentCore): [KB: AWS Bedrock](../kb/enterprise/aws-bedrock.md).

---

## Джерела

- [AI SDK — Amazon Bedrock Provider](https://ai-sdk.dev/providers/ai-sdk-providers/amazon-bedrock)
- [AI SDK — Azure Provider](https://ai-sdk.dev/providers/ai-sdk-providers/azure)
- [AI SDK — Google Vertex Provider](https://ai-sdk.dev/providers/ai-sdk-providers/google-vertex)
- [AWS — Bedrock AgentCore SDK (TypeScript)](https://github.com/aws/bedrock-agentcore-sdk-typescript)
- [AWS — Strands Agents SDK](https://github.com/strands-agents/sdk-typescript)
