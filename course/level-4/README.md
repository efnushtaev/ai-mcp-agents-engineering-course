# Блок 4. MCP-сервер для Компас-3D

В этом блоке вы спроектируете и реализуете специализированный MCP-сервер, который предоставляет агентному ИИ инструменты для работы с системой автоматизированного проектирования Компас-3D. Поскольку установка и лицензирование реального Компас-3D требуют Windows и COM-объектов, мы создадим кроссплатформенный эмулятор САПР, имитирующий основные операции. Затем обернём его REST API в MCP-инструменты, а в финале напишем агента-конструктора, способного по текстовому описанию сгенерировать STEP- или DXF-файл детали.

Вы будете использовать:

- **Fastify** — для построения эмулятора;
- **Zod** — для валидации данных;
- **MCP SDK** (`@modelcontextprotocol/sdk`) — для реализации инструментов;
- **Универсальный ToolCallingClient** из Блока 2 — для связи с LLM (DeepSeek / YandexGPT);
- **Vitest** и `light-my-request` — для тестирования.

К концу блока у вас появится полноценный связка «LLM → MCP-сервер → эмулятор Компас-3D», способная материализовать текстовые инструкции в файлы трёхмерных моделей.

---

## Модуль 4.1. Проектирование инструментов и эмулятор Компас-3D

### Теория

#### Зачем нужен эмулятор

Реальный API Компас-3D базируется на COM-технологиях, требует установленного приложения, работает только под Windows и предполагает наличие лицензии. Для учебного процесса и локальной разработки это серьёзное препятствие. Мы создаём легковесный HTTP-сервер, который:

- эмулирует поведение САПР: создание документа, построение эскиза, выдавливание, экспорт;
- хранит состояние в памяти (историю операций);
- возвращает осмысленные идентификаторы и файлы-заглушки с корректными расширениями.

Это позволяет разрабатывать и отлаживать весь конвейер «LLM → MCP-сервер → САПР» на любой ОС, без лицензионных ограничений. При переходе к реальному «Компасу» достаточно будет заменить HTTP-клиента в MCP-сервере на вызовы COM-интерфейса — контракт инструментов останется тем же.

#### Минимальный набор операций

Наш курсовой ассистент-конструктор должен уметь:

1. Создать новый документ (новую деталь).
2. Построить эскиз на одной из трёх ортогональных плоскостей (XY, XZ, YZ) с простейшей геометрией: прямоугольник или окружность.
3. Выполнить операцию выдавливания эскиза на заданную глубину и направление.
4. Экспортировать полученную деталь в файл формата STEP или DXF.

Этого достаточно, чтобы создавать базовые 3D-модели и формировать выходные файлы.

#### Проект REST API эмулятора

Эмулятор будет представлять собой HTTP-сервер на Fastify со следующими конечными точками:

- `POST /documents` — создать новый документ, возвращает `{ documentId }`.
- `POST /documents/:id/sketch` — добавить эскиз на документ. Тело запроса: `{ plane, geometry: { type, ...params } }`. Возвращает `{ sketchId }`.
- `POST /documents/:id/extrude` — выдавить эскиз. Тело: `{ sketchId, depth, direction? }`. Возвращает `{ featureId }`.
- `GET /documents/:id/export?format=step|dxf` — скачать файл. Эмулятор генерирует заглушку с правильным именем и расширением (например, `part_<id>.step`) и отправляет её.

Состояние хранится в памяти: объект, содержащий массив документов, у каждого — эскизы и операции. При экспорте просто создаётся текстовый файл с сообщением «эмуляция STEP-файла».

---

### Практика: создание эмулятора `kompas3d-emulator`

#### 1. Инициализация проекта

```bash
mkdir kompas3d-emulator && cd kompas3d-emulator
npm init -y
npm install fastify zod uuid
npm install -D typescript @types/node vitest light-my-request
```

Настройте `tsconfig.json`:

```json
{
  "compilerOptions": {
    "target": "ES2022",
    "module": "ESNext",
    "moduleResolution": "node",
    "outDir": "./dist",
    "rootDir": "./src",
    "strict": true,
    "esModuleInterop": true
  }
}
```

Добавьте скрипты в `package.json`:

```json
"scripts": {
  "build": "tsc",
  "dev": "tsx src/server.ts",
  "test": "vitest"
}
```

#### 2. Модель состояния и Zod-схемы

Создайте файл `src/state.ts`:

```typescript
import { z } from 'zod';

// ----- Геометрия эскиза -----
export const RectangleSchema = z.object({
  type: z.literal('rectangle'),
  width: z.number().positive(),
  height: z.number().positive(),
  centerX: z.number().default(0),
  centerY: z.number().default(0),
});
export type Rectangle = z.infer<typeof RectangleSchema>;

export const CircleSchema = z.object({
  type: z.literal('circle'),
  radius: z.number().positive(),
  centerX: z.number().default(0),
  centerY: z.number().default(0),
});
export type Circle = z.infer<typeof CircleSchema>;

export const GeometrySchema = z.discriminatedUnion('type', [
  RectangleSchema,
  CircleSchema,
]);
export type Geometry = z.infer<typeof GeometrySchema>;

// ----- Плоскость -----
export const PlaneSchema = z.enum(['XY', 'XZ', 'YZ']);
export type Plane = z.infer<typeof PlaneSchema>;

// ----- Эскиз -----
export interface Sketch {
  id: string;
  plane: Plane;
  geometry: Geometry;
}

// ----- Операция выдавливания -----
export interface ExtrudeFeature {
  id: string;
  sketchId: string;
  depth: number;
  direction: 'forward' | 'backward';
}

// ----- Документ (деталь) -----
export interface Document {
  id: string;
  name: string;
  sketches: Sketch[];
  extrudes: ExtrudeFeature[];
}
```

Хранилище — `Map<string, Document>`:

```typescript
export const documents = new Map<string, Document>();
```

#### 3. Сервер Fastify с эндпоинтами

`src/server.ts`:

```typescript
import Fastify from 'fastify';
import { z } from 'zod';
import { v4 as uuidv4 } from 'uuid';
import {
  documents,
  PlaneSchema,
  GeometrySchema,
} from './state.js';

const app = Fastify({ logger: true });

// Health-check
app.get('/health', async () => ({ status: 'ok' }));

// Создать документ
app.post('/documents', async (req, reply) => {
  const { name } = z.object({ name: z.string().min(1) }).parse(req.body);
  const id = uuidv4();
  documents.set(id, {
    id,
    name,
    sketches: [],
    extrudes: [],
  });
  reply.code(201);
  return { documentId: id };
});

// Добавить эскиз
app.post('/documents/:id/sketch', async (req, reply) => {
  const { id } = z.object({ id: z.string().uuid() }).parse(req.params);
  const doc = documents.get(id);
  if (!doc) {
    reply.code(404);
    return { error: 'Document not found' };
  }

  const bodySchema = z.object({
    plane: PlaneSchema,
    geometry: GeometrySchema,
  });
  const { plane, geometry } = bodySchema.parse(req.body);

  const sketchId = uuidv4();
  doc.sketches.push({ id: sketchId, plane, geometry });

  reply.code(201);
  return { sketchId };
});

// Выдавить эскиз
app.post('/documents/:id/extrude', async (req, reply) => {
  const { id } = z.object({ id: z.string().uuid() }).parse(req.params);
  const doc = documents.get(id);
  if (!doc) {
    reply.code(404);
    return { error: 'Document not found' };
  }

  const bodySchema = z.object({
    sketchId: z.string().uuid(),
    depth: z.number().positive(),
    direction: z.enum(['forward', 'backward']).default('forward'),
  });
  const { sketchId, depth, direction } = bodySchema.parse(req.body);

  const sketch = doc.sketches.find(s => s.id === sketchId);
  if (!sketch) {
    reply.code(404);
    return { error: 'Sketch not found in this document' };
  }

  const featureId = uuidv4();
  doc.extrudes.push({ id: featureId, sketchId, depth, direction });

  reply.code(201);
  return { featureId };
});

// Экспорт
app.get('/documents/:id/export', async (req, reply) => {
  const { id } = z.object({ id: z.string().uuid() }).parse(req.params);
  const { format } = z.object({
    format: z.enum(['step', 'dxf']),
  }).parse(req.query);
  const doc = documents.get(id);
  if (!doc) {
    reply.code(404);
    return { error: 'Document not found' };
  }

  const content = `Эмуляция файла ${format.toUpperCase()} для детали "${doc.name}" (id: ${doc.id})\n`;
  const filename = `part_${doc.id}.${format}`;
  reply
    .header('Content-Disposition', `attachment; filename="${filename}"`)
    .header('Content-Type', 'application/octet-stream')
    .send(Buffer.from(content, 'utf-8'));
});

// Запуск
const start = async () => {
  try {
    await app.listen({ port: 3001, host: '0.0.0.0' });
    console.log('Эмулятор Компас-3D запущен на http://localhost:3001');
  } catch (err) {
    app.log.error(err);
    process.exit(1);
  }
};
start();
```

#### 4. Интеграционные тесты

`tests/emulator.test.ts`:

```typescript
import { test, expect, beforeAll, afterAll } from 'vitest';
import Fastify from 'fastify';
import { documents } from '../src/state.js';
import { app } from '../src/server.js'; // экспортируем для тестирования

let server: ReturnType<typeof app.listen>;

beforeAll(async () => {
  server = await app.listen({ port: 0 }); // случайный порт
});

afterAll(async () => {
  await server.close();
  documents.clear();
});

test('health check', async () => {
  const res = await fetch(`http://localhost:${server.port}/health`);
  expect(res.status).toBe(200);
  expect(await res.json()).toEqual({ status: 'ok' });
});

