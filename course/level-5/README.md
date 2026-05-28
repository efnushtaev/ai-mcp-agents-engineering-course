# Блок 5. MCP-сервер для лазерного гравировщика CarverAll 15Pro

## О блоке
Вы уже умеете оборачивать сложные инженерные системы в MCP (Блок 4 — эмулятор Компас-3D) и выстраивать цикл Tool Calling с LLM (Блок 2). Теперь шагнём в цех — научимся управлять производственным оборудованием с длительными задачами, асинхронным мониторингом и интеграцией в агентную логику.  
Мы разработаем полноценный MCP-сервер для гипотетического лазерного гравировщика **CarverAll 15Pro**, покроем эмулятором всё поведение реального станка и создадим агента-оператора, который принимает команды на естественном языке, контролирует выполнение и умеет ждать завершения гравировки.

**Глобальная цель** — сформировать у вас навык проектирования MCP-серверов для длительных операций, использования ресурсов для реактивного мониторинга и построения Production-Ready агентов, способных заменить оператора на реальном производстве.

---

## Модуль 5.1. Проектирование инструментов и эмулятор гравировщика

### 5.1.1 Теория

**Характеристики CarverAll 15Pro (условные)**
- Принимает задания в форматах **G-код** или **SVG**.
- Параметры лазера: мощность (%), скорость перемещения головы (мм/с), количество проходов.
- Учитывает материал и толщину заготовки.
- Поддерживает состояния: `QUEUED`, `RUNNING`, `PAUSED`, `COMPLETED`, `STOPPED`.
- Длительность операции варьируется от секунд до десятков минут, прогресс отслеживается в процентах.

**Зачем снова эмулятор?**
Реальный станок не всегда под рукой, а отлаживать протокол MCP и поведение LLM‑агента на «живом» железе — дорого и рискованно. Эмулятор воспроизводит полный жизненный цикл задания, позволяет гонять автотесты и тренировать агента без привязки к цеху.

**Проектируем HTTP API эмулятора на Fastify**
Мы создадим standalone-сервис, имитирующий встроенный контроллер станка. Все эндпоинты принимают и возвращают JSON.

| Метод | Путь                 | Описание                                                                 |
|-------|----------------------|--------------------------------------------------------------------------|
| POST  | `/jobs`              | Загрузить файл задания (multipart или JSON с содержимым SVG/G-кода), указать `material`, `thickness`. Возвращает `jobId` и статус `QUEUED`. |
| PUT   | `/jobs/:id/params`   | Установить параметры лазера: `power`, `speed`, `passes`.                  |
| POST  | `/jobs/:id/start`    | Запустить выполнение. Статус → `RUNNING`.                                 |
| POST  | `/jobs/:id/pause`    | Приостановить. Статус → `PAUSED`.                                         |
| POST  | `/jobs/:id/resume`   | Возобновить. Статус → `RUNNING`.                                          |
| POST  | `/jobs/:id/stop`     | Остановить. Статус → `STOPPED`.                                           |
| GET   | `/jobs/:id/status`   | Текущий статус, `progress` (0-100), расчётное оставшееся время `estimatedTimeLeft`. |

**Эмуляция длительного процесса**
При старте задания запускается интервальный таймер (например, каждые 200 мс), который увеличивает прогресс. При достижении 100% статус автоматически переходит в `COMPLETED`. Если срабатывает пауза — таймер останавливается, при возобновлении — запускается снова. Остановка сбрасывает прогресс и удаляет таймер.

**Хранение состояния**
- В памяти: Map `jobs` с объектами состояния.
- Файлы заданий сохраняются во временной директории `tmp/uploads` (для возможности повторного использования). В эмуляторе мы не будем реально обрабатывать SVG/G-код, только логировать факт загрузки.

### 5.1.2 Практика

#### Инициализация проекта
```bash
mkdir carverall-emulator && cd carverall-emulator
npm init -y
npm i fastify zod fastify-zod @fastify/multipart
npm i -D typescript @types/node tsx
npx tsc --init
```
`package.json` добавим скрипт `"dev": "tsx watch src/index.ts"`.

#### Реализация эмулятора (основной код)
Создадим `src/index.ts`:

```typescript
import Fastify from 'fastify';
import { z } from 'zod';
import { randomUUID } from 'node:crypto';
import { join } from 'node:path';
import { mkdirSync, writeFileSync } from 'node:fs';

// ---- Zod-схемы для валидации ----
const CreateJobSchema = z.object({
  material: z.string().min(1),
  thickness: z.number().positive(),
  fileContent: z.string(), // содержимое файла (упрощённо, строка)
});

const ParamsSchema = z.object({
  power: z.number().min(0).max(100),
  speed: z.number().positive(),
  passes: z.number().int().positive(),
});

// ---- Типы ----
interface Job {
  id: string;
  material: string;
  thickness: number;
  filePath: string;
  status: 'QUEUED' | 'RUNNING' | 'PAUSED' | 'COMPLETED' | 'STOPPED';
  progress: number;
  power?: number;
  speed?: number;
  passes?: number;
  timer?: NodeJS.Timeout;
}

const jobs = new Map<string, Job>();
const uploadDir = join(process.cwd(), 'tmp', 'uploads');
mkdirSync(uploadDir, { recursive: true });

const fastify = Fastify({ logger: true });

// ---- Эндпоинты ----

// Загрузка задания
fastify.post('/jobs', async (req, reply) => {
  const body = CreateJobSchema.parse(req.body);
  const jobId = randomUUID();
  const filePath = join(uploadDir, `${jobId}.svg`);
  writeFileSync(filePath, body.fileContent, 'utf-8');

  const job: Job = {
    id: jobId,
    material: body.material,
    thickness: body.thickness,
    filePath,
    status: 'QUEUED',
    progress: 0,
  };
  jobs.set(jobId, job);
  return { jobId, status: 'QUEUED' };
});

// Установка параметров
fastify.put('/jobs/:id/params', async (req, reply) => {
  const { id } = req.params as { id: string };
  const job = jobs.get(id);
  if (!job) return reply.status(404).send({ error: 'Job not found' });
  const params = ParamsSchema.parse(req.body);
  Object.assign(job, params);
  return { success: true };
});

// Запуск
fastify.post('/jobs/:id/start', async (req, reply) => {
  const { id } = req.params as { id: string };
  const job = jobs.get(id);
  if (!job) return reply.status(404).send({ error: 'Job not found' });
  if (job.status !== 'QUEUED') return reply.status(409).send({ error: 'Job cannot be started' });
  // Запускаем эмуляцию прогресса
  job.status = 'RUNNING';
  job.progress = 0;
  job.timer = setInterval(() => {
    if (job.status !== 'RUNNING') return; // если на паузе или остановлен
    job.progress = Math.min(100, job.progress + 1);
    if (job.progress >= 100) {
      clearInterval(job.timer!);
      job.status = 'COMPLETED';
    }
  }, 200);
  return { success: true, status: 'RUNNING' };
});

// Пауза
fastify.post('/jobs/:id/pause', async (req, reply) => {
  const { id } = req.params as { id: string };
  const job = jobs.get(id);
  if (!job || job.status !== 'RUNNING') return reply.status(409).send({ error: 'Cannot pause' });
  if (job.timer) clearInterval(job.timer);
  job.status = 'PAUSED';
  return { success: true, status: 'PAUSED' };
});

// Возобновление
fastify.post('/jobs/:id/resume', async (req, reply) => {
  const { id } = req.params as { id: string };
  const job = jobs.get(id);
  if (!job || job.status !== 'PAUSED') return reply.status(409).send({ error: 'Cannot resume' });
  job.status = 'RUNNING';
  job.timer = setInterval(() => {
    if (job.status !== 'RUNNING') return;
    job.progress = Math.min(100, job.progress + 1);
    if (job.progress >= 100) {
      clearInterval(job.timer!);
      job.status = 'COMPLETED';
    }
  }, 200);
  return { success: true, status: 'RUNNING' };
});

// Остановка
fastify.post('/jobs/:id/stop', async (req, reply) => {
  const { id } = req.params as { id: string };
  const job = jobs.get(id);
  if (!job) return reply.status(404).send();
  if (job.timer) clearInterval(job.timer);
  job.status = 'STOPPED';
  return { success: true, status: 'STOPPED' };
});

// Статус
fastify.get('/jobs/:id/status', async (req, reply) => {
  const { id } = req.params as { id: string };
  const job = jobs.get(id);
  if (!job) return reply.status(404).send({ error: 'Job not found' });
  const remainingSteps = 100 - job.progress;
  // примерно 200мс на шаг
  const estimatedTimeLeft = job.status === 'RUNNING' ? remainingSteps * 200 : 0;
  return {
    jobId: job.id,
    status: job.status,
    progress: job.progress,
    estimatedTimeLeft,
    material: job.material,
    thickness: job.thickness,
  };
});

// Старт сервера
fastify.listen({ port: 3100 }, (err, address) => {
  if (err) throw err;
  console.log(`CarverAll emulator listening on ${address}`);
});
```

