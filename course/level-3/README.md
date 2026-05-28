# Блок 3. Введение в MCP и первый кастомный сервер

**Место в курсе:** после создания универсального клиента `IToolCallingClient` (Блок 2). Вы уже умеете вызывать LLM, передавать инструменты и обрабатывать цикл Function Calling. Теперь мы делаем шаг на уровень ниже: учимся создавать собственный MCP-сервер, предоставляющий инструменты и данные по стандартизованному протоколу.

**Глобальная цель блока:** к концу модулей у вас будет шаблонный MCP-сервер `toolbox`, который вы сможете клонировать под любую задачу, и понимание, как он встраивается в нашу агентную инфраструктуру.

---

## Модуль 3.1. Философия MCP и базовый сервер «Hello, Tools»

### Теория

#### Что такое Model Context Protocol (MCP)?
MCP — это открытый протокол, предложенный Anthropic, который стандартизирует способ предоставления контекста и инструментов для LLM. Представьте USB‑C для ИИ‑приложений: любой MCP‑сервер можно подключить к любому MCP‑клиенту, и они «поймут» друг друга без кастомной интеграции.

Протокол решает две ключевые задачи:
1. **Инструменты (tools)** — функции, которые LLM может вызывать (как в Function Calling).
2. **Ресурсы (resources)** — структурированные данные, которые клиент может читать или на которые может подписываться (файлы, состояния, показания датчиков).

Дополнительно MCP определяет **подсказки (prompts)** — шаблоны, но мы в этом блоке сосредоточимся на инструментах и ресурсах.

#### Архитектура клиент-сервер
Классическая схема:

```
[LLM-хост / оркестратор]  ← MCP-клиент →  [MCP-сервер]
   (ваш TypeScript код)                    (Toolbox и другие серверы)
```

- **Клиент** (мы будем его реализовывать частично, а в Блоке 2 у нас уже есть `IToolCallingClient`) управляет LLM и сам решает, когда вызывать инструменты.
- **Сервер** предоставляет инструменты и ресурсы, ничего не зная о LLM.

Почему это мощно: вы можете запустить десятки специализированных MCP-серверов (файловая система, БД, интернет, Kubernetes), и ваш оркестратор будет динамически собирать «арсенал» инструментов из всех подключённых серверов.

#### Транспорт: stdio и HTTP (SSE)
MCP абстрагирует транспорт. Два основных:
- **stdio** — общение через стандартный ввод/вывод процесса. Идеально для локальных серверов, запускаемых как дочерние процессы. Просто, безопасно, не требует сетевых настроек. Мы начнём с него.
- **HTTP + Server-Sent Events (SSE)** — для удалённых серверов. Клиент подключается к HTTP‑эндпоинту, события приходят через SSE.

> **Аналогия:** stdio — это как разговор по внутренней телефонной линии внутри одного здания. SSE — как радиотрансляция на город: один говорит, остальные слушают.

Для учебных целей stdio идеален: вы запускаете сервер как `node dist/server.js`, клиент стартует его как подпроцесс и общается JSON‑сообщениями.

#### JSON-RPC 2.0 — основа протокола
Все сообщения MCP кодируются в JSON-RPC 2.0. Основные типы сообщений:
- **Request:** `{ jsonrpc: "2.0", id: 1, method: "tools/list", params: {} }`
- **Response (успех):** `{ jsonrpc: "2.0", id: 1, result: { tools: [...] } }`
- **Error:** `{ jsonrpc: "2.0", id: 1, error: { code: -32600, message: "Invalid Request" } }`
- **Notification** (без `id`, не ожидает ответа): `{ jsonrpc: "2.0", method: "initialized" }`

MCP определяет набор методов, например:
- `initialize` → обмен возможностями.
- `tools/list` → получить список инструментов.
- `tools/call` → вызвать инструмент.
- `resources/list`, `resources/read` и др.

