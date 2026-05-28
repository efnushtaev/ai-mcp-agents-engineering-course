# Блок 7. Тестирование и отладка

Добро пожаловать в финальный блок нашего курса! Вы уже построили сложную агентную систему: пара MCP-серверов с эмуляторами промышленных устройств, универсальный клиент для Tool Calling, конфигурацию оркестратора OpenCode и сквозные сценарии. Теперь перед нами стоит задача превратить эту конструкцию в надёжный, предсказуемый продукт, готовый к изменениям и совместной разработке. Этот блок посвящён систематическому тестированию и отладке всех компонентов.

Вы научитесь писать юнит-тесты для MCP-серверов, изолируя HTTP-зависимости с помощью `msw`, проверять логику клиентов LLM на моках, проводить end‑to‑end испытания оркестратора с реальными эмуляторами и уверенно искать корень проблем с помощью Inspector’а и структурированного логирования. К концу блока вы соберёте собственный CI‑скрипт, который за пару минут проверит весь проект от фундамента до крыши.

---

## Модуль 7.1. Юнит-тестирование MCP-серверов и эмуляторов

### Цели тестирования MCP-серверов

Наши MCP‑серверы «Компас‑3D» и «CarverAll 15Pro» умеют общаться с внешними эмуляторами по HTTP. Главная задача тестов — убедиться, что **каждый инструмент** корректно принимает валидные параметры, отбрасывает невалидные с понятной ошибкой и совершает ожидаемые HTTP‑вызовы к эмулятору. Мы не хотим гонять настоящий эмулятор ради проверки одного обработчика, поэтому HTTP‑зависимости будем заменять моками.

Сервер MCP общается с клиентом по протоколу JSON‑RPC через транспорт (у нас `stdio`). Чтобы протестировать инструменты в условиях, максимально приближенных к реальным, мы запустим и сервер, и клиент **в одном Node.js‑процессе**, соединив их парой потоков. Исходящие HTTP‑запросы сервера перехватывает `msw` — библиотека Mock Service Worker, работающая на уровне нативного модуля `http`.

Для Fastify‑эмуляторов мы напишем классические юнит‑тесты эндпоинтов, проверяя бизнес‑логику напрямую, без подъёма TCP‑сокетов.

### Настройка тестового окружения

Установите зависимости в корне каждого MCP‑проекта (`kompas3d-mcp`, `carverall-mcp`):

```bash
npm install -D vitest msw @types/node
```

Добавьте в `package.json` секцию для тестов:

```json
{
  "scripts": {
    "test": "vitest run",
    "test:watch": "vitest"
  },
  "type": "module"
}
```

Создайте файл `vitest.config.ts`:

```typescript
import { defineConfig } from 'vitest/config';

export default defineConfig({
  test: {
    globals: true,        // чтобы не импортировать describe/it каждый раз
    environment: 'node',  // окружение Node.js
  },
});
```

Мы будем активно использовать `msw`. В тестах потребуется запустить перехватчик до начала работы MCP‑сервера. Шаблон включения `msw` в Node‑окружении выглядит так:

```typescript
import { setupServer } from 'msw/node';
import { http, HttpResponse } from 'msw';

const handlers = [
  http.post('http://localhost:3001/api/part', () => {
    return HttpResponse.json({ documentId: 'doc-123' });
  }),
];

const server = setupServer(...handlers);

beforeAll(() => server.listen({ onUnhandledRequest: 'error' }));
afterEach(() => server.resetHandlers());
afterAll(() => server.close());
```

Если сервер совершит запрос, не попавший ни в один обработчик, `onUnhandledRequest: 'error'` уронит тест — это лучшая страховка от утечек реального трафика.

### Создание тестового транспорта для MCP

В тестах нам нужно запустить MCP‑сервер с `StdioServerTransport` и подключиться к нему клиентом `StdioClientTransport` внутри одного процесса. Сделаем это через пару `PassThrough`‑стримов:

```typescript
import { PassThrough } from 'node:stream';
import { StdioClientTransport, StdioServerTransport } from '@modelcontextprotocol/sdk/client/stdio.js';
// ... импорты сервера

export function createTestTransportPair(): {
  clientTransport: StdioClientTransport;
  serverTransport: StdioServerTransport;
} {
  const clientToServer = new PassThrough();
  const serverToClient = new PassThrough();

  const clientTransport = new StdioClientTransport({
    input: serverToClient,   // то, что сервер пишет, читает клиент
    output: clientToServer,  // то, что клиент пишет, читает сервер
  });

  const serverTransport = new StdioServerTransport({
    input: clientToServer,
    output: serverToClient,
  });

  return { clientTransport, serverTransport };
}
```

Теперь мы можем запустить сервер, передав ему `serverTransport`, а клиент использовать для вызовов инструментов через `clientTransport`.

### Тестирование MCP-сервера «Компас‑3D»