**Тесты через curl**
1. Создать задание:
```bash
curl -X POST http://localhost:3100/jobs \
  -H "Content-Type: application/json" \
  -d '{"material":"фанера","thickness":3,"fileContent":"<svg>...</svg>"}'
# → {"jobId":"...","status":"QUEUED"}
```
2. Установить параметры, запустить, проверить статус, поставить на паузу, возобновить, остановить.

### 5.1.3 Типичные ошибки и их решение

1. **Потеря таймера при паузе/возобновлении**  
   Если не очистить интервал перед созданием нового, получится несколько параллельных таймеров, прогресс будет прыгать. Всегда вызывайте `clearInterval` и проверяйте, что `job.timer` не активен.

2. **Гонка состояний при быстрых запросах**  
   Остановка или пауза могут прийти, когда прогресс уже 100% и таймер завершился, но статус ещё не обновлён. Добавляйте защитные условия: внутри колбэка таймера проверяйте `if (job.status !== 'RUNNING') return;`.

3. **Zod-валидация падает без подробного ответа**  
   Fastify по умолчанию возвращает 500. Оберните парсинг в try/catch или используйте `fastify-zod` для автоматических ответов 400 с деталями ошибок.

4. **Забыли про `estimatedTimeLeft` = 0 для неактивных заданий**  
   Агент может решить, что задание завершено, если увидит нулевое время, когда статус ещё QUEUED. Явно возвращайте 0 для не-RUNNING.

### 5.1.4 Вопросы для самопроверки
1. Почему мы выбрали Fastify для эмулятора, а не express? Какие преимущества дают встроенная валидация и высокая производительность?
2. Как бы вы изменили таймер, чтобы прогресс рос нелинейно, имитируя реальную гравировку сложных участков?
3. Какие HTTP-статусы вернуть, если задание уже завершено, а клиент пытается его запустить?
4. Что произойдёт, если два клиента одновременно вызовут pause и resume? Как обеспечить атомарность?
5. Предложите способ сохранять логи выполненных заданий для последующего аудита.

### 5.1.5 Практическое задание к модулю 5.1
Добавьте в эмулятор эндпоинт `GET /jobs/:id/gcode`, который возвращает сгенерированный по SVG примитивный G-код (заглушку — просто строку с комментариями). Это понадобится в итоговом домашнем задании.

---

## Модуль 5.2. Реализация MCP-сервера carverall-mcp

### 5.2.1 Теория

Мы обернём HTTP API эмулятора в MCP-инструменты и добавим **ресурс** для реактивного мониторинга состояния задания. Студенты вспомнят, что ресурс — это адресуемый URI, содержимое которого сервер может обновлять и рассылать уведомления клиентам.

**Список инструментов (Tools)**
Каждый инструмент маппится на соответствующий эндпоинт эмулятора:
- `upload_job` — принимает путь к локальному файлу (содержимое читаем на стороне сервера), `material`, `thickness`.
- `set_laser_params` — `jobId`, `power`, `speed`, `passes`.
- `start_job`, `pause_job`, `resume_job`, `stop_job` — идентификатор задания.
- `get_job_status` — возвращает полный объект статуса.

**Ресурс `job://{jobId}/status`**
Сервер регистрирует ресурс с URI `job://<id>/status`. При вызове `resources/read` отдаётся текстовое представление JSON-статуса (можно просто сериализовать). При изменении статуса (запуск, прогресс, завершение) сервер отправляет уведомление `notifications/resources/updated`.  
Для упрощения будем опрашивать эмулятор внутри сервера с интервалом (например, 2 секунды) для активных заданий и сравнивать с кэшем; если изменилось — уведомляем.