#### Обзор пакета `@modelcontextprotocol/sdk`
Официальный SDK на TypeScript предоставляет:
- `Server` — класс для создания MCP-сервера.
- `StdioServerTransport` — транспорт для stdio.
- Обработчики через `server.setRequestHandler(Схема, обработчик)`.
- Предопределённые схемы запросов: `ListToolsRequestSchema`, `CallToolRequestSchema` и т.д.

Эти схемы основаны на Zod, что позволяет нам легко валидировать входящие параметры.

### Практика

#### Шаг 1. Инициализация проекта
Создадим каталог и установим зависимости:

```bash
mkdir toolbox-mcp-server
cd toolbox-mcp-server
npm init -y
npm install @modelcontextprotocol/sdk zod
npm install -D typescript @types/node
npx tsc --init
```

Настроим `tsconfig.json` (целевая среда Node.js 18+, ESNext модули):

```json
{
  "compilerOptions": {
    "target": "ES2022",
    "module": "Node16",
    "moduleResolution": "Node16",
    "outDir": "./dist",
    "rootDir": "./src",
    "strict": true,
    "esModuleInterop": true,
    "declaration": true
  },
  "include": ["src/**/*"]
}
```

#### Шаг 2. Минимальный сервер с `echo`

Создадим `src/server.ts`:

```typescript
import { Server } from "@modelcontextprotocol/sdk/server/index.js";
import { StdioServerTransport } from "@modelcontextprotocol/sdk/server/stdio.js";
import {
  CallToolRequestSchema,
  ListToolsRequestSchema,
} from "@modelcontextprotocol/sdk/types.js";
import { z } from "zod";

/**
 * Создаём экземпляр MCP-сервера с метаданными.
 */
const server = new Server(
  {
    name: "toolbox-mcp-server",
    version: "1.0.0",
  },
  {
    capabilities: {
      tools: {}, // заявляем поддержку инструментов
      // resources: {} // добавим позже
    },
  }
);

/**
 * Регистрируем обработчик списка инструментов.
 * Он будет вызван, когда клиент запросит tools/list.
 */
server.setRequestHandler(ListToolsRequestSchema, async () => {
  return {
    tools: [
      {
        name: "echo",
        description: "Возвращает переданный текст без изменений",
        inputSchema: {
          type: "object",
          properties: {
            text: {
              type: "string",
              description: "Текст для повторения",
            },
          },
          required: ["text"],
        },
      },
    ],
  };
});

/**
 * Обработчик вызова инструмента.
 * Здесь мы определяем логику каждого инструмента.
 */
server.setRequestHandler(CallToolRequestSchema, async (request) => {
  const { name, arguments: args } = request.params;

  switch (name) {
    case "echo": {
      // Валидация и извлечение параметра через Zod
      const schema = z.object({
        text: z.string(),
      });
      const parsed = schema.parse(args);
      return {
        content: [
          {
            type: "text",
            text: parsed.text,
          },
        ],
      };
    }
    default:
      throw new Error(`Неизвестный инструмент: ${name}`);
  }
});

/**
 * Запуск сервера на stdio транспорте.
 */
async function main() {
  const transport = new StdioServerTransport();
  await server.connect(transport);
  console.error("MCP сервер toolbox запущен (stdio)");
}

main().catch((err) => {
  console.error("Ошибка запуска сервера:", err);
  process.exit(1);
});
```

#### Шаг 3. Запуск и ручная проверка

Скомпилируйте и запустите:

```bash
npx tsc
node dist/server.js
```

Сервер будет висеть в ожидании JSON-RPC сообщений на stdin. В другом терминале можно отправить инициализацию (просто для понимания, но обычно это делает клиент). Например:

```bash
echo '{"jsonrpc":"2.0","id":1,"method":"initialize","params":{"protocolVersion":"0.1","clientInfo":{"name":"test","version":"1.0"},"capabilities":{}}}' | node dist/server.js
```

Ответ вы увидите в stdout. Это работает, но для разработки гораздо удобнее MCP Inspector.

#### Шаг 4. MCP Inspector

MCP Inspector — это графический инструмент для тестирования серверов. Установите его глобально:

```bash
npm install -g @anthropic-ai/mcp-inspector
```