test('create document, sketch, extrude, export', async () => {
  const base = `http://localhost:${server.port}`;

  // Создание документа
  const docRes = await fetch(`${base}/documents`, {
    method: 'POST',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify({ name: 'Тестовая деталь' }),
  });
  const { documentId } = await docRes.json();
  expect(documentId).toBeDefined();

  // Добавление эскиза
  const sketchRes = await fetch(`${base}/documents/${documentId}/sketch`, {
    method: 'POST',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify({
      plane: 'XY',
      geometry: { type: 'rectangle', width: 10, height: 10 },
    }),
  });
  const { sketchId } = await sketchRes.json();
  expect(sketchId).toBeDefined();

  // Выдавливание
  const extrudeRes = await fetch(`${base}/documents/${documentId}/extrude`, {
    method: 'POST',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify({ sketchId, depth: 10 }),
  });
  const { featureId } = await extrudeRes.json();
  expect(featureId).toBeDefined();

  // Экспорт STEP
  const exportRes = await fetch(
    `${base}/documents/${documentId}/export?format=step`
  );
  expect(exportRes.status).toBe(200);
  expect(exportRes.headers.get('content-disposition')).toContain(
    `part_${documentId}.step`
  );
  const buffer = await exportRes.arrayBuffer();
  expect(buffer.byteLength).toBeGreaterThan(0);
});
```

Запустите тесты: `npm test`.

#### 5. Возможные ошибки и их устранение

| Симптом | Вероятная причина | Решение |
|---------|-------------------|---------|
| `ECONNREFUSED 127.0.0.1:3001` | Эмулятор не запущен | Запустите `npm run dev` и проверьте порт |
| `Invalid uuid` при запросе | Передан неправильный documentId | Убедитесь, что используете ID из ответа создания документа |
| ZodError при создании эскиза | Неполное тело запроса | Проверьте, что отправляете `plane` и `geometry` строго по схеме |
| 404 "Document not found" | Документ удалён или ID не существует | Проверьте, что перед вызовом эскиза/выдавливания создан документ |
| Файл экспорта содержит неожиданный текст | Это ожидаемо — эмулятор не генерирует реальный STEP | В реальном сценарии замените заглушку на вызов API Компаса |

---

### Вопросы для самопроверки

1. Почему для разработки MCP-сервера выгодно начинать с эмулятора, а не с реального API?
2. Какие типы геометрии эскиза поддерживает наш эмулятор? Как можно расширить набор в будущем?
3. Каким образом эмулятор хранит историю операций и почему именно in-memory?
4. Объясните, как Zod помогает обеспечить надёжность REST API эмулятора.
5. Как проверить, что сервер корректно обрабатывает отсутствующий документ?

### Практическое задание

1. Добавьте в эмулятор операцию `create_cut` (вырезание выдавливанием). Реализуйте новый эндпоинт `POST /documents/:id/cut`, который принимает `sketchId`, `depth` и возвращает `featureId`. Модифицируйте состояние документа, чтобы хранить и операции вырезания. Напишите тест, проверяющий создание выреза после выдавливания.
2. Реализуйте endpoint `DELETE /documents/:id` для удаления документа и связанных с ним эскизов/операций. Убедитесь, что после удаления попытка экспорта возвращает 404.

---

## Модуль 4.2. Реализация MCP-сервера `kompas3d-mcp`

### Теория

Теперь эмулятор играет роль «внешнего мира» — реальной САПР. Наш MCP-сервер выступает мостом между LLM-агентом и этим миром. Он будет запускаться как дочерний процесс, подключаться по stdio и предоставлять четыре инструмента, соответствующих операциям эмулятора.

#### Инструменты и их сигнатуры

1. **create_part**  
   Создаёт новую деталь в Компас (эмуляторе).  
   Вход: `name: string`  
   Выход: `documentId`

2. **create_sketch**  
   Добавляет эскиз на плоскость с заданной геометрией.  
   Вход: `documentId`, `plane` (XY|XZ|YZ), `geometry: { type, ...params }`  
   Выход: `sketchId`

3. **extrude**  
   Выдавливает эскиз на глубину.  
   Вход: `documentId`, `sketchId`, `depth: number`, `direction?` (forward|backward, по умолчанию forward)  
   Выход: `featureId`

4. **export_document**  
   Экспортирует документ в файл и сохраняет на диск.  
   Вход: `documentId`, `format` (step|dxf), `outputPath` — путь для сохранения файла.  
   Выход: `filePath` — подтверждение сохранения.

Каждый инструмент будет иметь Zod-схему для валидации входных аргументов, которая автоматически преобразуется в `inputSchema` MCP-инструмента.

#### Транспорт и обработка ошибок

Используем `StdioServerTransport` — это стандартный способ интеграции MCP-сервера с клиентом (оркестратором). Сервер получает запросы через stdin/stdout, что упрощает управление жизненным циклом.

В случае ошибок (эмулятор недоступен, документ не найден и т.п.) инструмент должен возвращать MCP-ошибку с понятным описанием. Внутренние исключения мы обернём в текстовые сообщения, чтобы LLM могла интерпретировать и, возможно, скорректировать запрос.

---

### Практика: создание `kompas3d-mcp`

#### 1. Инициализация проекта

```bash
mkdir kompas3d-mcp && cd kompas3d-mcp
npm init -y
npm install @modelcontextprotocol/sdk zod
npm install -D typescript @types/node vitest
```

Используем тот же `tsconfig.json`, что и в эмуляторе. Для запуска сервера удобно применять `tsx`.

#### 2. Реализация инструментов

`src/emulator-client.ts` — обёртка над REST API эмулятора:

```typescript
const EMULATOR_URL = process.env.EMULATOR_URL || 'http://localhost:3001';