**Транспорт** — `stdio`, чтобы агент мог запускать сервер как дочерний процесс и общаться через стандартные потоки.

### 5.2.2 Практика

#### Инициализация проекта
```bash
mkdir carverall-mcp && cd carverall-mcp
npm init -y
npm i @modelcontextprotocol/sdk zod undici
npm i -D typescript @types/node tsx
npx tsc --init
```
Скрипт `"dev": "tsx src/index.ts"`.

#### Реализация сервера
Создадим `src/index.ts`. Используем `McpServer` из SDK, инструменты и ресурсы.

```typescript
import { McpServer, ResourceTemplate } from '@modelcontextprotocol/sdk/server/mcp.js';
import { StdioServerTransport } from '@modelcontextprotocol/sdk/server/stdio.js';
import { z } from 'zod';
import { request } from 'undici';

// Базовый URL эмулятора (можно передавать через env)
const EMULATOR_URL = process.env.EMULATOR_URL || 'http://localhost:3100';

// Вспомогательная функция вызова API
async function callApi(path: string, method: string, body?: any) {
  const { body: resBody, statusCode } = await request(EMULATOR_URL + path, {
    method,
    headers: { 'Content-Type': 'application/json' },
    body: body ? JSON.stringify(body) : undefined,
  });
  if (statusCode < 200 || statusCode >= 300) {
    const errText = await resBody.text();
    throw new Error(`API error ${statusCode}: ${errText}`);
  }
  return resBody.json();
}

// Кэш состояний для отслеживания изменений (ресурс)
const jobStatusCache = new Map<string, any>();

// Создаём сервер с идентификатором и названием
const server = new McpServer({
  name: 'carverall-mcp',
  version: '1.0.0',
});

// ---- Регистрируем инструменты ----

server.tool(
  'upload_job',
  'Загрузить SVG/G-код задание для гравировки',
  {
    filePath: z.string().describe('Локальный путь к файлу задания'),
    material: z.string().describe('Материал заготовки'),
    thickness: z.number().positive().describe('Толщина в мм'),
  },
  async ({ filePath, material, thickness }) => {
    // Читаем содержимое файла (в синхронном варианте для простоты)
    const fs = await import('fs/promises');
    const content = await fs.readFile(filePath, 'utf-8');
    const result = await callApi('/jobs', 'POST', {
      material,
      thickness,
      fileContent: content,
    });
    return {
      content: [{ type: 'text', text: JSON.stringify(result) }],
    };
  }
);

server.tool(
  'set_laser_params',
  'Настроить мощность, скорость и число проходов лазера',
  {
    jobId: z.string(),
    power: z.number().min(0).max(100),
    speed: z.number().positive(),
    passes: z.number().int().positive(),
  },
  async ({ jobId, power, speed, passes }) => {
    await callApi(`/jobs/${jobId}/params`, 'PUT', { power, speed, passes });
    return {
      content: [{ type: 'text', text: 'Параметры лазера установлены' }],
    };
  }
);

// Аналогично остальные инструменты (start_job, pause_job и т.д.)
// Приведу пример start_job:

server.tool(
  'start_job',
  'Запустить выполнение задания',
  { jobId: z.string() },
  async ({ jobId }) => {
    const result = await callApi(`/jobs/${jobId}/start`, 'POST');
    // Инициируем отслеживание изменений для ресурса
    startMonitoring(jobId);
    return {
      content: [{ type: 'text', text: `Задание ${jobId} запущено` }],
    };
  }
);

server.tool(
  'pause_job',
  'Приостановить гравировку',
  { jobId: z.string() },
  async ({ jobId }) => {
    await callApi(`/jobs/${jobId}/pause`, 'POST');
    stopMonitoring(jobId); // останавливаем опрос на паузе
    return {
      content: [{ type: 'text', text: `Задание ${jobId} приостановлено` }],
    };
  }
);

server.tool(
  'resume_job',
  'Возобновить гравировку после паузы',
  { jobId: z.string() },
  async ({ jobId }) => {
    await callApi(`/jobs/${jobId}/resume`, 'POST');
    startMonitoring(jobId);
    return {
      content: [{ type: 'text', text: `Задание ${jobId} возобновлено` }],
    };
  }
);

server.tool(
  'stop_job',
  'Остановить выполнение задания',
  { jobId: z.string() },
  async ({ jobId }) => {
    await callApi(`/jobs/${jobId}/stop`, 'POST');
    stopMonitoring(jobId);
    return {
      content: [{ type: 'text', text: `Задание ${jobId} остановлено` }],
    };
  }
);

server.tool(
  'get_job_status',
  'Получить текущий статус, прогресс и оставшееся время задания',
  { jobId: z.string() },
  async ({ jobId }) => {
    const status = await callApi(`/jobs/${jobId}/status`, 'GET');
    return {
      content: [{ type: 'text', text: JSON.stringify(status, null, 2) }],
    };
  }
);

// ---- Ресурс: job://{jobId}/status ----

// Хранилище таймеров мониторинга
const monitors = new Map<string, NodeJS.Timeout>();

function startMonitoring(jobId: string) {
  if (monitors.has(jobId)) return;
  const timer = setInterval(async () => {
    try {
      const newStatus = await callApi(`/jobs/${jobId}/status`, 'GET');
      const oldStatus = jobStatusCache.get(jobId);
      jobStatusCache.set(jobId, newStatus);
      if (!oldStatus || JSON.stringify(oldStatus) !== JSON.stringify(newStatus)) {
        // Отправляем уведомление об изменении ресурса
        server.notification({
          method: 'notifications/resources/updated',
          params: { uri: `job://${jobId}/status` },
        });
      }
      // Если задание завершено или остановлено, прекращаем мониторинг
      if (newStatus.status === 'COMPLETED' || newStatus.status === 'STOPPED') {
        stopMonitoring(jobId);
      }
    } catch (err) {
      console.error(`Monitoring error for ${jobId}:`, err);
      stopMonitoring(jobId);
    }
  }, 2000);
  monitors.set(jobId, timer);
}

