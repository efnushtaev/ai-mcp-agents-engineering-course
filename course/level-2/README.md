# Блок 2. Единый интерфейс для Tool Calling с open-source моделями

**Глобальная цель** — создать универсальный TypeScript-клиент `ToolCallingClient`, который предоставляет единый интерфейс для генерации текста с вызовом инструментов. Вы скроете различия между DeepSeek API и YandexGPT API за общей абстракцией и покроете всё надёжными unit-тестами с mock-серверами.

По итогам блока вы получите слой, готовый к построению агентов, MCP-клиентов и оркестраторов из следующих модулей.

---

## Модуль 2.1. Проектирование универсального интерфейса

### Теория

#### Зачем нужна абстракция

Прямая интеграция с конкретным LLM-провайдером в кодовой базе порождает три проблемы:

1. **Вендор-лок (vendor lock-in).** Замена DeepSeek на YandexGPT или любую совместимую модель требует переписывания всех мест, где вызывается API.
2. **Сложность тестирования.** Реальные HTTP-запросы делают тесты медленными, недетерминированными и зависящими от сети/квот.
3. **Разрастание дублирования.** Без общего контракта для каждого провайдера пишутся собственные хелперы, конвертеры и проверки ошибок.

Универсальный интерфейс решает эти задачи, пряча специфику за фасадом и позволяя переключаться между провайдерами изменением одной строки конфигурации.

#### Принципы SOLID в AI-клиентах

Из пяти принципов нас особенно интересуют два:

- **Dependency Inversion (DIP)** — модули верхнего уровня (агенты) не должны зависеть от низкоуровневых реализаций (DeepSeekClient, YandexGPTClient). Оба уровня зависят от абстракции (`IToolCallingClient`).
- **Interface Segregation (ISP)** — клиент не должен знать о методах, которые ему не нужны. Мы разделим «сгенерировать ответ» и «выполнить инструмент»; последнее оставим за вызывающей стороной.

#### Общий контракт

Наш клиент должен уметь:

- Принимать историю сообщений (как в чате) и опциональный список инструментов.
- Возвращать либо **текстовый ответ**, либо **запрос на вызов одного или нескольких инструментов**.
- Быть легко расширяемым под новый провайдер без изменения сигнатуры.

Спроектируем этот контракт в типах.

#### Проектируем типы

```
┌─────────────────────────────────────────────────────────────┐
│                      IToolCallingClient                     │
│  generate(messages, tools?) → Promise<LLMResponse>          │
└─────────────────────────────────────────────────────────────┘
        ↑                              ↑
        │                              │
┌───────┴──────────┐          ┌────────┴─────────┐
│  DeepSeekClient  │          │ YandexGPTClient  │
└──────────────────┘          └──────────────────┘

LLMResponse = { text: string } | { toolCalls: ToolCall[] }

ToolDef {
  name: string
  description: string
  parameters: ZodType  // + автоматическая генерация JSON Schema
}

ToolCall {
  id: string
  name: string
  arguments: Record<string, unknown> // уже распарсенный JSON
}
```

- **ToolDef** хранит Zod-схему для валидации аргументов «на стороне клиента» и одновременно может быть преобразован в JSON Schema для отправки провайдеру.
- **ToolCall** содержит готовые аргументы, извлечённые из ответа модели.
- **LLMResponse** — размеченный union: либо текст, либо вызовы инструментов.

Почему мы отделяем описание инструментов от их выполнения? Потому что в этом слое мы **не запускаем** код инструментов, а только получаем решение модели. Выполнение остаётся за оркестратором. Такой подход упрощает тестирование и позволяет использовать один клиент для разных рантаймов инструментов (CLI, MCP, песочницы).

### Практика

Создадим файлы типов и интерфейса.

#### 1. `tool-calling-client/types.ts`