Запустите инспектор, указав команду запуска вашего сервера:

```bash
mcp-inspector node dist/server.js
```

Откроется веб‑интерфейс. Вы увидите список инструментов (echo), сможете ввести параметры и вызвать инструмент. Инспектор показывает логи, формат запроса/ответа — идеально для отладки.

### Диаграмма JSON-RPC обмена (ASCII)

```
Клиент (MCP Inspector)         Сервер (stdio)
     |                            |
     |-- tools/list (id:1) ------>|
     |<-- result: [echo] ---------|
     |                            |
     |-- tools/call (id:2) ------>|
     |   { name:"echo",           |
     |     arguments:{text:"Hi"}} |
     |<-- result: "Hi" ----------|
```

### Типичные ошибки и их решение

1. **Сервер не запускается: `connect` не вызывается или transport не обработан**  
   Убедитесь, что вызван `await server.connect(transport);`. Без этого сервер не начнёт слушать stdin.

2. **Ошибка «Method not found» при ручном тестировании**  
   JSON-RPC требует поля `method` и корректную структуру. Проверьте, что отправляете `tools/list`, а не `list_tools`.

3. **Инспектор не подключается: команда запуска не найдена**  
   Передавайте полный путь или убедитесь, что `node` в PATH. Лучше указывать `npx tsx src/server.ts` при использовании TypeScript без компиляции (если установлен `tsx`).

4. **Сервер падает при невалидных аргументах**  
   Мы использовали `schema.parse`, который выбрасывает ZodError. Чтобы сервер не падал, нужно обработать ошибку и вернуть JSON-RPC ошибку. Мы это исправим в Модуле 3.2.

### Вопросы для самопроверки

1. Зачем нужен Model Context Protocol, если уже есть обычное Function Calling?
2. Почему мы начинаем с транспорта stdio, а не HTTP?
3. Какие три обязательных поля присутствуют в любом JSON-RPC запросе?
4. Что произойдёт, если мы не укажем `required` поля в `inputSchema`?
5. Какую роль выполняет `server.setRequestHandler(ListToolsRequestSchema, ...)`?

### Практическое задание

1. Создайте сервер, как описано выше. Проверьте его через MCP Inspector.
2. Добавьте второй инструмент `reverse`, который принимает строку и возвращает её в обратном порядке. Не забудьте обновить список инструментов и добавить case в обработчик вызова. Протестируйте.

---

## Модуль 3.2. Инструменты с параметрами, валидация и обработка ошибок

### Теория

#### JSON Schema для LLM: как писать описания
Когда LLM получает список инструментов, она видит `name`, `description` и `inputSchema` (JSON Schema). Именно по этим полям модель «понимает», что делает инструмент и какие параметры ей нужно передать.

Правила хорошего тона:
- **description** на естественном языке, глагол в инфинитиве: «Складывает два числа».
- **inputSchema.properties** с человеческими `description`.
- Перечисляйте поля в том порядке, в котором их логично вводить.
- Указывайте `required` массив только для обязательных параметров.

#### Валидация через Zod и генерация JSON Schema
Zod позволяет одновременно выполнять и валидацию рантайм-аргументов, и генерировать JSON Schema через библиотеку `zod-to-json-schema`. Это сохраняет единственный источник истины.

Пример:

```typescript
import { z } from "zod";
import { zodToJsonSchema } from "zod-to-json-schema";

const AddArgs = z.object({
  a: z.number().describe("Первое слагаемое"),
  b: z.number().describe("Второе слагаемое"),
});

const inputSchema = zodToJsonSchema(AddArgs);
// используем inputSchema в tools/list
```

#### Паттерн ToolHandler
Чтобы сервер легко расширялся, определим единый интерфейс для инструментов:

```typescript
export interface ToolHandler {
  name: string;
  description: string;
  inputSchema: Record<string, unknown>; // JSON Schema object
  handler: (args: unknown) => Promise<{ content: Array<{ type: "text"; text: string }> }>;
}
```

Все инструменты будем складывать в массив, а регистрировать автоматически.