function stopMonitoring(jobId: string) {
  const timer = monitors.get(jobId);
  if (timer) {
    clearInterval(timer);
    monitors.delete(jobId);
  }
}

// Регистрация ресурса
server.resource(
  'job-status',
  new ResourceTemplate('job://{jobId}/status', { list: undefined }),
  async (uri, { jobId }) => {
    // При чтении отдаём кэшированное или свежее значение
    let status = jobStatusCache.get(jobId);
    if (!status) {
      status = await callApi(`/jobs/${jobId}/status`, 'GET');
      jobStatusCache.set(jobId, status);
    }
    return {
      contents: [{
        uri: uri.href,
        text: JSON.stringify(status, null, 2),
      }],
    };
  }
);

// Запуск сервера
const transport = new StdioServerTransport();
await server.connect(transport);
console.error('CarverAll MCP server running via stdio');
```

**Тестирование через MCP Inspector**
```bash
npx @modelcontextprotocol/inspector tsx src/index.ts
```
Проверьте список инструментов, загрузите тестовый SVG, установите параметры, запустите и наблюдайте, как ресурс `job://<id>/status` обновляется.

### 5.2.3 Типичные ошибки и их решение

1. **Сервер не успевает обработать уведомление ресурса — проверьте асинхронность**  
   `setInterval` внутри MCP-сервера не должен блокировать основной поток. Используйте асинхронные колбэки и `await` для запросов. SDK корректно обрабатывает `sendNotification`.

2. **Утечка таймеров мониторинга**  
   Если клиент отключается, а таймеры продолжают слать уведомления, это ведёт к утечке памяти. В реальном приложении очищайте все интервалы в хуке `server.onclose` или при завершении процесса.

3. **Ресурс не отображается в инспекторе**  
   Убедитесь, что `ResourceTemplate` правильно определяет `list` (можно передать `undefined` или функцию, возвращающую список URI). Если URI не перечислен, клиент может не подписаться.

4. **Ошибка «URI already registered»**  
   Нельзя регистрировать один и тот же URI дважды. Если хотите несколько заданий, используйте шаблон с параметром `{jobId}`.