```typescript
import { z } from "zod";

/**
 * Описание инструмента (функции), передаваемого модели.
 * Zod-схема parameters одновременно служит для валидации и для генерации JSON Schema.
 */
export interface ToolDef {
  /** Уникальное имя инструмента (например, "calculator") */
  name: string;
  /** Человекочитаемое описание, помогающее модели выбрать инструмент */
  description: string;
  /** Zod-схема входных параметров */
  parameters: z.ZodTypeAny;
}

/**
 * Запрос модели на вызов одного инструмента.
 */
export interface ToolCall {
  /** Уникальный идентификатор вызова в рамках ответа модели */
  id: string;
  /** Имя вызываемого инструмента */
  name: string;
  /** Уже разобранные аргументы (JSON -> объект) */
  arguments: Record<string, unknown>;
}

/**
 * Результат выполнения инструмента, возвращаемый обратно в историю.
 */
export interface ToolResult {
  /** Идентификатор соответствующего ToolCall */
  tool_call_id: string;
  /** Строковое представление результата (обычно JSON или текст) */
  output: string;
}

/**
 * Унифицированное сообщение в истории диалога.
 */
export interface Message {
  role: "user" | "assistant" | "system" | "tool";
  content: string | null;
  /** Опциональные вызовы инструментов, сделанные ассистентом */
  tool_calls?: ToolCall[];
  /** Идентификатор вызова, если это ответ инструмента */
  tool_call_id?: string;
  /** Имя инструмента (для role="tool") */
  name?: string;
}

/**
 * Размеченный ответ модели: либо обычный текст, либо список вызовов инструментов.
 */
export type LLMResponse =
  | { text: string; toolCalls?: undefined }
  | { text?: undefined; toolCalls: ToolCall[] };
```

#### 2. `tool-calling-client/interface.ts`

```typescript
import type { Message, ToolDef, LLMResponse } from "./types";

/**
 * Единый интерфейс для клиентов, генерирующих ответы с возможным вызовом инструментов.
 * Все реализации (DeepSeek, YandexGPT, …) должны удовлетворять этому контракту.
 */
export interface IToolCallingClient {
  /**
   * Отправить запрос к LLM и получить текстовый ответ или запросы на вызов инструментов.
   * @param messages История диалога в формате чата.
   * @param tools Опциональный список доступных инструментов.
   * @returns Унифицированный ответ модели.
   */
  generate(
    messages: Message[],
    tools?: ToolDef[]
  ): Promise<LLMResponse>;
}
```

#### 3. JSDoc-документирование

В примерах выше уже добавлены JSDoc-комментарии. Они помогают IDE показывать подсказки и служат контрактом для всех реализаций.

### Типичные подводные камни

- **Зависимость от формата дат/валют.** JSON Schema, сгенерированная из Zod, может не полностью соответствовать ожиданиям конкретного провайдера. Всегда проверяйте, что `z.number()` превращается в `{"type": "number"}`, а не в нестандартные ключи.
- **Отсутствие поля `tool_calls` у ассистента.** В истории сообщений вы должны уметь хранить сообщения роли `assistant` с `tool_calls: [...]`. Убедитесь, что тип Message это разрешает.
- **Злоупотребление дженериками.** На этапе проектирования не пытайтесь параметризовать `LLMResponse` типом инструментов — это усложнит сигнатуры без реальной пользы.

### Вопросы для самопроверки

1. Какой принцип SOLID требует введения `IToolCallingClient` и почему?
2. Почему `ToolResult` не является частью `LLMResponse`, а вынесен отдельно?
3. В чём преимущество использования Zod в `ToolDef.parameters` вместо сырой JSON Schema?
4. Какую проблему решает размеченный union в `LLMResponse`?

### Практическое задание к модулю 2.1

1. Инициализируйте npm-проект с TypeScript и библиотекой `zod`.
2. Создайте файлы `types.ts` и `interface.ts`, скопировав код из модуля, но дополните `ToolDef` полем `strict?: boolean`.
3. Напишите утилиту `zodToJsonSchema` (можно использовать библиотеку `zod-to-json-schema`) и продемонстрируйте конвертацию одного `ToolDef` в JSON Schema.
4. Добавьте в `interface.ts` вспомогательный тип `ClientConfig` с полями `apiKey` и опциональным `timeoutMs`.