export async function createPart(name: string): Promise<{ documentId: string }> {
  const res = await fetch(`${EMULATOR_URL}/documents`, {
    method: 'POST',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify({ name }),
  });
  if (!res.ok) {
    const body = await res.json().catch(() => ({}));
    throw new Error(`Failed to create part: ${res.status} ${JSON.stringify(body)}`);
  }
  return res.json();
}

export async function createSketch(
  documentId: string,
  plane: string,
  geometry: Record<string, unknown>
): Promise<{ sketchId: string }> {
  const res = await fetch(`${EMULATOR_URL}/documents/${documentId}/sketch`, {
    method: 'POST',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify({ plane, geometry }),
  });
  if (!res.ok) {
    const body = await res.json().catch(() => ({}));
    throw new Error(`Failed to create sketch: ${res.status} ${JSON.stringify(body)}`);
  }
  return res.json();
}

export async function extrude(
  documentId: string,
  sketchId: string,
  depth: number,
  direction: 'forward' | 'backward' = 'forward'
): Promise<{ featureId: string }> {
  const res = await fetch(`${EMULATOR_URL}/documents/${documentId}/extrude`, {
    method: 'POST',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify({ sketchId, depth, direction }),
  });
  if (!res.ok) {
    const body = await res.json().catch(() => ({}));
    throw new Error(`Failed to extrude: ${res.status} ${JSON.stringify(body)}`);
  }
  return res.json();
}

export async function exportDocument(
  documentId: string,
  format: 'step' | 'dxf',
  outputPath: string
): Promise<{ filePath: string }> {
  const url = `${EMULATOR_URL}/documents/${documentId}/export?format=${format}`;
  const res = await fetch(url);
  if (!res.ok) {
    const body = await res.json().catch(() => ({}));
    throw new Error(`Failed to export: ${res.status} ${JSON.stringify(body)}`);
  }

  // Сохраняем файл на диск
  const arrayBuffer = await res.arrayBuffer();
  const fs = await import('fs/promises');
  await fs.writeFile(outputPath, Buffer.from(arrayBuffer));

  return { filePath: outputPath };
}
```

`src/server.ts` — MCP-сервер:

```typescript
import { Server } from '@modelcontextprotocol/sdk/server/index.js';
import { StdioServerTransport } from '@modelcontextprotocol/sdk/server/stdio.js';
import {
  CallToolRequestSchema,
  ListToolsRequestSchema,
  Tool,
} from '@modelcontextprotocol/sdk/types.js';
import { z } from 'zod';
import * as emulator from './emulator-client.js';

// ----- Определение инструментов -----
const tools: Tool[] = [
  {
    name: 'create_part',
    description: 'Создать новую деталь в Компас-3D',
    inputSchema: {
      type: 'object',
      properties: {
        name: { type: 'string', description: 'Название детали' },
      },
      required: ['name'],
    },
  },
  {
    name: 'create_sketch',
    description: 'Создать эскиз на плоскости с геометрией (прямоугольник или окружность)',
    inputSchema: {
      type: 'object',
      properties: {
        documentId: { type: 'string', description: 'Идентификатор документа' },
        plane: { type: 'string', enum: ['XY', 'XZ', 'YZ'] },
        geometry: {
          type: 'object',
          oneOf: [
            {
              properties: {
                type: { const: 'rectangle' },
                width: { type: 'number', minimum: 0 },
                height: { type: 'number', minimum: 0 },
                centerX: { type: 'number', default: 0 },
                centerY: { type: 'number', default: 0 },
              },
              required: ['type', 'width', 'height'],
            },
            {
              properties: {
                type: { const: 'circle' },
                radius: { type: 'number', minimum: 0 },
                centerX: { type: 'number', default: 0 },
                centerY: { type: 'number', default: 0 },
              },
              required: ['type', 'radius'],
            },
          ],
        },
      },
      required: ['documentId', 'plane', 'geometry'],
    },
  },
  {
    name: 'extrude',
    description: 'Выдавить эскиз на заданную глубину',
    inputSchema: {
      type: 'object',
      properties: {
        documentId: { type: 'string' },
        sketchId: { type: 'string' },
        depth: { type: 'number', minimum: 0 },
        direction: {
          type: 'string',
          enum: ['forward', 'backward'],
          default: 'forward',
        },
      },
      required: ['documentId', 'sketchId', 'depth'],
    },
  },
  {
    name: 'export_document',
    description: 'Экспортировать документ в STEP или DXF и сохранить на диск',
    inputSchema: {
      type: 'object',
      properties: {
        documentId: { type: 'string' },
        format: { type: 'string', enum: ['step', 'dxf'] },
        outputPath: { type: 'string', description: 'Абсолютный путь для сохранения файла' },
      },
      required: ['documentId', 'format', 'outputPath'],
    },
  },
];

