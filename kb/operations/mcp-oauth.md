# Модуль 24: OAuth для MCP — Автентифікація та production deployment

## 🎯 Що ви отримаєте з цього модуля

Після проходження ви будете:
- Реалізовувати OAuth 2.1 для MCP-серверів
- Захищати MCP-сервери в production (HTTPS, rate limiting, CORS)
- Деплоїти MCP-сервери як standalone HTTP сервіси
- Розуміти SSE transport та remote MCP з'єднання

**Які задачі це дозволяє вирішувати:** Модуль 11 навчив створювати MCP-сервери через stdio (локально). Цей модуль вчить робити їх production-ready: з автентифікацією, через інтернет, безпечно.

---

## 24.1 Два транспорти MCP: stdio vs HTTP/SSE

### stdio (локальний)

```
AI Client ←── stdin/stdout ──→ MCP Server (процес на тій же машині)
```

Працює тільки локально. Використовується в Claude Desktop, Cursor.

### HTTP/SSE (remote)

```
AI Client ←── HTTPS + SSE ──→ MCP Server (будь-де в інтернеті)
```

Працює через інтернет. Потрібен для production.

```typescript
// Підключення до remote MCP-сервера
import { createMCPClient } from '@ai-sdk/mcp';

const client = await createMCPClient({
  transport: {
    type: 'sse',  // Server-Sent Events
    url: 'https://mcp.yourcompany.com/sse',
    headers: {
      'Authorization': `Bearer ${accessToken}`,
    },
  },
});

const tools = await client.tools();
```

---

## 24.2 OAuth 2.1 для MCP

MCP специфікація рекомендує OAuth 2.1 для автентифікації remote серверів.

### Потік автентифікації

```
1. Client запитує /.well-known/oauth-authorization-server
2. Server повертає authorization endpoint, token endpoint
3. Client перенаправляє користувача на authorization endpoint
4. Користувач логіниться, дає дозвіл
5. Client отримує authorization code
6. Client обмінює code на access token
7. Client використовує token для запитів до MCP-сервера
```

### Реалізація MCP-сервера з OAuth

```typescript
// mcp-server-with-auth.ts
import { McpServer } from '@modelcontextprotocol/sdk/server/mcp.js';
import { SSEServerTransport } from '@modelcontextprotocol/sdk/server/sse.js';
import express from 'express';
import { z } from 'zod';

const app = express();

// OAuth metadata endpoint
app.get('/.well-known/oauth-authorization-server', (req, res) => {
  res.json({
    issuer: 'https://mcp.yourcompany.com',
    authorization_endpoint: 'https://auth.yourcompany.com/authorize',
    token_endpoint: 'https://auth.yourcompany.com/token',
    scopes_supported: ['read', 'write', 'admin'],
    response_types_supported: ['code'],
    grant_types_supported: ['authorization_code', 'refresh_token'],
    code_challenge_methods_supported: ['S256'], // PKCE обов'язковий в OAuth 2.1
  });
});

// Middleware для перевірки токенів
async function verifyToken(req: express.Request, res: express.Response, next: express.NextFunction) {
  const authHeader = req.headers.authorization;
  if (!authHeader?.startsWith('Bearer ')) {
    return res.status(401).json({ error: 'Missing or invalid Authorization header' });
  }

  const token = authHeader.slice(7);

  try {
    // Верифікація через ваш auth provider
    const payload = await verifyJWT(token, {
      audience: 'mcp-server',
      issuer: 'https://auth.yourcompany.com',
    });

    req.user = payload;
    req.scopes = payload.scope?.split(' ') || [];
    next();
  } catch (error) {
    return res.status(401).json({ error: 'Invalid token' });
  }
}

// MCP Server
const server = new McpServer({
  name: 'company-data-server',
  version: '1.0.0',
});

// Tool з перевіркою прав доступу
server.tool(
  'query_customers',
  'Пошук клієнтів у базі даних',
  {
    query: z.string(),
    limit: z.number().default(10),
  },
  async ({ query, limit }, { meta }) => {
    // Перевірка scope
    if (!meta?.scopes?.includes('read')) {
      return {
        content: [{ type: 'text', text: 'Error: insufficient permissions. Required scope: read' }],
        isError: true,
      };
    }

    const results = await db.customers.search(query, limit);
    return {
      content: [{ type: 'text', text: JSON.stringify(results, null, 2) }],
    };
  }
);

server.tool(
  'update_customer',
  'Оновити дані клієнта',
  {
    customerId: z.string(),
    updates: z.record(z.string()),
  },
  async ({ customerId, updates }, { meta }) => {
    if (!meta?.scopes?.includes('write')) {
      return {
        content: [{ type: 'text', text: 'Error: insufficient permissions. Required scope: write' }],
        isError: true,
      };
    }

    const result = await db.customers.update(customerId, updates);
    return {
      content: [{ type: 'text', text: `Customer ${customerId} updated successfully` }],
    };
  }
);

// SSE endpoint з автентифікацією
app.get('/sse', verifyToken, (req, res) => {
  const transport = new SSEServerTransport('/messages', res);
  // Передаємо user info та scopes в MCP server context
  transport.meta = { user: req.user, scopes: req.scopes };
  server.connect(transport);
});

app.post('/messages', verifyToken, (req, res) => {
  // Обробка вхідних повідомлень від клієнта
});

app.listen(3001, () => {
  console.log('MCP Server running on https://mcp.yourcompany.com:3001');
});
```