---

## Модуль 2.2. Реализация клиента для DeepSeek

### Теория

DeepSeek предоставляет OpenAI-совместимое API. Это значит, что мы можем использовать официальный JS-клиент `openai` версии 4.x, передав ему кастомный `baseURL` (`https://api.deepseek.com`) и модель `deepseek-chat`.

Ключевые особенности:

- Поддержка параллельных `tool_calls` (модель может запросить вызов нескольких инструментов в одном ответе).
- Параметр `tool_choice` позволяет принудительно задать режим выбора инструментов (`"auto"`, `"none"` или конкретная функция).
- Ответ приходит в формате `response.choices[0].message.tool_calls`, где каждый вызов содержит `id`, `function.name` и `function.arguments` (строка JSON).

Нам нужно преобразовать наш `ToolDef[]` в `ChatCompletionTool[]` и распарсить ответ в наш `ToolCall[]`.

#### Поток данных

```
ToolDef[] ──> map to ChatCompletionTool[] ──┐
                                            ├──> openai.chat.completions.create()
Message[] ──> map to ChatCompletionMessageParam[]
                                            │
LLMResponse <── parse response ─────────────┘
```

### Практика

Установим зависимости:

```bash
npm install openai zod zod-to-json-schema
```

#### Реализация `DeepSeekClient`

```typescript
import OpenAI from "openai";
import type { ChatCompletionMessageParam, ChatCompletionTool } from "openai/resources";
import { zodToJsonSchema } from "zod-to-json-schema";
import type { IToolCallingClient } from "./interface";
import type { Message, ToolDef, ToolCall, LLMResponse } from "./types";

/**
 * Клиент для DeepSeek API, реализующий универсальный интерфейс.
 */
export class DeepSeekClient implements IToolCallingClient {
  private client: OpenAI;
  private model: string;

  /**
   * @param apiKey API-ключ DeepSeek
   * @param baseURL Базовый URL (по умолчанию https://api.deepseek.com)
   * @param model Идентификатор модели (по умолчанию deepseek-chat)
   */
  constructor(
    apiKey: string,
    baseURL = "https://api.deepseek.com",
    model = "deepseek-chat"
  ) {
    this.client = new OpenAI({ apiKey, baseURL });
    this.model = model;
  }

  async generate(messages: Message[], tools?: ToolDef[]): Promise<LLMResponse> {
    const openaiMessages = this.mapMessages(messages);
    const openaiTools = tools?.map(tool => this.mapTool(tool));

    const response = await this.client.chat.completions.create({
      model: this.model,
      messages: openaiMessages,
      tools: openaiTools,
      tool_choice: tools?.length ? "auto" : undefined,
    });

    const choice = response.choices[0];
    if (!choice) {
      throw new Error("No response from DeepSeek");
    }

    const message = choice.message;

    // Если есть tool_calls, возвращаем их
    if (message.tool_calls && message.tool_calls.length > 0) {
      const toolCalls: ToolCall[] = message.tool_calls.map(tc => ({
        id: tc.id,
        name: tc.function.name,
        arguments: this.parseArguments(tc.function.arguments),
      }));
      return { toolCalls };
    }

    // Иначе текстовый ответ
    return { text: message.content ?? "" };
  }

  /** Преобразует внутреннее представление сообщения в формат OpenAI */
  private mapMessages(messages: Message[]): ChatCompletionMessageParam[] {
    return messages.map(msg => {
      switch (msg.role) {
        case "user":
          return { role: "user", content: msg.content ?? "" };
        case "assistant":
          return {
            role: "assistant",
            content: msg.content,
            tool_calls: msg.tool_calls?.map(tc => ({
              id: tc.id,
              type: "function" as const,
              function: {
                name: tc.name,
                arguments: JSON.stringify(tc.arguments),
              },
            })),
          };
        case "system":
          return { role: "system", content: msg.content ?? "" };
        case "tool":
          return {
            role: "tool",
            tool_call_id: msg.tool_call_id!,
            content: msg.output ?? "",
          };
        default:
          throw new Error(`Unsupported message role: ${msg.role}`);
      }
    });
  }

  /** Преобразует ToolDef в формат OpenAI ChatCompletionTool */
  private mapTool(tool: ToolDef): ChatCompletionTool {
    return {
      type: "function",
      function: {
        name: tool.name,
        description: tool.description,
        parameters: zodToJsonSchema(tool.parameters) as Record<string, unknown>,
      },
    };
  }

  /** Безопасный парсинг JSON-аргументов */
  private parseArguments(raw: string): Record<string, unknown> {
    try {
      return JSON.parse(raw);
    } catch {
      throw new Error(`Invalid JSON in tool arguments: ${raw}`);
    }
  }
}
```