// Создаём сервер
const server = new Server(
  {
    name: 'kompas3d-mcp',
    version: '0.1.0',
  },
  {
    capabilities: { tools: {} },
  }
);

// Обработчик списка инструментов
server.setRequestHandler(ListToolsRequestSchema, async () => ({ tools }));

// Обработчик вызова инструмента
server.setRequestHandler(CallToolRequestSchema, async (request) => {
  const { name, arguments: args } = request.params;
  try {
    switch (name) {
      case 'create_part': {
        const schema = z.object({ name: z.string().min(1) });
        const { name: partName } = schema.parse(args);
        const result = await emulator.createPart(partName);
        return { content: [{ type: 'text', text: JSON.stringify(result) }] };
      }
      case 'create_sketch': {
        const schema = z.object({
          documentId: z.string().uuid(),
          plane: z.enum(['XY', 'XZ', 'YZ']),
          geometry: z.discriminatedUnion('type', [
            z.object({
              type: z.literal('rectangle'),
              width: z.number().positive(),
              height: z.number().positive(),
              centerX: z.number().default(0),
              centerY: z.number().default(0),
            }),
            z.object({
              type: z.literal('circle'),
              radius: z.number().positive(),
              centerX: z.number().default(0),
              centerY: z.number().default(0),
            }),
          ]),
        });
        const { documentId, plane, geometry } = schema.parse(args);
        const result = await emulator.createSketch(documentId, plane, geometry);
        return { content: [{ type: 'text', text: JSON.stringify(result) }] };
      }
      case 'extrude': {
        const schema = z.object({
          documentId: z.string().uuid(),
          sketchId: z.string().uuid(),
          depth: z.number().positive(),
          direction: z.enum(['forward', 'backward']).default('forward'),
        });
        const { documentId, sketchId, depth, direction } = schema.parse(args);
        const result = await emulator.extrude(documentId, sketchId, depth, direction);
        return { content: [{ type: 'text', text: JSON.stringify(result) }] };
      }
      case 'export_document': {
        const schema = z.object({
          documentId: z.string().uuid(),
          format: z.enum(['step', 'dxf']),
          outputPath: z.string().min(1),
        });
        const { documentId, format, outputPath } = schema.parse(args);
        const result = await emulator.exportDocument(documentId, format, outputPath);
        return { content: [{ type: 'text', text: `Файл сохранён: ${result.filePath}` }] };
      }
      default:
        throw new Error(`Unknown tool: ${name}`);
    }
  } catch (error: any) {
    return {
      content: [{ type: 'text', text: `Ошибка: ${error.message}` }],
      isError: true,
    };
  }
});

// Запуск
async function main() {
  const transport = new StdioServerTransport();
  await server.connect(transport);
  console.error('kompas3d-mcp сервер запущен');
}

main().catch(console.error);
```

#### 3. Тестирование через MCP Inspector

1. Убедитесь, что эмулятор запущен (`npm run dev` в `kompas3d-emulator`).
2. Установите MCP Inspector глобально (если ещё нет): `npm install -g @modelcontextprotocol/inspector`.
3. Запустите сервер с инспектором:
   ```bash
   npx @modelcontextprotocol/inspector tsx src/server.ts
   ```
4. В открывшемся интерфейсе выполните цепочку вызовов:
   - `create_part` (имя «Куб»)
   - `create_sketch` (прямоугольник 10×10, плоскость XY)
   - `extrude` (глубина 10)
   - `export_document` (format step, path `./output/cube.step`)
5. Проверьте, что файл создался в указанном каталоге и содержит текст-заглушку.

#### 4. Возможные ошибки

| Ошибка | Причина | Решение |
|--------|---------|---------|
| `ECONNREFUSED` при вызове инструментов | Эмулятор не запущен или порт не совпадает | Проверьте `EMULATOR_URL` (по умолчанию `http://localhost:3001`) |
| `Invalid uuid` в ответе | Передан некорректный ID | Убедитесь, что используете валидный UUID из ответа `create_part` |
| MCP-сервер не стартует, ошибка импорта SDK | Некорректная версия `@modelcontextprotocol/sdk` | Установите последнюю версию: `npm i @modelcontextprotocol/sdk@latest` |
| Инспектор показывает пустой список инструментов | Обработчик `ListToolsRequestSchema` не зарегистрирован | Проверьте, что `server.setRequestHandler` вызван до `server.connect` |
| Ошибка при сохранении файла: `EACCES` | Нет прав на запись в указанную директорию | Укажите путь, доступный для записи (например, `./output`) |

