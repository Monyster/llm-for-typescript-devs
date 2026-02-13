# 🚀 Доповнення до курсу — Відсутні теми

> Цей файл містить теми, які були пропущені в основному курсі.

---

## 📚 Few-Shot Prompting

### Що це таке?

**Few-shot prompting** — техніка, коли ви даєте моделі кілька прикладів бажаної поведінки перед основним запитом. Модель "вчиться" на цих прикладах і відповідає у такому ж стилі/форматі.

```
┌─────────────────────────────────────────────────────────────────┐
│                    FEW-SHOT PROMPTING                           │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  Zero-shot:   "Переклади на англійську: Привіт"                │
│               → "Hello" (модель сама здогадується)             │
│                                                                 │
│  One-shot:    "Приклад: Добрий день → Good afternoon           │
│               Переклади: Привіт"                                │
│               → "Hello" (один приклад)                          │
│                                                                 │
│  Few-shot:    "Приклади:                                       │
│               Добрий день → Good afternoon                      │
│               Дякую → Thank you                                 │
│               На добраніч → Good night                          │
│               Переклади: Привіт"                                │
│               → "Hello" (кілька прикладів)                      │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### Коли використовувати?

| Сценарій | Zero-shot | Few-shot |
|----------|-----------|----------|
| Простий переклад | ✅ | Не потрібно |
| Специфічний формат виводу | ❌ | ✅ |
| Класифікація з вашими категоріями | ❌ | ✅ |
| Стиль/тон відповіді | ⚠️ | ✅ |
| Доменна термінологія | ❌ | ✅ |

### Практичний приклад

```typescript
// few-shot-example.ts
import { generateText } from 'ai';
import { openai } from '@ai-sdk/openai';

async function classifyTicket(ticket: string): Promise<string> {
  const { text } = await generateText({
    model: openai('gpt-4o-mini'),
    prompt: `Класифікуй тікет підтримки в одну з категорій: billing, technical, account, other.

Приклади:
Тікет: "Не можу оплатити карткою Visa"
Категорія: billing

Тікет: "Додаток вилітає при відкритті"
Категорія: technical

Тікет: "Як змінити email в профілі?"
Категорія: account

Тікет: "Коли буде знижка на Black Friday?"
Категорія: other

Тікет: "${ticket}"
Категорія:`
  });
  
  return text.trim();
}

// Використання
const category = await classifyTicket("Мене двічі списали гроші за підписку");
console.log(category); // billing
```

### 💡 Best Practices

1. **3-5 прикладів** зазвичай достатньо (більше не завжди краще)
2. **Різноманітні приклади** — покрийте edge cases
3. **Консистентний формат** — однаковий стиль у всіх прикладах
4. **Релевантні приклади** — схожі на реальні запити

---

## 🔧 Tool vs Skill в контексті AI агентів

### Визначення

| Концепція | Опис | Приклад |
|-----------|------|---------|
| **Tool** | Конкретна функція/API яку агент може викликати | `getWeather()`, `sendEmail()`, `searchDB()` |
| **Skill** | Набір tools + логіка для виконання складної задачі | "Бронювання подорожі" = пошук рейсів + готелів + оренда авто |

### Візуалізація

```
┌─────────────────────────────────────────────────────────────────┐
│                         AI AGENT                                │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  SKILLS (високорівневі можливості)                             │
│  ┌─────────────────────────────────────────────────────────┐   │
│  │  "Travel Booking" Skill                                  │   │
│  │  ┌─────────┐  ┌─────────┐  ┌─────────┐  ┌─────────┐    │   │
│  │  │ search  │  │ search  │  │  book   │  │  send   │    │   │
│  │  │ Flights │  │ Hotels  │  │ Payment │  │ Confirm │    │   │
│  │  └─────────┘  └─────────┘  └─────────┘  └─────────┘    │   │
│  │       ↓            ↓            ↓            ↓          │   │
│  │  [TOOLS]      [TOOLS]      [TOOLS]      [TOOLS]        │   │
│  └─────────────────────────────────────────────────────────┘   │
│                                                                 │
│  TOOLS (атомарні операції)                                     │
│  ┌─────────┐ ┌─────────┐ ┌─────────┐ ┌─────────┐              │
│  │searchAPI│ │ database│ │ payment │ │  email  │              │
│  │         │ │  query  │ │ gateway │ │  send   │              │
│  └─────────┘ └─────────┘ └─────────┘ └─────────┘              │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### Коли що використовувати?

**Tools** — коли агенту потрібно:
- Отримати зовнішні дані (API, DB)
- Виконати конкретну дію (відправити email)
- Інтегруватись з сервісами

**Skills** — коли потрібно:
- Складний workflow з кількох кроків
- Повторювана бізнес-логіка
- Абстракція для нетехнічних користувачів