#### Обработка ошибок MCP
При ошибке мы должны вернуть JSON-RPC ошибку с кодом и сообщением. SDK позволяет выбросить `McpError` с нужным кодом, либо обычную ошибку, которая превратится в `-32603` (Internal error). Лучше явно возвращать `{ isError: true, content: [...] }` или использовать `McpError`.

### Практика

#### Шаг 1. Установка доп. зависимостей

```bash
npm install zod-to-json-schema
```

#### Шаг 2. Переработка сервера с реестром инструментов

Структура каталогов:
```
src/
  server.ts
  tools/
    echo.ts
    add.ts
    multiply.ts
    types.ts
```

`src/tools/types.ts`:

```typescript
/**
 * Интерфейс обработчика инструмента.
 */
export interface ToolHandler {
  name: string;
  description: string;
  inputSchema: Record<string, unknown>;
  handler: (args: unknown) => Promise<{
    content: Array<{ type: "text"; text: string }>;
    isError?: boolean;
  }>;
}
```

`src/tools/echo.ts`:

```typescript
import { z } from "zod";
import { zodToJsonSchema } from "zod-to-json-schema";
import type { ToolHandler } from "./types.js";

const EchoArgs = z.object({
  text: z.string().describe("Текст для повторения"),
});

export const echoTool: ToolHandler = {
  name: "echo",
  description: "Возвращает переданный текст без изменений",
  inputSchema: zodToJsonSchema(EchoArgs),
  handler: async (args: unknown) => {
    const { text } = EchoArgs.parse(args);
    return {
      content: [{ type: "text", text }],
    };
  },
};
```

`src/tools/add.ts`:

```typescript
import { z } from "zod";
import { zodToJsonSchema } from "zod-to-json-schema";
import type { ToolHandler } from "./types.js";

const AddArgs = z.object({
  a: z.number().describe("Первое слагаемое"),
  b: z.number().describe("Второе слагаемое"),
});

export const addTool: ToolHandler = {
  name: "add",
  description: "Складывает два числа",
  inputSchema: zodToJsonSchema(AddArgs),
  handler: async (args: unknown) => {
    const { a, b } = AddArgs.parse(args);
    return {
      content: [{ type: "text", text: String(a + b) }],
    };
  },
};
```

`src/tools/multiply.ts`:

```typescript
import { z } from "zod";
import { zodToJsonSchema } from "zod-to-json-schema";
import type { ToolHandler } from "./types.js";

const MultiplyArgs = z.object({
  x: z.number().describe("Первый множитель"),
  y: z.number().describe("Второй множитель"),
});

export const multiplyTool: ToolHandler = {
  name: "multiply",
  description: "Умножает два числа",
  inputSchema: zodToJsonSchema(MultiplyArgs),
  handler: async (args: unknown) => {
    const { x, y } = MultiplyArgs.parse(args);
    return {
      content: [{ type: "text", text: String(x * y) }],
    };
  },
};
```

#### Шаг 3. Сборка в server.ts

```typescript
import { Server } from "@modelcontextprotocol/sdk/server/index.js";
import { StdioServerTransport } from "@modelcontextprotocol/sdk/server/stdio.js";
import {
  CallToolRequestSchema,
  ListToolsRequestSchema,
} from "@modelcontextprotocol/sdk/types.js";
import { echoTool } from "./tools/echo.js";
import { addTool } from "./tools/add.js";
import { multiplyTool } from "./tools/multiply.js";
import type { ToolHandler } from "./tools/types.js";

const tools: ToolHandler[] = [echoTool, addTool, multiplyTool];

const server = new Server(
  {
    name: "toolbox-mcp-server",
    version: "1.1.0",
  },
  {
    capabilities: {
      tools: {},
    },
  }
);

/**
 * Автоматическая регистрация списка инструментов из массива.
 */
server.setRequestHandler(ListToolsRequestSchema, async () => {
  return {
    tools: tools.map((t) => ({
      name: t.name,
      description: t.description,
      inputSchema: t.inputSchema,
    })),
  };
});

/**
 * Обработчик вызова инструмента с безопасной обработкой ошибок.
 */
server.setRequestHandler(CallToolRequestSchema, async (request) => {
  const { name, arguments: args } = request.params;
  const tool = tools.find((t) => t.name === name);
  if (!tool) {
    return {
      content: [{ type: "text", text: `Инструмент "${name}" не найден` }],
      isError: true,
    };
  }

  try {
    return await tool.handler(args);
  } catch (err: any) {
    // ZodError или другие исключения
    return {
      content: [
        {
          type: "text",
          text: `Ошибка выполнения инструмента "${name}": ${err.message}`,
        },
      ],
      isError: true,
    };
  }
});

/**
 * Точка входа.
 */
async function main() {
  const transport = new StdioServerTransport();
  await server.connect(transport);
  console.error("Toolbox MCP сервер запущен");
}

main().catch((err) => {
  console.error("Ошибка запуска:", err);
  process.exit(1);
});
```