Предположим, что сервер экспортирует асинхронную функцию `startServer(transport)`, которая создаёт экземпляр `McpServer` и подключает транспорт. В реальном коде вы, вероятно, получаете сервер через вызов фабрики. Адаптируйте пример под свою архитектуру.

#### Тест инструмента `create_part`

```typescript
import { describe, it, expect, beforeAll, afterAll, afterEach } from 'vitest';
import { Client } from '@modelcontextprotocol/sdk/client/index.js';
import { setupServer } from 'msw/node';
import { http, HttpResponse } from 'msw';
import { createTestTransportPair } from './test-transport.js';
import { startServer } from '../src/index.js'; // условный экспорт

const mswServer = setupServer(
  http.post('http://localhost:3001/api/part', async ({ request }) => {
    const body = await request.json();
    // Проверим, что в теле пришли нужные поля
    expect(body).toMatchObject({ name: 'Деталь_1', type: 'part' });
    return HttpResponse.json({ documentId: 'part-001', status: 'created' });
  })
);

describe('kompas3d-mcp: create_part', () => {
  let client: Client;
  let cleanup: () => Promise<void>;

  beforeAll(async () => {
    mswServer.listen({ onUnhandledRequest: 'error' });

    const { clientTransport, serverTransport } = createTestTransportPair();
    // Запускаем сервер в фоне
    const serverPromise = startServer(serverTransport);
    // Клиент
    client = new Client(
      { name: 'test-client', version: '0.0.0' },
      { capabilities: {} }
    );
    await client.connect(clientTransport);

    cleanup = async () => {
      await client.close();
      // Даём серверу корректно завершиться (в реальном коде нужно предусмотреть сигнал)
      // Для простоты можно принудительно закрыть стримы
      clientTransport.close();
      serverTransport.close();
    };
  });

  afterAll(async () => {
    await cleanup();
    mswServer.close();
  });

  afterEach(() => {
    mswServer.resetHandlers();
  });

  it('возвращает documentId и делает правильный HTTP-вызов', async () => {
    const result = await client.callTool({
      name: 'create_part',
      arguments: { name: 'Деталь_1', type: 'part' },
    });

    expect(result).toBeDefined();
    // Результат — массив content, в котором текст с JSON
    const textContent = result.content.find((c) => c.type === 'text');
    const parsed = JSON.parse(textContent.text);
    expect(parsed).toMatchObject({ documentId: 'part-001', status: 'created' });
  });

  it('выбрасывает ошибку при пустом имени детали', async () => {
    await expect(
      client.callTool({
        name: 'create_part',
        arguments: { name: '', type: 'part' },
      })
    ).rejects.toThrow(/имя детали не может быть пустым/i);
  });
});
```

Пояснения:
- `msw` перехватывает POST‑запросы, отправленные сервером к эмулятору.
- Мы проверяем не только ответ инструмента, но и корректность тела запроса (через `expect` внутри обработчика). Если формат не совпадёт, тест упадёт.
- Для теста с пустым именем детали предполагается, что на сервере стоит валидация через Zod, которая бросает исключение, превращая его в MCP‑ошибку. Клиент в этом случае выбрасывает ошибку, которую мы ловим.

#### Тест цепочки: создать документ → эскиз → выдавить → экспортировать

Здесь мы последовательно вызываем несколько инструментов и проверяем, что HTTP‑запросы к эмулятору идут в правильном порядке и с нужными параметрами. Используем `msw` для отслеживания вызовов:

```typescript
it('полный цикл создания детали', async () => {
  const calls: string[] = [];
  mswServer.use(
    http.post('http://localhost:3001/api/part', () => {
      calls.push('create_part');
      return HttpResponse.json({ documentId: 'doc-chain' });
    }),
    http.post('http://localhost:3001/api/doc-chain/sketch', async ({ request }) => {
      calls.push('create_sketch');
      return HttpResponse.json({ sketchId: 'sketch-1' });
    }),
    http.post('http://localhost:3001/api/doc-chain/extrude', async ({ request }) => {
      calls.push('extrude');
      return HttpResponse.json({ status: 'extruded' });
    }),
    http.post('http://localhost:3001/api/doc-chain/export', async ({ request }) => {
      calls.push('export');
      return HttpResponse.json({ filePath: '/tmp/model.step' });
    })
  );

  await client.callTool({ name: 'create_part', arguments: { name: 'цикл' } });
  await client.callTool({ name: 'create_sketch', arguments: { documentId: 'doc-chain', plane: 'XY' } });
  await client.callTool({ name: 'extrude', arguments: { documentId: 'doc-chain', sketchId: 'sketch-1', depth: 10 } });
  const exportResult = await client.callTool({ name: 'export', arguments: { documentId: 'doc-chain', format: 'step' } });

  expect(calls).toEqual(['create_part', 'create_sketch', 'extrude', 'export']);
  const text = JSON.parse(exportResult.content[0].text);
  expect(text.filePath).toBe('/tmp/model.step');
});
```