---

### Вопросы для самопроверки

1. Какую роль выполняет `inputSchema` в определении MCP-инструмента и зачем она нужна LLM?
2. Почему мы используем `StdioServerTransport`, а не HTTP транспорт?
3. Какие меры предприняты для обработки ошибок и почему это важно для LLM-агента?
4. Можно ли было бы обойтись без эмулятора, если бы мы тестировали напрямую с реальным Компас-3D? Какие сложности это вызвало бы?
5. Как бы вы добавили поддержку асинхронной операции длительного рендеринга (например, генерация сложной модели)?

### Практическое задание

1. Добавьте инструмент `cut` (вырезание), зеркально похожий на `extrude`, но вызывающий `POST /documents/:id/cut` в эмуляторе. Обновите `emulator-client.ts`, добавьте новый инструмент в MCP-сервер и протестируйте через Inspector.
2. Реализуйте инструмент `list_documents`, который возвращает список всех созданных на данный момент документов (GET `/documents`). Добавьте этот endpoint в эмулятор.

---

## Модуль 4.3. Интеграция с LLM и создание агента-конструктора

### Теория

Мы подошли к финальному шагу — созданию интеллектуального агента, который способен по текстовому описанию проектировать деталь, вызывая инструменты MCP-сервера. Используем универсальный клиент `IToolCallingClient` из Блока 2 (поддерживающий DeepSeek и YandexGPT).

#### Архитектура агента

Агент-конструктор — это скрипт, который:

1. Запускает эмулятор и MCP-сервер как дочерние процессы (либо предполагает, что они уже запущены).
2. Инициализирует MCP-клиент через `StdioClientTransport` и получает список инструментов.
3. Преобразует MCP-инструменты в формат, понятный LLM (массив function definitions).
4. Создаёт экземпляр `ToolCallingClient` (DeepSeek или YandexGPT) с системным промптом, описывающим роль «инженера-конструктора, использующего Компас-3D».
5. Запускает диалог: пользователь вводит задание → агент отправляет его в LLM, LLM при необходимости вызывает инструменты, агент выполняет их через MCP-клиент, результат возвращается обратно в LLM, и так до получения финального ответа.
6. По окончании выводит сообщение с путём к сгенерированному файлу.

Системный промпт должен чётко описывать порядок работы: сначала создать документ, затем эскиз, выдавить, потом экспортировать. Если LLM ошибается в последовательности, агент может подсказать через дополнительный промпт.

#### Цикл обработки вызовов инструментов

Типичный цикл:
- Пользователь: «Создай куб 10×10×10 мм и сохрани в step».
- LLM решает: `create_part("Куб")` → получает `documentId`.
- Затем `create_sketch(documentId, "XY", { type: "rectangle", width: 10, height: 10 })` → `sketchId`.
- Затем `extrude(documentId, sketchId, 10)` → `featureId`.
- Наконец `export_document(documentId, "step", "./output/cube.step")` → сообщение об успехе.
- LLM возвращает финальный ответ: «Куб сохранён в ./output/cube.step».

Агент должен обрабатывать множественные вызовы за один ход (LLM может запросить несколько функций параллельно, если это возможно, но в нашем случае они зависимы — поэтому инструменты будут вызываться последовательно).

---

### Практика: реализация агента-конструктора

#### 1. Подготовка

Убедитесь, что у вас есть проект с универсальным клиентом из Блока 2. Обычно это класс `ToolCallingClient`, принимающий провайдера (DeepSeek или YandexGPT) и поддерживающий методы `chat` с передачей инструментов.

Создайте новый скрипт `agent-constructor.ts` в отдельной директории (можно в корне монорепозитория) или прямо в `kompas3d-mcp`. Для упрощения предположим, что `ToolCallingClient` доступен через импорт из пакета `toolcalling-client`.

#### 2. Код агента