#### Шаг 4. Тестирование через Inspector

Запускаем инспектор:

```bash
mcp-inspector node dist/server.js
```

Проверяем:
- `add` с `a: 3, b: 5` → результат `8`.
- `multiply` с `x: 4, y: 7` → `28`.
- Передаём невалидные параметры, например, `a: "строка"` → сервер должен вернуть ошибку с понятным текстом.

### Типичные ошибки и их решение

1. **ZodError: Required при отсутствии обязательного поля**  
   Проверьте, что передаёте все поля из схемы. В инспекторе заполняйте поля в соответствии с `required` в JSON Schema.

2. **Генерация JSON Schema через zod-to-json-schema не содержит `type: "object"`**  
   По умолчанию библиотека включает `type`. Если нет, оберните схему в `z.object(...)`. Для простых схем всё генерируется корректно.

3. **Ошибка «content missing» или неверный формат ответа**  
   Ответ инструмента должен содержать массив `content`, где каждый элемент имеет `type: "text"` и `text`. Если забыли обернуть текст в массив, клиент упадёт.

4. **Сервер падает при необработанном исключении в обработчике**  
   Обязательно оберните вызов `tool.handler` в try/catch и возвращайте ошибку через `isError: true`, а не дайте исключению уйти в транспорт.

### Вопросы для самопроверки

1. Почему выгодно использовать Zod для описания параметров, а не писать JSON Schema вручную?
2. Какое поле в ответе инструмента указывает клиенту, что вызов завершился ошибкой?
3. Что произойдёт, если инструмент не будет найден в реестре при вызове `tools/call`?
4. Как MCP-клиент понимает, какие параметры обязательны?

### Практическое задание

1. Добавьте инструмент `divide` (деление). Не забудьте обработку деления на ноль — возвращайте осмысленную ошибку.
2. Создайте инструмент `random`, генерирующий случайное целое число в заданном диапазоне (параметры min, max). Интегрируйте его в сервер.

---

## Модуль 3.3. Ресурсы и подключение к LLM-клиенту

### Теория

#### Ресурсы в MCP
Ресурсы — это модель для предоставления контекстных данных, которые клиент может читать. Они идентифицируются уникальным URI, например `clock://current`. Ресурс может иметь тип `text` или `binary`. Клиент может запросить список ресурсов (`resources/list`), прочитать конкретный (`resources/read` с параметром `uri`), а также подписаться на обновления (`resources/subscribe`), тогда сервер будет отправлять уведомления `notifications/resources/updated`.

Аналогия: инструменты — это «нажатие кнопок», ресурсы — это «показания датчиков». Часы — идеальный пример ресурса: мы не вызываем команду «показать время», а читаем ресурс, представляющий текущее время.

#### Интеграция с IToolCallingClient (из Блока 2)
У нас есть абстрактный клиент `IToolCallingClient`, который умеет вызывать LLM и передавать определения инструментов в формате провайдера. Чтобы подключить MCP-сервер, мы напишем небольшой MCP-клиент, который:
1. Запускает сервер как дочерний процесс (stdio).
2. Получает список инструментов.
3. Преобразует их в формат `ToolDef`, ожидаемый нашим универсальным клиентом.
4. В цикле Function Calling: LLM выдаёт запрос на вызов инструмента → MCP-клиент выполняет `tools/call` → результат добавляется в историю диалога.