### Тестирование MCP-сервера «CarverAll 15Pro»

Структура тестов аналогична. Отдельного внимания заслуживает проверка ресурса `job://{jobId}/status`, который отдаёт динамически изменяющиеся данные.

```typescript
it('ресурс job://{jobId}/status возвращает статус после эмуляции прогресса', async () => {
  mswServer.use(
    http.post('http://localhost:3002/api/jobs', () => {
      return HttpResponse.json({ jobId: 'job-42', status: 'QUEUED' });
    }),
    http.get('http://localhost:3002/api/jobs/job-42/status', () => {
      return HttpResponse.json({ status: 'RUNNING', progress: 50 });
    })
  );

  await client.callTool({ name: 'upload_job', arguments: { filePath: '/tmp/test.svg', material: 'plywood_3mm' } });

  // Читаем ресурс
  const resourceResult = await client.readResource({ uri: 'job://job-42/status' });
  const statusText = resourceResult.contents[0].text;
  const status = JSON.parse(statusText);
  expect(status).toMatchObject({ status: 'RUNNING', progress: 50 });
});
```

### Юнит-тесты эмуляторов на Fastify

Эмуляторы — это обычные REST‑серверы. Для них мы пишем тесты без поднятия сетевого сокета, используя встроенный в Fastify метод `inject` (или `light-my-request`):

```typescript
import Fastify from 'fastify';
import { describe, it, expect, beforeAll, afterAll } from 'vitest';
import { buildApp } from '../src/app.js'; // функция, создающая экземпляр Fastify с маршрутами

let app: Fastify.FastifyInstance;

beforeAll(async () => {
  app = await buildApp();
  await app.ready();
});

afterAll(async () => {
  await app.close();
});

describe('Эмулятор гравировщика', () => {
  it('создаёт задание и возвращает статус QUEUED', async () => {
    const response = await app.inject({
      method: 'POST',
      url: '/api/jobs',
      payload: { fileUrl: 'http://files/test.svg', material: 'wood' },
    });
    expect(response.statusCode).toBe(201);
    const body = response.json();
    expect(body).toHaveProperty('jobId');
    expect(body.status).toBe('QUEUED');
  });

  it('переводит статус в COMPLETED после эмуляции', async () => {
    // Создаём задание
    const createRes = await app.inject({ method: 'POST', url: '/api/jobs', payload: {} });
    const { jobId } = createRes.json();

    // Форсируем смену статуса (в эмуляторе есть внутренний механизм)
    // Вызываем эндпоинт запуска
    await app.inject({ method: 'POST', url: `/api/jobs/${jobId}/start` });
    // Ждём завершения эмуляции (можно опросить статус)
    let status = '';
    for (let i = 0; i < 10; i++) {
      const s = await app.inject({ method: 'GET', url: `/api/jobs/${jobId}/status` });
      status = s.json().status;
      if (status === 'COMPLETED') break;
      await new Promise((r) => setTimeout(r, 50));
    }
    expect(status).toBe('COMPLETED');
  });
});
```

### Типичные ошибки и их решение

| Проблема | Причина и решение |
|----------|-------------------|
| `msw` не перехватывает запросы, тест стучится в настоящий эмулятор | Убедитесь, что `server.listen()` вызван до старта MCP‑сервера. Если сервер стартует внутри `beforeAll`, `listen` должен быть выше. |
| Ошибка `onUnhandledRequest` из-за непойманного запроса | Добавьте недостающий обработчик или временно используйте `onUnhandledRequest: 'bypass'` в `listen()` для отладки. |
| `TypeError: body.stream is not a function` при использовании `light-my-request` с новыми версиями Fastify | Проверьте версию Fastify; используйте `app.inject`, он совместим. Если используете `light-my-request` напрямую, передавайте `payload` как строку или объект. |
| Клиент MCP выбрасывает «MCP error -32603» при валидации параметров | Сервер должен оборачивать ошибки Zod в `McpError`. Убедитесь, что в коде сервера для каждого инструмента стоит `try/catch` и при ошибке валидации выбрасывается `new McpError(ErrorCode.InvalidParams, e.message)`. |
| Тест падает с таймаутом, сервер не отвечает | Возможно, сервер ожидает закрытия стримов. В тестовом транспорте используйте `PassThrough` и после теста вызывайте `clientTransport.close()` и `serverTransport.close()`. |

### Вопросы для самопроверки

1. Почему для тестирования инструментов MCP мы создаём пару стримов вместо запуска дочернего процесса?
2. Какие преимущества даёт использование `msw` по сравнению с ручной подменой `fetch` через `vi.mock`?
3. Как проверить, что HTTP-запрос к эмулятору был отправлен с правильным телом?
4. Для чего нужна опция `onUnhandledRequest: 'error'` при запуске `msw`?
5. В чём разница между `app.inject` и поднятием реального сервера на порту?