5. **Забыли очистить кэш `jobStatusCache` после остановки**  
   Может привести к тому, что при следующем запуске того же `jobId` агент получит старые данные. Сбрасывайте кэш при остановке или перезапуске задания.

### 5.2.4 Вопросы для самопроверки
1. Почему для ресурса мы выбрали push-модель с уведомлениями, а не только pull через `resources/read`?
2. Какие преимущества даёт использование `undici` перед `node-fetch` в контексте MCP-сервера?
3. Как можно расширить ресурс, чтобы отдавать не только статус, но и историю изменения прогресса (например, в формате SSE)?
4. Какие ещё URI ресурсов вы бы предложили для гравировщика? (превью файла, лог ошибок, температура лазера и т.д.)
5. Как обеспечить graceful shutdown MCP-сервера, чтобы все таймеры были остановлены?

### 5.2.5 Практическое задание к модулю 5.2
Добавьте инструмент `list_jobs` (GET /jobs), возвращающий список всех заданий с их статусами. Реализуйте ресурс `jobs://list`, который автоматически уведомляет при появлении новых или изменении статуса любого задания.

---

## Модуль 5.3. Интеграция с LLM и создание агента-оператора станка

### 5.3.1 Теория

Теперь объединим эмулятор, MCP-сервер и LLM в единого агента, который принимает команды на естественном языке и управляет гравировкой. Мы используем `ToolCallingClient` из Блока 2, который поддерживает DeepSeek и YandexGPT.

**Системный промпт агента-оператора**
```
Ты — оператор лазерного гравировального станка CarverAll 15Pro. Твои возможности:
- Загружать задания (SVG/G-код) с указанием материала и толщины.
- Настраивать мощность лазера (0-100%), скорость (мм/с) и количество проходов.
- Запускать, приостанавливать, возобновлять и останавливать выполнение.
- Проверять статус задания, прогресс и расчётное время завершения.
Общайся с пользователем, выполняй его указания, предупреждай о недопустимых параметрах. Если запустил задание, отслеживай статус, пока не завершится, и сообщи результат.
```

**Цикл агента для длительных задач**
Особенность в том, что после вызова `start_job` агенту нужно дождаться завершения гравировки, прежде чем отвечать пользователю. Есть два подхода:
- **Активный поллинг** (наш выбор для учебного примера): после `start_job` агент периодически вызывает `get_job_status` с фиксированной задержкой (например, 2 с), пока статус не станет `COMPLETED` или `STOPPED`. Такой подход прост и не требует поддержки уведомлений на клиенте.
- **Подписка на ресурс**: клиент подписывается на `notifications/resources/updated`, но наша текущая реализация `ToolCallingClient` может не иметь встроенной обработки нотификаций. Поэтому пока остановимся на поллинге.

**Обработка ошибок**
- Если файл не найден на диске — инструмент `upload_job` выбросит исключение, LLM получит сообщение об ошибке и должна сообщить пользователю.
- Неверные параметры (мощность > 100%) — Zod отклонит вызов, агент должен скорректировать.
- Станок аварийно остановился (эмулятор вернёт статус `STOPPED`) — агент информирует пользователя и предлагает дальнейшие действия.

### 5.3.2 Практика

#### Сценарий агента
Напишем `agent-operator.ts`. Он будет:
1. Запускать эмулятор (если не запущен) и MCP-сервер как дочерние процессы (или подключаться к уже работающим).
2. Инициализировать MCP-клиент через `StdioClientTransport`.
3. Использовать `ToolCallingClient` для общения с LLM.
4. В цикле обрабатывать запросы пользователя, включая автоматический поллинг после старта задания.

**Код агента** (упрощённая версия, интегрирующая концепции из Блока 2)