#### Ручная проверка

Скрипт `check-deepseek.ts`:

```typescript
import { DeepSeekClient } from "./tool-calling-client/DeepSeekClient";
import type { ToolDef } from "./tool-calling-client/types";
import { z } from "zod";

const client = new DeepSeekClient(process.env.DEEPSEEK_API_KEY!);

const calculatorTool: ToolDef = {
  name: "calculator",
  description: "Выполняет арифметическую операцию",
  parameters: z.object({
    operation: z.enum(["add", "subtract", "multiply", "divide"]),
    a: z.number(),
    b: z.number(),
  }),
};

const converterTool: ToolDef = {
  name: "unit_converter",
  description: "Конвертирует единицы измерения",
  parameters: z.object({
    value: z.number(),
    from: z.string(),
    to: z.string(),
  }),
};

async function main() {
  const response = await client.generate(
    [{ role: "user", content: "Сложи 5 и 7, результат переведи в дюймы" }],
    [calculatorTool, converterTool]
  );

  if (response.toolCalls) {
    console.log("Модель хочет вызвать инструменты:", JSON.stringify(response.toolCalls, null, 2));
  } else {
    console.log("Текстовый ответ:", response.text);
  }
}

main().catch(console.error);
```

### Типичные подводные камни

- **Параллельные вызовы.** Модель может вернуть несколько `tool_calls` в одном ответе. Убедитесь, что ваша логика обработки (оркестратор) умеет выполнять их все, прежде чем отправлять следующий запрос.
- **Принудительный `tool_choice`**. Если передать `"none"`, модель не вызовет инструменты, даже если они нужны. Мы используем `"auto"`, когда инструменты есть.
- **Пропущенный content.** Когда модель возвращает `tool_calls`, поле `content` может быть `null` или пустой строкой. Это нормально; наша реализация корректно возвращает `toolCalls` и игнорирует текст.
- **Невалидный JSON в аргументах.** Теоретически модель может сгенерировать битый JSON. Мы выбрасываем ошибку, чтобы вызывающий код мог повторить запрос или залогировать проблему.

### Вопросы для самопроверки

1. Почему мы передаём `tool_choice: "auto"`, а не `"required"` или имя конкретной функции?
2. Что произойдёт, если в ответе DeepSeek одновременно есть и `content`, и `tool_calls`? Как наш код обрабатывает такой случай?
3. Какие заголовки и тело запроса формирует библиотека `openai` при вызове DeepSeek?
4. Как обрабатывается ошибка аутентификации (неверный ключ)?

### Практическое задание к модулю 2.2

1. Реализуйте `DeepSeekClient` по приведённому коду. Проверьте его на реальном ключе с инструментами «погода» и «поиск».
2. Добавьте в конструктор опциональный параметр `timeoutMs`, прокидывая его в `OpenAI` через `timeout` (см. документацию пакета `openai`).
3. Модифицируйте метод `generate` так, чтобы при возникновении ошибки сети он выбрасывал типизированную ошибку `LLMClientError` с полем `provider` и `statusCode`.

---

## Модуль 2.3. YandexGPT, фабрика и юнит-тестирование

### Теория

#### Способы работы с YandexGPT