### Практическое задание

Добавьте в проект `carverall-mcp` тест для инструмента `set_laser_params`:
- Проверьте успешную установку параметров (мощность 80, скорость 120).
- Проверьте, что при отрицательной мощности возвращается ошибка валидации.
- Убедитесь, что HTTP-запрос к эмулятору содержит корректные поля.

---

## Модуль 7.2. Тестирование Tool Calling и LLM-интеграции

### Особенности тестирования компонентов, зависящих от LLM

LLM‑провайдеры (DeepSeek, YandexGPT) возвращают недетерминированные тексты. Мы не можем жёстко проверять содержимое ответа, но можем и должны проверять **поведение клиента**:

- Правильно ли формируется тело запроса (сообщения, инструменты, параметры)?
- Корректно ли парсится ответ: извлекаются текстовые сообщения, распознаются `tool_calls` (единичные и множественные), не теряются аргументы?
- Преобразуются ли ошибки HTTP и API в ожидаемые исключения?

Для этого мы перехватываем HTTP‑запросы с помощью `msw` и подсовываем заранее заготовленные JSON‑ответы, имитирующие реальное API. Так мы изолируем клиент от живой модели и делаем тесты быстрыми и предсказуемыми.

### Тесты для DeepSeekClient

Предположим, что `DeepSeekClient` реализует интерфейс `LLMProvider` с методом `generate(messages, options)`. Напишем тесты.

#### Мокирование ответа с простым текстом

```typescript
import { describe, it, expect, beforeAll, afterAll } from 'vitest';
import { setupServer } from 'msw/node';
import { http, HttpResponse } from 'msw';
import { DeepSeekClient } from '../src/providers/deepseek-client.js';

const mockServer = setupServer(
  http.post('https://api.deepseek.com/v1/chat/completions', () => {
    return HttpResponse.json({
      id: 'chatcmpl-123',
      object: 'chat.completion',
      choices: [{
        index: 0,
        message: {
          role: 'assistant',
          content: 'Привет! Чем могу помочь?',
        },
        finish_reason: 'stop',
      }],
    });
  })
);

describe('DeepSeekClient', () => {
  beforeAll(() => mockServer.listen({ onUnhandledRequest: 'error' }));
  afterAll(() => mockServer.close());

  it('возвращает текстовый ответ', async () => {
    const client = new DeepSeekClient({ apiKey: 'test-key' });
    const result = await client.generate([{ role: 'user', content: 'Привет' }]);

    expect(result).toHaveProperty('text');
    expect(result.text).toBe('Привет! Чем могу помочь?');
    expect(result.toolCalls).toBeUndefined();
  });
});
```

#### Мокирование ответа с одним `tool_call`

```typescript
it('распознаёт одиночный tool_call', async () => {
  mockServer.use(
    http.post('https://api.deepseek.com/v1/chat/completions', () => {
      return HttpResponse.json({
        choices: [{
          message: {
            role: 'assistant',
            content: null,
            tool_calls: [{
              id: 'call_1',
              type: 'function',
              function: {
                name: 'get_weather',
                arguments: '{"location":"Moscow"}',
              },
            }],
          },
          finish_reason: 'tool_calls',
        }],
      });
    })
  );

  const client = new DeepSeekClient({ apiKey: 'test-key' });
  const result = await client.generate([{ role: 'user', content: 'Какая погода в Москве?' }], {
    tools: [{ type: 'function', function: { name: 'get_weather', parameters: {} } }],
  });

  expect(result.text).toBeUndefined();
  expect(result.toolCalls).toHaveLength(1);
  expect(result.toolCalls![0]).toMatchObject({
    id: 'call_1',
    name: 'get_weather',
    arguments: { location: 'Moscow' }, // клиент парсит JSON-строку
  });
});
```

#### Мокирование параллельных `tool_calls`

DeepSeek может возвращать массив из нескольких `tool_calls`. Убедимся, что клиент правильно их парсит:

```typescript
it('обрабатывает несколько параллельных tool_calls', async () => {
  mockServer.use(
    http.post('https://api.deepseek.com/v1/chat/completions', () => {
      return HttpResponse.json({
        choices: [{
          message: {
            role: 'assistant',
            tool_calls: [
              { id: 'c1', type: 'function', function: { name: 'add', arguments: '{"a":1,"b":2}' } },
              { id: 'c2', type: 'function', function: { name: 'multiply', arguments: '{"x":3,"y":4}' } },
            ],
          },
          finish_reason: 'tool_calls',
        }],
      });
    })
  );

  const result = await client.generate([{ role: 'user', content: 'Посчитай' }]);
  expect(result.toolCalls).toHaveLength(2);
  expect(result.toolCalls![0].name).toBe('add');
  expect(result.toolCalls![1].arguments).toEqual({ x: 3, y: 4 });
});
```

#### Мокирование ошибки аутентификации (401)