---

## 24.3 Production Deployment

### Docker

```dockerfile
FROM node:22-alpine
WORKDIR /app
COPY package*.json ./
RUN npm ci --production
COPY . .
EXPOSE 3001
CMD ["node", "dist/mcp-server.js"]
```

### Nginx reverse proxy з HTTPS

```nginx
server {
    listen 443 ssl;
    server_name mcp.yourcompany.com;

    ssl_certificate /etc/ssl/certs/cert.pem;
    ssl_certificate_key /etc/ssl/private/key.pem;

    location / {
        proxy_pass http://localhost:3001;
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection "upgrade";
        # SSE: вимкнути буферизацію
        proxy_buffering off;
        proxy_cache off;
        proxy_read_timeout 86400s;
    }
}
```

### Rate Limiting

```typescript
import rateLimit from 'express-rate-limit';

// Загальний ліміт
const generalLimiter = rateLimit({
  windowMs: 60 * 1000,  // 1 хвилина
  max: 60,               // 60 запитів/хв
  message: { error: 'Rate limit exceeded' },
});

// Ліміт на tool calls (дорожчі операції)
const toolLimiter = rateLimit({
  windowMs: 60 * 1000,
  max: 20,
  message: { error: 'Tool call rate limit exceeded' },
});

app.use('/sse', generalLimiter);
app.use('/messages', toolLimiter);
```

---

## 24.4 Security Checklist для production MCP

**Transport:** Тільки HTTPS (ніколи HTTP), certificate pinning для критичних серверів, timeout на всі з'єднання.

**Auth:** OAuth 2.1 з PKCE, JWT з коротким TTL (15 хв), refresh tokens з rotation, scope-based access control на кожен tool.

**Input:** Валідація всіх параметрів через Zod, rate limiting per-user та per-tool, максимальний розмір запиту (body limit).

**Monitoring:** Логування всіх tool calls з user ID, алерти на аномальну активність, audit trail для деструктивних операцій.

---

## Перевір себе

1. Чим HTTP/SSE транспорт відрізняється від stdio? Коли що?
2. Навіщо OAuth 2.1 для MCP і що таке PKCE?
3. Як реалізувати scope-based access control для MCP tools?
4. Створіть MCP-сервер з HTTP/SSE транспортом та базовою auth перевіркою
5. Які елементи потрібні для production deployment MCP?

---

**Назад:** [← Модуль 23 — Mastra](23-mastra.md) | **Далі:** [Модуль 25 — Multi-agent orchestration →](25-multi-agent.md)