```typescript
import { StdioClientTransport } from '@modelcontextprotocol/sdk/client/stdio.js';
import { Client } from '@modelcontextprotocol/sdk/client/index.js';
import { ToolCallingClient } from './tool-calling-client'; // Ваш клиент из Блока 2
import { spawn } from 'child_process';
import readline from 'readline';

const EMULATOR_CMD = 'npx tsx ../carverall-emulator/src/index.ts';
const MCP_SERVER_CMD = 'npx tsx ../carverall-mcp/src/index.ts';

async function startProcess(command: string) {
  const [cmd, ...args] = command.split(' ');
  const child = spawn(cmd, args, { stdio: 'pipe', shell: true });
  child.stderr.on('data', (data) => console.error(`[process] ${data}`));
  return child;
}

async function main() {
  console.log('Starting emulator and MCP server...');
  const emulator = await startProcess(EMULATOR_CMD);
  // Даём время на запуск
  await new Promise(resolve => setTimeout(resolve, 2000));

  const mcpProcess = await startProcess(MCP_SERVER_CMD);
  await new Promise(resolve => setTimeout(resolve, 1000));

  // Создаём MCP-клиент
  const transport = new StdioClientTransport({
    command: MCP_SERVER_CMD.split(' ')[0],
    args: MCP_SERVER_CMD.split(' ').slice(1),
  });
  const client = new Client({ name: 'agent-operator', version: '1.0.0' });
  await client.connect(transport);

  // Список инструментов с маппингом на MCP-вызовы
  const tools = [
    { name: 'upload_job', /* ... */ },
    // ... полный список инструментов
  ];

  const toolCallingClient = new ToolCallingClient({
    provider: 'deepseek', // или 'yandexgpt'
    model: 'deepseek-chat',
    tools,
    systemPrompt: `Ты — оператор лазерного гравировального станка CarverAll 15Pro...`,
  });

  // Функция поллинга статуса задания
  async function waitForCompletion(jobId: string) {
    while (true) {
      const statusResult = await client.callTool({
        name: 'get_job_status',
        arguments: { jobId },
      });
      const status = JSON.parse(statusResult.content[0].text);
      console.log(`[Progress] ${status.progress}% - Status: ${status.status}`);
      if (status.status === 'COMPLETED' || status.status === 'STOPPED') {
        return status;
      }
      await new Promise(resolve => setTimeout(resolve, 2000));
    }
  }

  // Диалоговый цикл
  const rl = readline.createInterface({ input: process.stdin, output: process.stdout });
  const askQuestion = () => new Promise<string>(resolve => rl.question('> ', resolve));

  while (true) {
    const userInput = await askQuestion();
    if (userInput === 'exit') break;

    // Отправляем запрос LLM, получаем последовательность tool calls
    const llmResponse = await toolCallingClient.chat(userInput);
    // Обрабатываем вызовы инструментов
    for (const call of llmResponse.toolCalls) {
      const result = await client.callTool({
        name: call.function.name,
        arguments: JSON.parse(call.function.arguments),
      });
      // Если запустили задание, автоматически начинаем поллинг
      if (call.function.name === 'start_job') {
        const { jobId } = JSON.parse(call.function.arguments);
        const finalStatus = await waitForCompletion(jobId);
        console.log(`Задание завершено со статусом: ${finalStatus.status}`);
        // Можно отправить результат обратно в LLM для формирования ответа
      }
    }
  }

  rl.close();
  await client.close();
  emulator.kill();
  mcpProcess.kill();
}

main().catch(console.error);
```

**Тестовые сценарии**
1. «Загрузи файл logo.svg для фанеры 3 мм, установи мощность 60%, скорость 100 мм/с, 2 прохода, и запусти гравировку».  
   Агент вызовет `upload_job`, `set_laser_params`, `start_job`, и будет выводить прогресс каждые 2 секунды, пока не завершится.

2. «Поставь на паузу текущее задание, измени мощность на 70% и продолжи».  
   Агент должен запомнить последний `jobId` (нужно доработать хранение контекста) или попросить уточнить ID, вызвать `pause_job`, `set_laser_params`, `resume_job`.

3. «Останови задание и сообщи причину».  
   Агент вызывает `stop_job` и выводит сообщение.

**Демонстрация прогресса в консоли**
Функция `waitForCompletion` печатает прогресс, а в реальном UI можно использовать progress bar.

### 5.3.3 Типичные ошибки и их решение

1. **Поллинг зависает, если не ограничить таймаут**  
   Если станок завис в статусе RUNNING без прогресса, поллинг никогда не закончится. Добавьте максимальное время ожидания и проверку на отсутствие прогресса за N итераций. При таймауте выбрасывайте ошибку.