```typescript
import { StdioClientTransport } from '@modelcontextprotocol/sdk/client/stdio.js';
import { Client } from '@modelcontextprotocol/sdk/client/index.js';
import { spawn, ChildProcess } from 'child_process';
import path from 'path';
import { ToolCallingClient } from 'toolcalling-client'; // гипотетический импорт
import type { ChatMessage, FunctionDefinition } from 'toolcalling-client';

// ----- Запуск вспомогательных процессов -----
function startEmulator(): ChildProcess {
  const emulator = spawn('node', ['dist/server.js'], {
    cwd: path.resolve('../kompas3d-emulator'),
    env: { ...process.env, PORT: '3001' },
    stdio: 'pipe',
  });
  emulator.stdout?.on('data', (data) => console.log(`[Эмулятор] ${data}`));
  emulator.stderr?.on('data', (data) => console.error(`[Эмулятор] ${data}`));
  return emulator;
}

function startMcpServer(): { process: ChildProcess; transport: StdioClientTransport } {
  const mcpProcess = spawn('tsx', ['src/server.ts'], {
    cwd: path.resolve('../kompas3d-mcp'),
    env: { ...process.env, EMULATOR_URL: 'http://localhost:3001' },
    stdio: ['pipe', 'pipe', 'pipe'],
  });
  const transport = new StdioClientTransport({
    reader: mcpProcess.stdout!,
    writer: mcpProcess.stdin!,
  });
  return { process: mcpProcess, transport };
}

// ----- Преобразование MCP-инструментов в FunctionDefinition для LLM -----
function mcpToolToFunctionDef(tool: any): FunctionDefinition {
  return {
    name: tool.name,
    description: tool.description,
    parameters: tool.inputSchema,
  };
}

// ----- Основная логика агента -----
async function main() {
  const userPrompt = process.argv[2];
  if (!userPrompt) {
    console.error('Использование: tsx agent-constructor.ts "текст задания"');
    process.exit(1);
  }

  // Запускаем эмулятор и MCP-сервер
  const emulator = startEmulator();
  // Даём время на запуск
  await new Promise((r) => setTimeout(r, 2000));
  const { process: mcpProcess, transport } = startMcpServer();
  const mcpClient = new Client(
    { name: 'agent-constructor', version: '1.0.0' },
    { capabilities: { tools: {} } }
  );
  await mcpClient.connect(transport);
  console.log('MCP-клиент подключён');

  // Получаем инструменты
  const { tools } = await mcpClient.listTools();
  const functionDefinitions = tools.map(mcpToolToFunctionDef);

  // Инициализация LLM-клиента (предположим DeepSeek)
  const llm = new ToolCallingClient({
    provider: 'deepseek',
    apiKey: process.env.DEEPSEEK_API_KEY!,
    model: 'deepseek-chat',
  });

  // Системный промпт
  const systemMessage: ChatMessage = {
    role: 'system',
    content: `Ты инженер-конструктор, работающий в САПР Компас-3D. Твоя задача — по описанию пользователя создать 3D-модель детали, используя доступные инструменты.
Порядок действий:
1. Создай документ (create_part).
2. Добавь эскиз на нужной плоскости (create_sketch). Поддерживаются прямоугольник и окружность.
3. Выдави эскиз (extrude). Глубина и направление указываются.
4. Экспортируй документ в указанный формат, сохранив файл в путь, который запросил пользователь, или в ./output/ по умолчанию.
Всегда возвращай путь к итоговому файлу. Если инструмент вернул ошибку, проанализируй и попробуй исправить запрос.`,
  };

  let messages: ChatMessage[] = [
    systemMessage,
    { role: 'user', content: userPrompt },
  ];

  const outputDir = path.resolve('./output');
  // Гарантируем существование выходной директории
  await import('fs/promises').then(fs => fs.mkdir(outputDir, { recursive: true }));

  // Цикл общения
  while (true) {
    const response = await llm.chat({
      messages,
      functions: functionDefinitions,
      function_call: 'auto',
    });

    const choice = response.choices[0];
    if (choice.finish_reason === 'stop') {
      console.log('Финальный ответ агента:', choice.message.content);
      break;
    }

    if (choice.finish_reason === 'function_call') {
      const functionCall = choice.message.function_call;
      console.log(`Вызов инструмента: ${functionCall.name}(${functionCall.arguments})`);

      // Добавляем сообщение ассистента в историю
      messages.push({
        role: 'assistant',
        content: null,
        function_call: functionCall,
      });

      // Выполняем инструмент через MCP
      const result = await mcpClient.callTool({
        name: functionCall.name,
        arguments: JSON.parse(functionCall.arguments),
      });

      // Текст результата (может быть массивом content)
      const resultText = result.content
        .filter((c: any) => c.type === 'text')
        .map((c: any) => c.text)
        .join('\n');

      console.log('Результат:', resultText);

      // Добавляем результат в историю
      messages.push({
        role: 'function',
        name: functionCall.name,
        content: result.isError ? `Ошибка: ${resultText}` : resultText,
      });
    } else {
      console.error('Неожиданный finish_reason:', choice.finish_reason);
      break;
    }
  }

  // Завершаем процессы
  mcpProcess.kill();
  emulator.kill();
  console.log('Агент завершил работу');
}

main().catch(console.error);
```

#### 3. Тестовые промпты и демонстрация