```typescript
it('выбрасывает исключение при 401', async () => {
  mockServer.use(
    http.post('https://api.deepseek.com/v1/chat/completions', () => {
      return new HttpResponse('Unauthorized', { status: 401 });
    })
  );

  const client = new DeepSeekClient({ apiKey: 'bad-key' });
  await expect(client.generate([{ role: 'user', content: '...' }]))
    .rejects.toThrow(/401|Unauthorized/);
});
```

### Тесты для YandexGPTClient

Принцип тот же, только меняется URL и формат ответа. Пример для YandexGPT API:

```typescript
it('парсит tool_calls из YandexGPT ответа', async () => {
  mockServer.use(
    http.post('https://llm.api.cloud.yandex.net/foundationModels/v1/completion', () => {
      return HttpResponse.json({
        result: {
          alternatives: [{
            message: {
              role: 'assistant',
              text: '', // может быть пустым
              tool_calls: [{
                id: 'tc1',
                function: {
                  name: 'search',
                  arguments: { query: 'погода' }, // уже объект
                },
              }],
            },
          }],
        },
      });
    })
  );

  const client = new YandexGPTClient({ apiKey: '...', folderId: '...' });
  const result = await client.generate([{ role: 'user', content: 'Найди погоду' }]);
  expect(result.toolCalls![0].arguments).toEqual({ query: 'погода' });
});
```

Убедитесь, что клиент YandexGPT правильно обрабатывает как объект `arguments`, так и JSON‑строку (зависит от версии API).

### Интеграционный тест: ToolCallingClient + MCP-сервер Toolbox

Здесь мы проверим связку «LLM‑клиент → вызов настоящего MCP‑инструмента». Используем `msw` для подмены ответа LLM, а MCP‑сервер Toolbox запустим в том же процессе через парные стримы (как в модуле 7.1).

Предположим, Toolbox‑сервер имеет инструмент `add`, который складывает два числа и возвращает результат.

```typescript
import { Client } from '@modelcontextprotocol/sdk/client/index.js';
import { createTestTransportPair } from './test-transport.js';
import { startToolboxServer } from '../src/toolbox-server.js'; // старт сервера Toolbox

it('ToolCallingClient выполняет tool_call на Toolbox и получает результат', async () => {
  // Поднимаем Toolbox MCP сервер
  const { clientTransport, serverTransport } = createTestTransportPair();
  const serverPromise = startToolboxServer(serverTransport);
  const mcpClient = new Client({ name: 'test', version: '0' }, { capabilities: {} });
  await mcpClient.connect(clientTransport);

  // Мокаем LLM: ответ с tool_call "add"
  mockServer.use(
    http.post('https://api.deepseek.com/v1/chat/completions', () => {
      return HttpResponse.json({
        choices: [{
          message: {
            role: 'assistant',
            tool_calls: [{
              id: 'call_add',
              type: 'function',
              function: { name: 'add', arguments: '{"a":5,"b":7}' },
            }],
          },
          finish_reason: 'tool_calls',
        }],
      });
    })
  );

  // Создаём ToolCallingClient, передаём LLM-клиент и MCP-клиент
  const toolCallingClient = new ToolCallingClient({
    llm: new DeepSeekClient({ apiKey: 'test' }),
    mcpClients: [mcpClient],
  });

  const conversation = await toolCallingClient.processMessage('Сложи 5 и 7');
  // Ожидаем, что в истории появится результат выполнения инструмента
  const assistantResult = conversation.find(m => m.role === 'tool');
  expect(assistantResult).toBeDefined();
  const parsed = JSON.parse(assistantResult!.content);
  expect(parsed).toBe(12); // 5+7
});
```

Такой тест подтверждает, что весь пайплайн — от распознавания намерения LLM до исполнения инструмента и возврата результата — работает корректно.

### Типичные ошибки и их решение

| Проблема | Причина и решение |
|----------|-------------------|
| `tool_calls` не парсятся, возвращается пустой массив | Проверьте формат ответа в моке: для DeepSeek `arguments` — это JSON‑строка; для YandexGPT может быть объект. Убедитесь, что клиент обрабатывает оба варианта. |
| Тест ожидает `toolCalls`, а получает текст | LLM может иногда отвечать текстом вместо вызова инструмента. В боевом коде это нормально, но в тесте мок должен возвращать `finish_reason: 'tool_calls'`. |
| Ошибка `TypeError: Body is unusable` при чтении тела запроса в моке | `msw` позволяет читать `request.json()` только один раз. Если вы уже прочитали тело в логировании, сохраните его в переменную. |
| Мок не срабатывает для YandexGPT из-за другого домена | Убедитесь, что URL в `http.post` полностью совпадает с тем, который использует клиент (включая `/foundationModels/v1/completion`). |
| `ToolCallingClient` не отправляет результат обратно в историю | Проверьте реализацию цикла: после выполнения инструмента результат должен быть добавлен как сообщение с `role: 'tool'` и `tool_call_id`. |