- **Нативный SDK `@yandex-cloud/nodejs-sdk`** — толстая обёртка над gRPC API. Требует генерации protobuf-типов, но даёт строгую типизацию.
- **REST API** — эндпоинт `https://llm.api.cloud.yandex.net/foundationModels/v1/completion` с простым JSON-интерфейсом. Легче тестировать и перехватывать через msw.

В учебных целях мы выберем REST API, имитируя работу SDK, но оставляя возможность миграции на gRPC в будущем.

#### Формат описания функций

YandexGPT ожидает в теле запроса поле `functions` — массив объектов `FunctionDefinition`:

```json
{
  "name": "calculator",
  "description": "Выполняет арифметическую операцию",
  "parameters": {
    "type": "object",
    "properties": { ... },
    "required": ["operation", "a", "b"]
  }
}
```

Это классическая JSON Schema. Мы сгенерируем её через `zodToJsonSchema`.

#### Аутентификация

Yandex Cloud принимает один из вариантов:

- API-ключ сервисного аккаунта в заголовке `Authorization: Api-Key <key>`;
- IAM-токен в заголовке `Authorization: Bearer <token>`.

Мы будем передавать `apiKey` и использовать заголовок `Api-Key`.

#### Ответ YandexGPT при вызове функций

Успешный ответ содержит `result.alternatives[0].message.function_call`:

```json
{
  "function_call": {
    "name": "calculator",
    "arguments": { "operation": "add", "a": 5, "b": 7 }
  }
}
```

Аргументы уже приходят в виде объекта, а не строки! Это важное отличие от DeepSeek.

### Практика

#### Реализация `YandexGPTClient`

```typescript
import { zodToJsonSchema } from "zod-to-json-schema";
import type { IToolCallingClient } from "./interface";
import type { Message, ToolDef, ToolCall, LLMResponse } from "./types";

interface YandexGPTConfig {
  folderId: string;
  apiKey: string;
  model?: string; // например, "yandexgpt-latest"
}

/**
 * Клиент для YandexGPT через REST API v1/completion.
 */
export class YandexGPTClient implements IToolCallingClient {
  private folderId: string;
  private apiKey: string;
  private model: string;

  constructor(config: YandexGPTConfig) {
    this.folderId = config.folderId;
    this.apiKey = config.apiKey;
    this.model = config.model ?? "yandexgpt-latest";
  }

  async generate(messages: Message[], tools?: ToolDef[]): Promise<LLMResponse> {
    const url = "https://llm.api.cloud.yandex.net/foundationModels/v1/completion";

    const body: any = {
      modelUri: `gpt://${this.folderId}/${this.model}`,
      completionOptions: {
        stream: false,
        temperature: 0.6,
      },
      messages: this.mapMessages(messages),
    };

    if (tools && tools.length > 0) {
      body.functions = tools.map(t => ({
        name: t.name,
        description: t.description,
        parameters: zodToJsonSchema(t.parameters),
      }));
    }

    const response = await fetch(url, {
      method: "POST",
      headers: {
        "Content-Type": "application/json",
        Authorization: `Api-Key ${this.apiKey}`,
      },
      body: JSON.stringify(body),
    });

    if (!response.ok) {
      const errorText = await response.text();
      throw new Error(`YandexGPT API error ${response.status}: ${errorText}`);
    }

    const data = await response.json();
    const alternative = data?.result?.alternatives?.[0];
    if (!alternative) {
      throw new Error("No alternatives in YandexGPT response");
    }

    const msg = alternative.message;
    if (msg?.function_call) {
      const toolCall: ToolCall = {
        id: this.generateToolCallId(),
        name: msg.function_call.name,
        arguments: msg.function_call.arguments, // уже объект
      };
      return { toolCalls: [toolCall] };
    }

    return { text: msg?.text ?? "" };
  }

  private mapMessages(messages: Message[]): any[] {
    return messages.map(msg => {
      const base: any = { role: msg.role };
      if (msg.content) {
        base.text = msg.content;
      }
      if (msg.tool_calls) {
        // YandexGPT пока не поддерживает native tool_calls в сообщениях,
        // поэтому мы опускаем или преобразуем в текстовое представление.
        // В реальном проекте стоит добавить сериализацию.
      }
      if (msg.role === "tool") {
        base.text = msg.output ?? "";
        base.tool_call_id = msg.tool_call_id;
      }
      return base;
    });
  }

  private generateToolCallId(): string {
    return `call_${Math.random().toString(36).substring(2, 10)}`;
  }
}
```

#### Фабрика клиентов

```typescript
import { IToolCallingClient } from "./interface";
import { DeepSeekClient } from "./DeepSeekClient";
import { YandexGPTClient } from "./YandexGPTClient";