Это классический паттерн «MCP bridge».

### Практика

#### Шаг 1. Добавление ресурса `clock://current` в toolbox-сервер

Расширим возможности сервера. Добавим в `server` поддержку ресурсов.

В `src/server.ts` после создания `server`:

```typescript
import {
  ListResourcesRequestSchema,
  ReadResourceRequestSchema,
} from "@modelcontextprotocol/sdk/types.js";

// В capabilities добавим resources: {}
const server = new Server(
  { name: "toolbox-mcp-server", version: "1.2.0" },
  {
    capabilities: {
      tools: {},
      resources: {}, // включили поддержку ресурсов
    },
  }
);

/**
 * Обработчик списка ресурсов.
 */
server.setRequestHandler(ListResourcesRequestSchema, async () => {
  return {
    resources: [
      {
        uri: "clock://current",
        name: "Текущее время",
        description: "Текущее системное время в ISO 8601",
        mimeType: "text/plain",
      },
    ],
  };
});

/**
 * Чтение ресурса по URI.
 */
server.setRequestHandler(ReadResourceRequestSchema, async (request) => {
  const uri = request.params.uri;
  if (uri === "clock://current") {
    const now = new Date().toISOString();
    return {
      contents: [
        {
          uri,
          mimeType: "text/plain",
          text: now,
        },
      ],
    };
  }
  throw new Error(`Ресурс ${uri} не найден`);
});
```

Теперь toolbox предоставляет и инструменты, и ресурс времени.

#### Шаг 2. MCP-клиент на TypeScript

Установим клиентскую часть SDK:

```bash
npm install @modelcontextprotocol/sdk
```

Создадим файл `src/mcp-client.ts` — это отдельная программа, демонстрирующая интеграцию. В реальном проекте она будет частью оркестратора.

```typescript
import { Client } from "@modelcontextprotocol/sdk/client/index.js";
import { StdioClientTransport } from "@modelcontextprotocol/sdk/client/stdio.js";
import { spawn } from "child_process";

/**
 * Запускает MCP-сервер как дочерний процесс и инициализирует клиент.
 * @param command Команда запуска сервера (например "node dist/server.js")
 * @returns Готовый MCP-клиент
 */
async function createMCPClient(command: string): Promise<Client> {
  const [cmd, ...args] = command.split(" ");
  const child = spawn(cmd, args, {
    stdio: ["pipe", "pipe", process.stderr],
  });

  const transport = new StdioClientTransport({
    reader: child.stdout.getReader(),
    writer: child.stdin.getWriter(),
  });

  const client = new Client(
    {
      name: "example-mcp-client",
      version: "1.0.0",
    },
    {
      capabilities: {},
    }
  );

  await client.connect(transport);
  return client;
}

/**
 * Преобразует инструменты MCP в упрощённый формат для LLM (ToolDef).
 */
async function getToolsForLLM(client: Client) {
  const { tools } = await client.listTools();
  return tools.map((tool) => ({
    name: tool.name,
    description: tool.description,
    parameters: tool.inputSchema, // некоторые провайдеры ожидают JSON Schema
  }));
}

// Демонстрация (без полной интеграции с IToolCallingClient – дадим схему)
async function main() {
  const client = await createMCPClient("node dist/server.js");

  const tools = await getToolsForLLM(client);
  console.log("Доступные инструменты:", JSON.stringify(tools, null, 2));

  // Вызов инструмента напрямую
  const result = await client.callTool({
    name: "add",
    arguments: { a: 15, b: 27 },
  });
  console.log("Результат add:", result.content);

  await client.close();
}

main().catch(console.error);
```

#### Шаг 3. Интеграция с IToolCallingClient (концептуальная схема)