### Вопросы для самопроверки

1. Почему мы тестируем LLM‑клиент, не обращаясь к настоящей модели?
2. Какие поля обязательно должны присутствовать в мокированном ответе для корректной работы клиента?
3. Зачем в интеграционном тесте Toolbox мы запускаем настоящий MCP‑сервер, а не мокаем вызов инструмента?
4. Как убедиться, что клиент правильно обрабатывает ситуацию, когда модель возвращает одновременно и текст, и `tool_calls` (если такое возможно)?

### Практическое задание

Напишите тест для `ToolCallingClient`, который проверяет обработку ошибки от MCP‑инструмента. Замокайте LLM‑ответ, вызывающий заведомо падающий инструмент (например, деление на ноль). Убедитесь, что `ToolCallingClient` корректно передаёт сообщение об ошибке обратно модели и не падает.

---

## Модуль 7.3. E2E-тестирование оркестратора и приёмы отладки

### Зачем нужны end‑to‑end тесты оркестратора?

Оркестратор на базе OpenCode — это связующее звено, которое по команде пользователя принимает решение, какие MCP‑инструменты вызвать, и делает это через LLM. Проверить такую цепочку изолированными юнит‑тестами невозможно. E2E‑тесты запускают **реальные** процессы эмуляторов, MCP‑серверов и сам OpenCode CLI, подают пользовательскую команду и анализируют результат: созданные файлы, статусы в эмуляторах, записи в логах.

### Инструментарий

- **vitest** или **node:test** — запуск тестов и управление дочерними процессами.
- **execa** или `child_process.spawn` — удобный запуск CLI‑команд.
- **wait-for-expect** — поллинг состояния (ожидание файла или статуса).
- Эмуляторы запускаются как отдельные процессы на известных портах.
- OpenCode запускается с флагом `--log-level DEBUG` и `--print-logs`, чтобы видеть подробные логи в stderr.

### Написание E2E‑теста для сценария «Куб 10×10×10 мм в STEP»

Создадим тестовый файл `e2e/kompas-cube.test.ts`.

```typescript
import { describe, it, expect, beforeAll, afterAll } from 'vitest';
import { spawn } from 'node:child_process';
import { setTimeout } from 'node:timers/promises';
import { existsSync, unlinkSync, openSync } from 'node:fs';
import path from 'node:path';

const EMULATOR_PORT = '3001';
const OPENCODE_BIN = './node_modules/.bin/opencode'; // или путь к глобальной установке

let emulatorProcess: ChildProcess;

beforeAll(async () => {
  // Запускаем эмулятор Компас-3D
  emulatorProcess = spawn('node', ['dist/emulator.js'], {
    env: { ...process.env, PORT: EMULATOR_PORT },
    stdio: 'pipe',
  });
  // Ждём, пока эмулятор начнёт слушать порт
  await waitForServer(`http://localhost:${EMULATOR_PORT}/health`);
}, 15000);

afterAll(() => {
  emulatorProcess?.kill();
  // Подчищаем созданные файлы
  const files = ['/tmp/test_cube.step', '/tmp/test_cube_10.step'];
  files.forEach(f => { if (existsSync(f)) unlinkSync(f); });
});

async function waitForServer(url: string, timeout = 10000) {
  const start = Date.now();
  while (Date.now() - start < timeout) {
    try {
      await fetch(url);
      return;
    } catch { /* ждём */ }
    await setTimeout(500);
  }
  throw new Error(`Server ${url} did not start`);
}