Запустите агента:

```bash
tsx agent-constructor.ts "Создай куб 10×10×10 мм и сохрани в step в файл ./output/cube.step"
```

Ожидаемое поведение:
- Агент создаст документ, эскиз (прямоугольник 10×10), выдавит на 10 мм, экспортирует в `./output/cube.step`.
- В консоли появится лог вызовов и финальное сообщение с путём.

Второй пример:

```bash
tsx agent-constructor.ts "Пластина 100x50 мм с отверстием диаметром 10 мм в центре, выдави на 5 мм, сохрани в dxf"
```

Здесь LLM может сначала создать эскиз с прямоугольником, потом добавить отверстие (потребуется инструмент `cut`), затем выдавить. Если инструмента `cut` ещё нет, модель может сообщить о невозможности. Это нормально — вы можете добавить его, выполнив задание из модуля 4.2.

#### 4. Возможные ошибки и их устранение

| Симптом | Причина | Решение |
|---------|---------|---------|
| LLM вызывает инструменты в неправильном порядке | Системный промпт недостаточно чёткий | Уточните инструкцию, добавьте примеры в промпт |
| `Function_call` содержит невалидный JSON | LLM сгенерировала некорректные аргументы | Добавьте обработку ошибки парсинга, попросите LLM исправить |
| MCP-инструмент возвращает ошибку «Document not found» | LLM использует неверный documentId | Убедитесь, что ID передаётся из предыдущего вызова, и в истории сохраняются корректные результаты |
| Агент зависает при запуске эмулятора | Процесс не успел стартовать | Увеличьте задержку или реализуйте ожидание через проверку health-check |
| Экспортированный файл пуст или повреждён | Эмулятор возвращает заглушку, это нормально | При интеграции с реальным Компасом замените вызовы `emulator-client` на реальные COM-интерфейсы |

---

### Вопросы для самопроверки

1. Объясните, почему цикл «LLM → вызов инструмента → результат → LLM» может быть бесконечным и как его контролировать.
2. Зачем агенту системный промпт, подробно описывающий порядок действий? Что произойдёт, если его убрать?
3. Каким образом агент обрабатывает ситуацию, когда инструмент возвращает ошибку? Как это помогает LLM скорректироваться?
4. Предложите способ, как агент может уточнить недостающие параметры (например, пользователь забыл указать формат).
5. Почему мы передаём функции в виде полной схемы (`parameters`), а не просто списком названий?

### Практическое задание

1. Доработайте агента так, чтобы он мог принимать несколько последовательных заданий в рамках одного сеанса (интерактивный диалог), не перезапуская процессы.
2. Реализуйте возможность прерывания цикла, если LLM повторяет один и тот же ошибочный вызов трижды — агенту следует выдать сообщение и завершить работу.

---

## Итоговый проект по блоку

В качестве финального задания вы расширите систему до работы со сборками.

### Домашнее задание: сборки и простые конструкции

1. **Расширьте эмулятор**  
   Добавьте конечные точки:  
   - `POST /assemblies` — создать сборку (возвращает `assemblyId`).  
   - `POST /assemblies/:id/insert` — вставить существующую деталь (по `documentId`) с указанием позиции (x,y,z). Возвращает `instanceId`.  
   - `GET /assemblies/:id/export?format=step` — экспортировать сборку (эмулятор объединяет имена деталей в файл-заглушку).

2. **Расширьте MCP-сервер**  
   Добавьте инструменты `create_assembly` и `insert_part`, оборачивающие новые эндпоинты.

3. **Модифицируйте агента**  
   Пусть он умеет интерпретировать команды вида: «Создай сборку из куба 10×10×10 и цилиндра диаметром 5, высотой 20, расположенных рядом».  
   Агент должен создать две детали (куб и цилиндр), затем сборку, вставить их в неё с разумными координатами и экспортировать сборку в STEP.

4. **Бонусное задание (по желанию)**  
   Добавьте в эмулятор и MCP операцию «вырезание по эскизу» для детали, чтобы можно было создавать отверстия. Протестируйте, поручив агенту сделать пластину с четырьмя отверстиями по углам.

### Критерии приёмки

- Эмулятор стабильно обрабатывает запросы сборок.
- MCP-сервер предоставляет новые инструменты, они корректно тестируются через MCP Inspector.
- Агент-конструктор успешно выполняет сложные сценарии, используя все доступные инструменты, и сохраняет результирующий файл.
- Все части системы покрыты интеграционными тестами (эмулятор, MCP-сервер, агент — smoke-тест).

---

Поздравляем! Вы создали полноценного AI-ассистента для трёхмерного моделирования. Полученные навыки проектирования MCP-серверов, работы с эмуляторами внешних систем и построения агентов станут фундаментом для реальных промышленных интеграций. В следующем блоке мы объединим всё в терминального оркестратора, управляющего несколькими MCP-серверами одновременно.