### Приклад в коді

```typescript
// Tools — атомарні операції
const tools = {
  searchFlights: {
    description: 'Шукає рейси',
    parameters: z.object({ from: z.string(), to: z.string(), date: z.string() }),
    execute: async (params) => { /* API call */ }
  },
  searchHotels: { /* ... */ },
  bookFlight: { /* ... */ },
  sendConfirmation: { /* ... */ }
};

// Skill — композиція tools з логікою
async function travelBookingSkill(request: {
  destination: string;
  dates: { start: string; end: string };
  budget: number;
}) {
  // 1. Пошук рейсів
  const flights = await tools.searchFlights.execute({ /* ... */ });
  
  // 2. Пошук готелів
  const hotels = await tools.searchHotels.execute({ /* ... */ });
  
  // 3. Вибір найкращого варіанту в межах бюджету
  const bestOption = selectBestOption(flights, hotels, request.budget);
  
  // 4. Бронювання
  const booking = await tools.bookFlight.execute({ /* ... */ });
  
  // 5. Підтвердження
  await tools.sendConfirmation.execute({ /* ... */ });
  
  return booking;
}
```

---

## 🔌 MCP vs REST API

### Чим MCP відрізняється від звичайного REST API?

| Аспект | REST API | MCP |
|--------|----------|-----|
| **Призначення** | Загальний обмін даними | Спеціально для LLM інтеграцій |
| **Протокол** | HTTP request/response | JSON-RPC over stdio/SSE |
| **Discovery** | Swagger/OpenAPI (опціонально) | Вбудований (`listTools`, `listResources`) |
| **Контекст** | Stateless | Може зберігати сесію |
| **Типи операцій** | CRUD | Tools (дії) + Resources (дані) + Prompts (шаблони) |
| **Безпека** | OAuth, API keys | Локальний процес, ізольований |

### Чому MCP, а не просто REST?