it('создаёт куб 10x10x10 и сохраняет STEP-файл', async () => {
  // Запускаем OpenCode с конфигурацией, в которой прописаны пути к скомпилированным MCP-серверам.
  // Предположим, конфиг лежит в .opencode/kompas3d.yml и уже указывает на наш эмулятор через переменные окружения.
  const opencodeProcess = spawn(
    OPENCODE_BIN,
    ['run', '--config', '.opencode/kompas3d.yml', '--log-level DEBUG', '--print-logs'],
    {
      env: { ...process.env, KOMPAS_EMULATOR_URL: `http://localhost:${EMULATOR_PORT}` },
      stdio: ['pipe', 'pipe', fs.openSync('./e2e-logs/kompas_cube.log', 'w')],
    }
  );

  // Подаём команду в stdin
  opencodeProcess.stdin.write('Создай куб 10×10×10 мм и сохрани в step\n');
  opencodeProcess.stdin.end();

  let stdout = '';
  opencodeProcess.stdout.on('data', (chunk) => { stdout += chunk; });

  // Ожидаем завершения процесса с таймаутом
  const exitCode = await new Promise<number>((resolve) => {
    opencodeProcess.on('close', resolve);
    // Таймаут 60 секунд
    setTimeout(60000).then(() => {
      opencodeProcess.kill();
      resolve(-1);
    });
  });

  expect(exitCode).toBe(0);

  // Проверяем, что файл создан (допустим, оркестратор сохраняет в /tmp с именем, содержащим "cube" и "step")
  const expectedFile = '/tmp/test_cube.step'; // имя должно генерироваться из запроса
  expect(existsSync(expectedFile)).toBe(true);

  // Анализируем лог-файл на наличие вызовов инструментов
  const logContent = require('fs').readFileSync('./e2e-logs/kompas_cube.log', 'utf-8');
  expect(logContent).toMatch(/create_part/);
  expect(logContent).toMatch(/create_sketch/);
  expect(logContent).toMatch(/extrude/);
  expect(logContent).toMatch(/export/);
}, 60000);
```

### Сценарий с гравировщиком

Аналогично запускаем эмулятор гравировщика на порту 3002, подаём команду «Загрузи файл test.svg для фанеры 3 мм, запусти гравировку» и проверяем через API эмулятора, что статус задания стал `COMPLETED`.

```typescript
it('выполняет гравировку тестового файла', async () => {
  // Команда в OpenCode
  opencodeProcess.stdin.write('Загрузи файл test.svg для фанеры 3 мм, запусти гравировку\n');
  // ... после завершения опрашиваем эмулятор
  const statusResp = await fetch(`http://localhost:3002/api/jobs/last/status`);
  const { status } = await statusResp.json();
  expect(status).toBe('COMPLETED');
});
```

### Приёмы отладки MCP-взаимодействий

Даже при идеальных тестах в реальной разработке приходится «чинить» транспорт, таймауты и ошибки валидации. Вот набор инструментов и практик, которые сэкономят вам часы.

#### 1. MCP Inspector

Интерактивный веб‑интерфейс для ручного тестирования MCP‑серверов. Установите глобально:

```bash
npm install -g @anthropic-ai/mcp-inspector
```

Запустите сервер в режиме stdio:

```bash
npx @anthropic-ai/mcp-inspector node dist/index.js
```

Инспектор поднимет локальный сайт, где вы сможете выбрать инструмент, заполнить параметры, выполнить вызов и увидеть сырой JSON‑RPC трафик.

#### 2. Структурированное логирование

Добавьте на сервер логирование каждого входящего запроса и ответа. Используйте `pino`:

```typescript
import pino from 'pino';
const logger = pino({ level: 'debug' });

// В обработчике инструмента:
server.setRequestHandler(CallToolRequestSchema, async (request) => {
  logger.info({ method: 'tools/call', params: request.params }, 'Incoming tool call');
  try {
    const result = await handleTool(request.params.name, request.params.arguments);
    logger.info({ result }, 'Tool call succeeded');
    return { content: [{ type: 'text', text: JSON.stringify(result) }] };
  } catch (err) {
    logger.error({ err, params: request.params }, 'Tool call failed');
    throw err;
  }
});
```

При запуске через `opencode` направляйте вывод сервера в файл:

```bash
node dist/server.js 2>mcpserver.log
```

#### 3. Просмотр JSON‑RPC трафика через `tee`

Если сервер работает через stdio, вы можете вклиниться в потоки, чтобы увидеть, что летит по трубе. В режиме разработки оберните вызов сервера в скрипт‑прокладку:

```bash
#!/bin/bash
# mcp-wrapper.sh
tee /dev/stderr | node dist/server.js 2>stderr.log | tee /dev/stderr
```

Это выведет на консоль (stderr) всё, что сервер получает и отправляет.

#### 4. Ручное тестирование инструмента через stdin

Можно напрямую отправить JSON‑RPC запрос в процесс сервера:

```bash
echo '{"jsonrpc":"2.0","id":1,"method":"tools/call","params":{"name":"create_part","arguments":{"name":"test"}}}' | node dist/index.js
```

Сервер ответит тем же JSON‑RPC сообщением на stdout. Удобно для быстрой проверки без Inspector’а.

### Типичные ошибки при E2E и отладке

| Проблема | Причина и решение |
|----------|-------------------|
| OpenCode не видит созданный файл | Проверьте рабочую директорию (`cwd`) при запуске процесса. Если файл сохраняется относительно неё, тест должен искать там же. Задайте `cwd` явно. |
| Тест завершается по таймауту, оркестратор «зависает» | Увеличьте таймаут, проверьте, что эмуляторы отвечают. Включите `--log-level DEBUG` и изучите, на каком шаге остановился OpenCode. Возможно, инструмент вернул ошибку, и LLM ушла в бесконечный цикл. |
| `msw` перехватывает запросы от эмулятора, тест падает | В E2E тестах не используйте `msw` — эмулятор должен быть настоящим. Если `msw` слушает в фоне от других тестов, убедитесь, что E2E‑процессы не наследуют его настройки (запускайте в отдельном потоке). |
| MCP Inspector показывает ошибку «Transport closed» | Убедитесь, что ваш сервер не завершается сразу после запуска. Добавьте в конце `server.ts` код, удерживающий процесс: `process.stdin.resume()` или бесконечный таймер. |
| Логирование через `pino` не попадает в файл | По умолчанию `pino` пишет в stdout. Используйте `pino.destination(2)` для stderr или направьте весь вывод процесса. |

### Вопросы для самопроверки

1. Чем E2E‑тест оркестратора отличается от интеграционного теста ToolCallingClient?
2. Какие метрики успеха мы проверяем в сценарии с кубом?
3. Зачем нужен поллинг готовности эмулятора перед запуском OpenCode?
4. Как с помощью MCP Inspector проверить ресурс `job://{jobId}/status`?
5. Опишите последовательность действий при отладке ошибки «MCP‑сервер не отвечает на tool_call».