2. **Конкуренция за стандартный ввод/вывод с MCP-сервером**  
   Если агент и сервер общаются через stdio, нельзя одновременно писать в консоль readline и ожидать ответа от сервера. Используйте отдельные процессы или pipe для MCP транспорта (как в коде, сервер запускается отдельно, а клиент подключается по stdio к нему).

3. **LLM «забывает» `jobId` между вызовами**  
   Необходимо либо передавать `jobId` в ответах, либо хранить его в контексте диалога. Можно после `upload_job` просить LLM явно вернуть идентификатор, а затем при следующих командах либо требовать от пользователя указать ID, либо сохранять в переменной агента и подставлять, если не указан.

4. **Ошибка парсинга JSON из ответа инструмента**  
   Всегда оборачивайте `JSON.parse` в try/catch, так как модель может сгенерировать не совсем валидный JSON для аргументов. Используйте режим `JSON mode`, если поддерживается.

5. **MCP-сервер падает при отсутствии эмулятора**  
   Добавьте повторные попытки подключения или healthcheck перед вызовом инструментов, чтобы агент мог сообщить «Станок не отвечает».

### 5.3.4 Вопросы для самопроверки
1. В чём преимущество и недостатки поллинга по сравнению с подпиской на ресурс в контексте агента-оператора?
2. Как можно оптимизировать поллинг, чтобы снизить нагрузку на эмулятор при большом количестве заданий?
3. Предложите стратегию обработки аварийной остановки станка (например, по сигналу от датчика) и уведомления агента об этом через MCP.
4. Как задействовать `ToolCallingClient` для продолжения диалога после завершения поллинга, чтобы агент мог ответить пользователю естественным языком, включив финальную статистику?
5. Какие метрики и логи стоит собирать в реальном агенте для аудита действий оператора?

### 5.3.5 Практическое задание к модулю 5.3
Модифицируйте агента так, чтобы он поддерживал несколько одновременно запущенных заданий (массив `jobId`) и мог по запросу «как там мои задания?» вывести статусы всех активных процессов.

---

## Итоговое домашнее задание по Блоку 5

**Цель**: углубить понимание производственной цепочки и расширить возможности агента.

1. **Добавьте в эмулятор генерацию примитивного G-кода**  
   При загрузке SVG (хотя бы по заглушке) эмулятор должен создавать массив «проходов», имитирующий G-код. При выполнении задания в лог (консоль эмулятора) выводите текущий проход и команду. Так вы сможете визуально отслеживать гравировку.

2. **Научите агента формировать G-код по текстовому описанию фигур**  
   Расширьте системный промпт: «Если пользователь просит гравировать простую фигуру, сгенерируй соответствующий G-код и сохрани во временный файл». Например, команда «Выгравируй квадрат 20×20 мм на фанере 3 мм» должна привести к созданию файла с командами G-кода для квадрата и последующей загрузке через `upload_job`. Агент должен сам сформировать файл и передать путь.

3. **Реализуйте поддержку реального SVG через `sharp` или `svg-parser`**  
   Добавьте возможность преобразования контура SVG в набор координат и генерации G-кода обхода. Хотя бы для простых фигур (rect, circle). Это максимально приблизит эмулятор к реальному станку.

4. **Внедрите мониторинг температуры и защиту**  
   Добавьте к эмулятору фиктивный датчик температуры лазера. Если температура превышает порог, задание должно ставиться на паузу с соответствующим статусом. MCP-сервер должен иметь инструмент `get_temperature`, а агент — уметь реагировать на перегрев.

5. **Создайте Docker-контейнеры** для эмулятора и MCP-сервера, чтобы оркестратор мог разворачивать их одной командой.

**Критерии оценки**:
- Эмулятор корректно отрабатывает жизненный цикл, включая генерацию G-кода.
- MCP-сервер стабилен, ресурсы уведомляют об изменениях.
- Агент успешно выполняет сценарии «загрузка SVG», «генерация G-кода по описанию», «контроль температуры».
- Код типизирован, покрыт комментариями, ошибки обрабатываются.

---

Вы прошли путь от «голого» эмулятора до агента-оператора производственного станка, полностью интегрированного в AI-контур. Теперь вы готовы к ещё более сложным системам, где MCP-серверы управляют целыми линиями IoT. Увидимся в следующем блоке!