```
┌─────────────────────────────────────────────────────────────────┐
│                    REST API підхід                              │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  Claude                                                         │
│     │                                                           │
│     ├──▶ Tool: callRestAPI({ url, method, body })              │
│     │        │                                                  │
│     │        ▼                                                  │
│     │    🌐 Internet ──▶ Your API ──▶ Database                 │
│     │                                                           │
│  ❌ Проблеми:                                                   │
│  • Потрібен public endpoint                                    │
│  • Безпека: API key в промпті                                  │
│  • Latency: мережа + Claude + мережа                           │
│  • Модель не знає структуру API без документації               │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────┐
│                       MCP підхід                                │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  Claude Desktop                                                 │
│     │                                                           │
│     ├──▶ MCP Server (локальний процес)                         │
│     │        │                                                  │
│     │        ▼                                                  │
│     │    📁 Local Files / 🗄️ Local DB / 🔐 Secure APIs        │
│     │                                                           │
│  ✅ Переваги:                                                   │
│  • Локальний доступ (без public endpoint)                      │
│  • Безпека: credentials в env, не в промпті                    │
│  • Швидкість: stdio швидше ніж HTTP                            │
│  • Auto-discovery: Claude знає доступні tools                  │
│  • Стандартизація: один протокол для всіх інтеграцій          │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### Коли що використовувати?

**REST API:**
- Публічний сервіс для багатьох клієнтів
- Web/Mobile додатки
- Інтеграції з third-party

**MCP:**
- Локальні інструменти для Claude Desktop
- Доступ до приватних/локальних даних
- Безпечна інтеграція без публічних endpoints

---

## 📦 MCP Components: Resources, Tools, Prompts

### Три типи можливостей MCP сервера

```
┌─────────────────────────────────────────────────────────────────┐
│                    MCP SERVER CAPABILITIES                       │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  ┌─────────────────────────────────────────────────────────┐   │
│  │  📄 RESOURCES — дані для читання                        │   │
│  │                                                          │   │
│  │  • Файли (file://path/to/doc.md)                        │   │
│  │  • Database records (db://users/123)                    │   │
│  │  • API responses (api://weather/kyiv)                   │   │
│  │                                                          │   │
│  │  Особливість: READ-ONLY, URI-based                      │   │
│  └─────────────────────────────────────────────────────────┘   │
│                                                                 │
│  ┌─────────────────────────────────────────────────────────┐   │
│  │  🔧 TOOLS — дії для виконання                           │   │
│  │                                                          │   │
│  │  • Модифікація даних (createFile, updateDB)             │   │
│  │  • Зовнішні виклики (sendEmail, postSlack)              │   │
│  │  • Обчислення (calculate, transform)                    │   │
│  │                                                          │   │
│  │  Особливість: можуть змінювати стан, мають параметри    │   │
│  └─────────────────────────────────────────────────────────┘   │
│                                                                 │
│  ┌─────────────────────────────────────────────────────────┐   │
│  │  💬 PROMPTS — готові шаблони промптів                   │   │
│  │                                                          │   │
│  │  • Інструкції для специфічних задач                     │   │
│  │  • Шаблони з плейсхолдерами                             │   │
│  │  • Доменні best practices                               │   │
│  │                                                          │   │
│  │  Особливість: користувач може вибрати і застосувати     │   │
│  └─────────────────────────────────────────────────────────┘   │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### Приклад MCP сервера з усіма типами

```typescript
// mcp-full-example.ts
import { Server } from '@modelcontextprotocol/sdk/server/index.js';

const server = new Server({
  name: 'my-mcp-server',
  version: '1.0.0',
}, {
  capabilities: {
    resources: {},  // Підтримуємо resources
    tools: {},      // Підтримуємо tools
    prompts: {}     // Підтримуємо prompts
  }
});

// === RESOURCES ===
// Дані для читання
server.setRequestHandler(ListResourcesRequestSchema, async () => ({
  resources: [
    {
      uri: 'config://app/settings',
      name: 'App Settings',
      mimeType: 'application/json',
      description: 'Current application configuration'
    },
    {
      uri: 'docs://api/readme',
      name: 'API Documentation',
      mimeType: 'text/markdown'
    }
  ]
}));

server.setRequestHandler(ReadResourceRequestSchema, async (request) => {
  if (request.params.uri === 'config://app/settings') {
    return {
      contents: [{
        uri: request.params.uri,
        mimeType: 'application/json',
        text: JSON.stringify({ theme: 'dark', language: 'uk' })
      }]
    };
  }
  // ... handle other resources
});

// === TOOLS ===
// Дії для виконання
server.setRequestHandler(ListToolsRequestSchema, async () => ({
  tools: [
    {
      name: 'update_settings',
      description: 'Updates application settings',
      inputSchema: {
        type: 'object',
        properties: {
          key: { type: 'string' },
          value: { type: 'string' }
        },
        required: ['key', 'value']
      }
    }
  ]
}));

server.setRequestHandler(CallToolRequestSchema, async (request) => {
  if (request.params.name === 'update_settings') {
    const { key, value } = request.params.arguments;
    // Виконуємо дію
    await updateSettings(key, value);
    return {
      content: [{ type: 'text', text: `Updated ${key} to ${value}` }]
    };
  }
});

// === PROMPTS ===
// Готові шаблони
server.setRequestHandler(ListPromptsRequestSchema, async () => ({
  prompts: [
    {
      name: 'code_review',
      description: 'Template for code review requests',
      arguments: [
        { name: 'language', description: 'Programming language', required: true },
        { name: 'focus', description: 'What to focus on', required: false }
      ]
    }
  ]
}));

server.setRequestHandler(GetPromptRequestSchema, async (request) => {
  if (request.params.name === 'code_review') {
    const { language, focus = 'general quality' } = request.params.arguments || {};
    return {
      messages: [
        {
          role: 'user',
          content: {
            type: 'text',
            text: `Please review the following ${language} code. Focus on: ${focus}.

Code:
\`\`\`${language}
{{CODE_HERE}}
\`\`\`

Provide:
1. Summary of what the code does
2. Issues found (bugs, security, performance)
3. Suggestions for improvement
4. Overall quality score (1-10)`
          }
        }
      ]
    };
  }
});
```

### Коли використовувати що?

| Компонент | Використовуйте для | Приклад |
|-----------|-------------------|---------|
| **Resource** | Читання даних без модифікації | Перегляд конфігу, документації |
| **Tool** | Виконання дій, зміна стану | Відправка email, оновлення DB |
| **Prompt** | Готові шаблони задач | Code review template, report generator |

---

## 🤔 Додаткові питання для самоперевірки

1. Чим few-shot prompting відрізняється від fine-tuning?
2. Коли skill краще ніж набір окремих tools?
3. Чому MCP сервер запускається як окремий процес?
4. Яка різниця між MCP Resource і Tool?

<details>
<summary>📝 Відповіді</summary>

1. **Few-shot** — приклади в промпті, працює одразу, временний ефект. **Fine-tuning** — донавчання моделі, потребує часу/даних, постійний ефект.

2. Skill краще коли: є повторюваний бізнес-процес, потрібна єдина точка входу, треба абстрагувати складність від користувача.

3. Ізоляція процесів забезпечує безпеку. Якщо MCP сервер впаде — Claude Desktop продовжить працювати. Credentials ізольовані.

4. **Resource** — read-only, URI-based, для отримання даних. **Tool** — може змінювати стан, має параметри, для виконання дій.
</details>