### Практическое задание

Реализуйте E2E‑тест для сценария «Создать чертёж детали по эскизу» (инструмент `create_drawing`). Проверьте, что оркестратор вызывает нужные инструменты, а в логах появляется путь к сгенерированному PDF. Напишите скрипт на Bash, который запускает оба эмулятора, выполняет тест и гасит процессы.

---

## Итоговое домашнее задание: CI‑скрипт

Финальный аккорд — соберите всё вместе в скрипте непрерывной интеграции, который можно запустить одной командой на чистом окружении.

### Требования

Скрипт (на Shell или GitHub Actions) должен:

1. Установить зависимости (`npm ci`) во всех проектах: корень, `kompas3d-mcp`, `carverall-mcp`, `toolbox-mcp`, `toolcalling-client`.
2. Собрать TypeScript (`npm run build` где необходимо).
3. Запустить юнит‑тесты MCP‑серверов (`npm test`).
4. Запустить юнит‑тесты клиентов и интеграционный тест ToolCallingClient.
5. Запустить E2E‑тест оркестратора:
   - Стартовать эмуляторы в фоне.
   - Дождаться их готовности.
   - Выполнить `npx vitest run e2e/`.
   - Погасить эмуляторы.
6. Сохранить отчёт о прохождении тестов и логи E2E как артефакты (если CI‑система позволяет).

### Пример скрипта `ci.sh`

```bash
#!/bin/bash
set -e

echo "=== Установка зависимостей ==="
cd kompas3d-mcp && npm ci && cd ..
cd carverall-mcp && npm ci && cd ..
cd toolbox-mcp && npm ci && cd ..
cd toolcalling-client && npm ci && cd ..
cd orchestrator && npm ci && cd ..

echo "=== Сборка ==="
cd kompas3d-mcp && npm run build && cd ..
cd carverall-mcp && npm run build && cd ..
cd toolbox-mcp && npm run build && cd ..

echo "=== Юнит-тесты MCP серверов ==="
cd kompas3d-mcp && npm test && cd ..
cd carverall-mcp && npm test && cd ..

echo "=== Тесты ToolCalling и интеграция ==="
cd toolcalling-client && npm test && cd ..

echo "=== Запуск эмуляторов ==="
node kompas3d-mcp/dist/emulator.js &
EMULATOR1_PID=$!
node carverall-mcp/dist/emulator.js &
EMULATOR2_PID=$!
sleep 2 # дать время на старт

echo "=== E2E тесты оркестратора ==="
cd orchestrator && npx vitest run e2e/ && cd ..

echo "=== Остановка эмуляторов ==="
kill $EMULATOR1_PID $EMULATOR2_PID
echo "Все тесты пройдены!"
```

Для GitHub Actions создайте `.github/workflows/ci.yml` на основе приведённого скрипта, добавив шаги `actions/checkout`, `actions/setup-node` и выгрузку артефактов.

### В отчёте

Приложите полный лог выполнения CI‑скрипта, подтверждающий, что все тесты проходят успешно. Опишите любые встретившиеся проблемы и как вы их решили.

---

## Ссылки на документацию

- [Vitest](https://vitest.dev/) — фреймворк для тестирования.
- [Mock Service Worker (MSW) для Node.js](https://mswjs.io/docs/getting-started/integrate/node) — перехват HTTP в тестах.
- [Тестирование Fastify](https://fastify.dev/docs/latest/Guides/Testing/) — раздел про `inject`.
- [MCP Inspector](https://www.anthropic.com/news/mcp-inspector) — интерактивная отладка серверов.
- [OpenCode CLI](https://github.com/opencode-ai/opencode) — репозиторий и документация оркестратора.
- [Pino](https://getpino.io/) — структурированный логгер для Node.js.

---

Поздравляем! Вы прошли все этапы построения надёжной агентной системы на практике. Теперь в вашем арсенале — системный подход к тестированию, умение изолировать LLM‑зависимости и отлаживать сложные MCP‑взаимодействия. Смело применяйте эти техники в реальных проектах и помните: хорошие тесты — залог спокойного сна разработчика.