export type Provider = "deepseek" | "yandex";

export function createToolCallingClient(
  provider: Provider,
  config: { apiKey: string; folderId?: string; baseURL?: string; model?: string }
): IToolCallingClient {
  switch (provider) {
    case "deepseek":
      return new DeepSeekClient(config.apiKey, config.baseURL, config.model);
    case "yandex":
      if (!config.folderId) throw new Error("folderId is required for YandexGPT");
      return new YandexGPTClient({
        folderId: config.folderId,
        apiKey: config.apiKey,
        model: config.model,
      });
    default:
      throw new Error(`Unsupported provider: ${provider}`);
  }
}
```

Использование:

```typescript
const client = createToolCallingClient("deepseek", {
  apiKey: process.env.DEEPSEEK_API_KEY!,
});
// или
const yandexClient = createToolCallingClient("yandex", {
  apiKey: process.env.YANDEX_API_KEY!,
  folderId: process.env.YANDEX_FOLDER_ID!,
});
```

#### Сравнительное ручное тестирование

Запустите один и тот же запрос с инструментами на обоих клиентах и визуально сравните:

- Качество распознавания нужного инструмента.
- Формат аргументов (строки, числа, вложенные объекты).
- Реакцию на отсутствие подходящего инструмента (должен вернуть текст или отказаться).

### Юнит-тестирование с MSW

Установим vitest и msw:

```bash
npm install -D vitest msw
```

#### Настройка mock-сервера

`src/__tests__/mocks/handlers.ts`:

```typescript
import { http, HttpResponse } from "msw";

export const handlers = [
  // DeepSeek — text response
  http.post("https://api.deepseek.com/v1/chat/completions", async ({ request }) => {
    const body = await request.json();
    // Здесь можно проверить содержимое body
    return HttpResponse.json({
      choices: [{
        message: { role: "assistant", content: "Hello from DeepSeek" }
      }]
    });
  }),

  // DeepSeek — tool_calls response
  http.post("https://api.deepseek.com/v1/chat/completions", async ({ request }) => {
    // упрощённо: если в теле есть tools, вернём tool_calls
    const body = await request.json();
    if (body.tools) {
      return HttpResponse.json({
        choices: [{
          message: {
            role: "assistant",
            tool_calls: [{
              id: "call_1",
              type: "function",
              function: { name: "test_tool", arguments: '{"param":42}' }
            }]
          }
        }]
      });
    }
    return HttpResponse.json({
      choices: [{ message: { role: "assistant", content: "No tools" } }]
    });
  }),

  // YandexGPT — text response
  http.post("https://llm.api.cloud.yandex.net/foundationModels/v1/completion", async () => {
    return HttpResponse.json({
      result: {
        alternatives: [{ message: { text: "Ответ YandexGPT" } }]
      }
    });
  }),

  // YandexGPT — function_call response
  http.post("https://llm.api.cloud.yandex.net/foundationModels/v1/completion", async ({ request }) => {
    const body = await request.json();
    if (body.functions) {
      return HttpResponse.json({
        result: {
          alternatives: [{
            message: {
              function_call: {
                name: "test_func",
                arguments: { param: 42 }
              }
            }
          }]
        }
      });
    }
    return HttpResponse.json({
      result: {
        alternatives: [{ message: { text: "Без функций" } }]
      }
    });
  }),
];
```

В реальном проекте следует использовать динамические обработчики, проверяющие тело запроса, или отдельные серверы для разных тестов. Здесь показан скелет.

#### Пример теста для DeepSeekClient

`src/__tests__/DeepSeekClient.test.ts`:

```typescript
import { describe, it, expect, beforeAll, afterAll } from "vitest";
import { setupServer } from "msw/node";
import { DeepSeekClient } from "../tool-calling-client/DeepSeekClient";
import { handlers } from "./mocks/handlers";