В реальном оркестраторе (из Блока 2) у вас есть метод `runConversation`, который принимает массив `ToolDef` и управляет циклом. Интегрируем MCP следующим образом:

```typescript
// Примерный код (не полный, но иллюстрирует подход)
import { IToolCallingClient } from "../core/interfaces.js";

async function runWithMCPTools(
  llmClient: IToolCallingClient,
  mcpClient: Client,
  userMessage: string
) {
  // Получаем инструменты от MCP и преобразуем
  const { tools: mcpTools } = await mcpClient.listTools();
  const toolDefs = mcpTools.map((t) => ({
    name: t.name,
    description: t.description,
    parameters: t.inputSchema as Record<string, unknown>,
  }));

  // Определяем общий обработчик вызова инструментов
  async function executeToolCall(name: string, args: Record<string, unknown>) {
    const result = await mcpClient.callTool({ name, arguments: args });
    // Извлекаем текст из контента
    return result.content
      .filter((c) => c.type === "text")
      .map((c) => c.text)
      .join("\n");
  }

  // Запускаем стандартный цикл, передавая toolDefs и обработчик
  const finalAnswer = await llmClient.runConversation(
    userMessage,
    toolDefs,
    executeToolCall
  );
  return finalAnswer;
}
```

Тестовый сценарий: «Сложи 15 и 27, потом умножь результат на 3 и скажи, который сейчас час». В этом сценарии LLM должна:
- Вызвать `add(15, 27)` → получить 42.
- Вызвать `multiply(42, 3)` → получить 126.
- Прочитать ресурс `clock://current` (но ресурс — не инструмент, LLM не может его вызвать напрямую, если только мы не сделаем специальный инструмент `get_time`). Поэтому лучше добавить инструмент `get_current_time`, который внутри читает ресурс. Это демонстрирует, что ресурсы можно использовать внутри сервера для реализации инструментов.

**Практическое дополнение:** Добавим в toolbox инструмент `get_current_time`, использующий `clock://current` логику:

```typescript
// src/tools/getCurrentTime.ts
import type { ToolHandler } from "./types.js";
import { z } from "zod";
import { zodToJsonSchema } from "zod-to-json-schema";

const TimeArgs = z.object({}); // без параметров

export const getCurrentTimeTool: ToolHandler = {
  name: "get_current_time",
  description: "Возвращает текущее время в ISO-формате",
  inputSchema: zodToJsonSchema(TimeArgs),
  handler: async () => {
    const now = new Date().toISOString();
    return {
      content: [{ type: "text", text: now }],
    };
  },
};
```

Подключим в общий реестр. Теперь LLM может напрямую вызывать этот инструмент.

### ASCII диаграмма цикла с MCP и LLM

```
User --> Orchestrator --> LLM
                           |
        (Function Call)    | add(15,27)
           <---------------|
Orchestrator --> MCP Client --> MCP Server (tools/call)
                                   |
           <-- result: 42 ---------|
Orchestrator --> LLM (с результатом)
                           |
        (Function Call)    | multiply(42,3)
           <---------------|
... аналогично ...
Orchestrator --> LLM (конечный ответ) --> User
```

### Типичные ошибки и их решение

1. **MCP-клиент не может подключиться: ошибка чтения стримов**  
   Проверьте, что команда запуска сервера корректна и сервер реально запускается. В `StdioClientTransport` нужно передать reader из stdout и writer в stdin.

2. **Инструменты не видны LLM: не тот формат `parameters`**  
   Некоторые LLM-провайдеры ожидают, что `parameters` будет объектом JSON Schema. Убедитесь, что передаёте `inputSchema` без обёрток.

3. **LLM пытается прочитать ресурс, а не вызвать инструмент**  
   Ресурсы не вызываются как функции. Обычно для получения времени создают инструмент-обёртку.

4. **Сервер падает при запросе ресурса с неверным URI**  
   Всегда обрабатывайте неизвестные URI и возвращайте ошибку, чтобы клиент не завис.

### Вопросы для самопроверки

1. Чем ресурс отличается от инструмента? Приведите примеры из реального мира.
2. Зачем может потребоваться инструмент `get_current_time`, если уже есть ресурс `clock://current`?
3. Какие шаги нужно выполнить для подключения MCP-сервера к универсальному клиенту `IToolCallingClient`?
4. Что произойдёт, если MCP-сервер вернёт ответ не в формате `content: [{ type: "text", text: ... }]`?

### Практическое задание

1. Добавьте в toolbox ресурс `config://version`, возвращающий версию сервера (строку). Научите клиент читать этот ресурс и выводить.
2. Модифицируйте клиент из `mcp-client.ts` так, чтобы он реализовывал описанный выше цикл с LLM: примите текстовый запрос, получите список инструментов, выполните несколько вызовов и выведите итоговый ответ. Используйте класс `IToolCallingClient` из Блока 2 (или заглушку, если нет готового).

---

## Итоговое домашнее задание (Блок 3)

### Создание MCP-сервера `file-manager`

**Цель:** закрепить все полученные навыки, создав полноценный сервер для безопасной работы с файловой системой через MCP, и протестировать его с LLM.

#### Требования

1. **Сервер `file-manager`** должен предоставлять три инструмента:
   - `list_directory` (параметр `path: string`) — возвращает список файлов и папок в указанной директории. Ограничьте рабочую область «песочницей»: базовый путь задаётся переменной окружения `SANDBOX_ROOT` или аргументом командной строки. Любые попытки выйти за пределы песочницы (пути с `..`) должны блокироваться с ошибкой.
   - `read_file` (параметр `filePath: string`) — читает текстовый файл (UTF-8) и возвращает его содержимое. Ограничено размером, например, 1 МБ.
   - `write_file` (параметры `filePath: string`, `content: string`) — записывает (перезаписывает) текстовый файл. Тоже в пределах песочницы.

2. **Валидация и безопасность:**
   - Все пути нормализуются и проверяются, что они находятся внутри `SANDBOX_ROOT`.
   - Используйте Zod для описания параметров.
   - Возвращайте понятные ошибки при попытке несанкционированного доступа, чтения несуществующего файла и т.д.

3. **Подключение к LLM-клиенту:**
   - Напишите небольшой скрипт `orchestrator.ts`, который:
     - Принимает текстовую задачу от пользователя (через аргументы командной строки).
     - Подключается к вашему `file-manager` серверу через MCP-клиент.
     - Использует `IToolCallingClient` (или упрощённую версию) с LLM (DeepSeek/YandexGPT) для выполнения задачи.
     - Демонстрирует сценарий: «Создай файл `hello.txt` с текстом 'Привет, MCP!', затем прочитай его и выведи содержимое».

4. **Документация:** в `README.md` опишите, как запустить сервер, инспектор и оркестратор.

#### Критерии оценки
- Сервер запускается и работает как MCP-процесс (stdio).
- Все три инструмента корректно функционируют через MCP Inspector.
- Безопасность: невозможно прочитать файл вне песочницы.
- Оркестратор успешно взаимодействует с LLM и сервером, выполняя поставленную задачу.
- Код структурирован, используются паттерны из модулей.

#### Подсказки
- Для нормализации пути используйте `path.resolve` и `path.normalize`.
- При реализации sandbox проверяйте, что `normalizedPath.startsWith(sandboxRoot)`.
- В `write_file` создавайте директории рекурсивно при необходимости (`fs.mkdirSync(dirname, { recursive: true })`).
- Для интеграции с LLM можно взять готовый `ToolCallingClient` из Блока 2 и просто передать ему преобразованные инструменты.

#### Ссылки на документацию
- [MCP TypeScript SDK](https://github.com/anthropics/anthropic-tools/tree/main/mcp/typescript)
- [MCP Inspector](https://www.npmjs.com/package/@anthropic-ai/mcp-inspector)
- [Zod](https://zod.dev)
- [zod-to-json-schema](https://github.com/StefanTerdell/zod-to-json-schema)

Удачи, и добро пожаловать в мир агентных систем на MCP!