const server = setupServer(...handlers);

beforeAll(() => server.listen());
afterAll(() => server.close());

describe("DeepSeekClient", () => {
  const client = new DeepSeekClient("test-key");

  it("возвращает текстовый ответ", async () => {
    const response = await client.generate([{ role: "user", content: "Привет" }]);
    expect(response.text).toBe("Hello from DeepSeek");
    expect(response.toolCalls).toBeUndefined();
  });

  it("возвращает tool_calls при наличии инструментов", async () => {
    const tool = { name: "test_tool", description: "", parameters: {} as any };
    const response = await client.generate(
      [{ role: "user", content: "Вызови инструмент" }],
      [tool]
    );
    expect(response.toolCalls).toHaveLength(1);
    expect(response.toolCalls![0].arguments).toEqual({ param: 42 });
  });

  it("выбрасывает ошибку при невалидном JSON аргументов", async () => {
    // переопределим обработчик на этот тест с битым JSON
    server.use(
      http.post("https://api.deepseek.com/v1/chat/completions", () =>
        HttpResponse.json({
          choices: [{
            message: {
              tool_calls: [{
                id: "x",
                function: { name: "t", arguments: "{" }
              }]
            }
          }]
        })
      )
    );
    await expect(client.generate([{ role: "user", content: "test" }], [{} as any]))
      .rejects.toThrow("Invalid JSON");
  });
});
```

Аналогично пишутся тесты для YandexGPTClient с проверкой преобразования форматов.

#### Тест фабрики

```typescript
import { createToolCallingClient } from "../tool-calling-client/factory";

it("фабрика создаёт DeepSeekClient", () => {
  const client = createToolCallingClient("deepseek", { apiKey: "key" });
  expect(client.constructor.name).toBe("DeepSeekClient");
});

it("фабрика создаёт YandexGPTClient", () => {
  const client = createToolCallingClient("yandex", { apiKey: "key", folderId: "folder" });
  expect(client.constructor.name).toBe("YandexGPTClient");
});
```

### Типичные подводные камни

- **IAM vs API Key.** Yandex Cloud требует разных заголовков. Проверьте документацию: `Api-Key` для ключа, `Bearer` для токена. Не перепутайте.
- **Несовместимость форматов.** YandexGPT в ответе даёт аргументы объектом, а DeepSeek — строкой. Наш `ToolCall.arguments` должен быть всегда объектом, поэтому для DeepSeek мы парсим JSON.
- **Ограничения по длине.** У YandexGPT есть ограничение на размер запроса (около 8 КБ токенов). При большом количестве инструментов можно превысить лимит — фильтруйте `ToolDef` или используйте эмбеддинги.
- **Отсутствие `function_call`.** Если модель решает не вызывать функцию, ответ может вообще не содержать этого поля. Мы корректно возвращаем текст.
- **Mock-сервер и реалистичность.** В тестах важно проверять не только успешный путь, но и ошибки аутентификации (401), таймауты и нестандартные ответы.

### Вопросы для самопроверки

1. Почему в `YandexGPTClient` мы генерируем `id` для `ToolCall` самостоятельно, а DeepSeek возвращает его из API?
2. Чем отличается формат `functions` в YandexGPT от `tools` в DeepSeek? Покажите на примере JSON.
3. Какие заголовки необходимо установить при использовании API-ключа Yandex Cloud?
4. Как в фабрике клиентов обрабатывается случай, когда передан неподдерживаемый провайдер?

### Практическое задание к модулю 2.3

1. Реализуйте `YandexGPTClient` и фабрику по примерам. Проверьте на реальном Yandex Cloud аккаунте (понадобится folderId и API-ключ).
2. Напишите юнит-тесты для `YandexGPTClient`, покрывающие текстовый ответ, вызов функции и ошибку сервера.
3. Добавьте в фабрику возможность чтения провайдера из переменной окружения `LLM_PROVIDER`, а ключей — из `LLM_API_KEY` и `LLM_FOLDER_ID`.

---

## Итоговое домашнее задание блока 2

### Поддержка стриминга ответов от DeepSeek

**Задача:** Расширить возможности нашего универсального слоя, добавив потоковую генерацию (streaming) для DeepSeek. При этом **интерфейс `IToolCallingClient` должен остаться неизменным**, а существующий код, использующий `generate()`, не должен сломаться.

**Почему это важно:** Агенты и UI часто требуют отображения ответа по мере генерации. Без стриминга пользователь ждёт весь ответ целиком.

**Требования:**

1. Создать новый интерфейс `IStreamingToolCallingClient`, который расширяет `IToolCallingClient` и добавляет метод `generateStream`:
   ```typescript
   import type { IToolCallingClient } from "./interface";
   import type { Message, ToolDef, LLMResponse } from "./types";

   export interface IStreamingToolCallingClient extends IToolCallingClient {
     /**
      * Потоковая генерация, возвращающая AsyncIterable частичных ответов.
      * Каждый чанк — LLMResponse с обновлённым текстом или tool_calls.
      * По завершении модель присылает финальный LLMResponse.
      */
     generateStream(
       messages: Message[],
       tools?: ToolDef[]
     ): AsyncIterable<LLMResponse>;
   }
   ```
2. Реализовать в классе `DeepSeekStreamingClient` оба интерфейса. Метод `generate` должен работать как раньше (без стриминга). Метод `generateStream` должен использовать `stream: true` в API DeepSeek и возвращать итератор, который по мере получения чанков эмитит частичные ответы. Учесть, что при стриминге `tool_calls` собираются постепенно (в чанках приходят дельты). Необходимо агрегировать дельты до финального `ToolCall`.

3. Покрыть юнит-тестами с использованием msw:
   - Эмуляция SSE-потока (Server-Sent Events) с чанками текста.
   - Эмуляция стрима с постепенным появлением `tool_calls`.
   - Проверка, что итератор корректно отдаёт промежуточные состояния и финальный результат.

4. Обратите внимание: остальные клиенты (YandexGPTClient) пока не обязаны поддерживать стриминг, но архитектура должна позволять добавить его в будущем без изменения базового интерфейса.

**Рекомендации по реализации:**

- Используйте потоковую возможность OpenAI-клиента: `client.chat.completions.create({ stream: true, ... })` возвращает `Stream<ChatCompletionChunk>`, который можно итерировать с помощью `for await (const chunk of stream)`.
- Для агрегации `tool_calls` отслеживайте индекс в массиве, имя функции и накапливайте строку аргументов. Когда чанк содержит `finish_reason`, значит, вызов завершён — парсим накопленные аргументы.
- В msw можно вернуть кастомный `ReadableStream` с чанками, либо использовать `HttpResponse.arrayBuffer` и хитрые заглушки; удобнее воспользоваться `HttpResponse` с `headers: { 'Content-Type': 'text/event-stream' }` и телом, составленным из `data: ...\n\n`.

**Критерии приёмки:**

- `IToolCallingClient` не изменился ни на байт.
- `DeepSeekStreamingClient` проходит тесты на текстовый стрим, стрим с одним инструментом, стрим с двумя инструментами.
- Фабрика может опционально создавать стриминговый клиент при флаге `streaming: true` (но это не меняет тип возврата для `generate`).

Удачи! Это задание закрепит навыки абстракции, обработки потоков и тестирования асинхронных итераторов.

---

По окончании блока вы получите надёжный фундамент для построения агентов, которые могут работать с разными LLM «под капотом», не меняя бизнес-логику. В следующем блоке мы начнём подключать кастомные MCP-серверы и управлять ими через этот универсальный клиент.