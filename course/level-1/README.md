# Блок 1. Фундамент: LLM, промпты и вызов инструментов

Добро пожаловать в самое сердце AI-инжиниринга. Ты — опытный Node.js/TypeScript-разработчик, а значит, у тебя уже есть главное: умение строить надёжные системы, работать с API и понимание асинхронного кода. Этот блок даст тебе фундамент, без которого невозможна дальнейшая работа с MCP-серверами и терминальным оркестратором. Мы пройдём путь от первого запроса к языковой модели до реализации полноценного цикла вызова внешних инструментов (Function Calling) на двух open-source гигантах: DeepSeek и YandexGPT.

К концу блока ты напишешь TypeScript-код, который не просто «болтает» с LLM, а умеет выполнять реальные действия: читать чертежи, опрашивать станки и конвертировать единицы измерения.

---

## Модуль 1.1. Введение в AI-инжиниринг и LLM

### Теория

#### Кто такой AI-инженер и почему ты?
AI-инжиниринг — это дисциплина, которая соединяет готовые большие языковые модели (LLM) с реальными программными системами. В отличие от Data Science, мы не обучаем модели с нуля и не копаемся в математике градиентного спуска. В отличие от классического ML-инжиниринга, мы не пакуем модели в Docker и не строим пайплайны переобучения. Наша задача — заставить LLM безопасно и предсказуемо взаимодействовать с кодом, базами данных, API и железом.

Node.js/TypeScript-разработчик — идеальный кандидат на эту роль: ты привык к асинхронности, умеешь проектировать контракты (типы!) и работать с сетевыми протоколами. LLM для тебя — это просто ещё один асинхронный источник данных, который иногда требует особого обращения.

#### Как работает LLM на пальцах (для бэкендера)
Представь себе LLM как очень сложную функцию `complete(prompt: string): string`, но с нюансами.

1. **Токенизация.** Текст перед подачей в модель разбивается на токены — кусочки слов или символы. Например, «Привет, мир!» превращается в что-то вроде `["При", "вет", ",", " мир", "!"]`. Для английского языка 1 токен ≈ 0.75 слова, для русского — примерно 1 токен ≈ 1.5–2 символа. Именно токенами измеряется и длина контекста, и стоимость.

2. **Transformer (упрощённо).** Внутри модели находится механизм внимания, который для каждого токена предсказывает следующий, глядя на все предыдущие токены в окне. Архитектура Transformer позволяет эффективно учитывать дальние связи в тексте. Никакой магии — просто миллиарды матричных умножений, оптимизированных под GPU.

3. **Контекстное окно** — это максимальное количество токенов, которое модель может «видеть» одновременно. Сумма твоего промпта, истории диалога и ответа модели не должна превышать этот лимит. Для DeepSeek V3 это 128K токенов, для YandexGPT 5 — 32K (уточняйте в документации). При превышении окна обрезается начало истории.

4. **Температура** — параметр от 0 до 2 (обычно), управляющий хаотичностью ответа.
   - `0` — модель детерминированная: всегда выбирает самый вероятный токен. Идеально для точных команд, генерации кода, извлечения структурированных данных.
   - `0.7–1.0` — сбалансированная креативность.
   - `>1.0` — модель начинает «бредить», слова становятся менее связанными. Для инженерных задач редко применяется.

#### Open-source ландшафт: где брать модели
Мы сознательно используем только бесплатные или open-source модели, чтобы ты мог экспериментировать без лимитов.

- **DeepSeek** (V3, R1). Китайская модель, доступная через API с очень демократичными ценами (на момент написания регистрация даёт бесплатные кредиты). Полностью совместима с OpenAI SDK. API: `api.deepseek.com`. Документация: [platform.deepseek.com](https://platform.deepseek.com/docs).
- **YandexGPT 5**. Российская модель, интегрирована в Yandex Cloud. Для доступа нужен аккаунт в облаке, сервисный аккаунт и API-ключ. Можно использовать REST API напрямую или через официальный Node.js SDK. Документация: [yandex.cloud/ru/docs/foundation-models](https://yandex.cloud/ru/docs/foundation-models).
- Другие достойные варианты: Qwen 2.5, Llama 3 (можно запускать локально через Ollama). В курсе мы концентрируемся на DeepSeek как основном провайдере и YandexGPT как альтернативе, чтобы показать различия в реализации Function Calling.

#### Общение с API: формат сообщений и параметры
Все современные LLM общаются по REST API с JSON-телом. Запрос содержит массив сообщений (`messages`) и параметры генерации. Каждое сообщение имеет роль:

- `system` — задаёт общее поведение, стиль, ограничения. Не обязательно, но крайне полезно.
- `user` — реплика пользователя, инструкция, вопрос.
- `assistant` — предыдущий ответ модели. Нужна для поддержания истории диалога.
- `tool` — результат выполнения функции (вводится позже).

Пример минимального тела запроса к DeepSeek:
```json
{
  "model": "deepseek-chat",
  "messages": [
    {"role": "system", "content": "Ты — помощник инженера-конструктора."},
    {"role": "user", "content": "Рассчитай допуск на вал диаметром 25 мм."}
  ],
  "temperature": 0.3
}
```

Параметры запроса:
- `model` — идентификатор модели (у DeepSeek `deepseek-chat` для V3, `deepseek-reasoner` для R1; у YandexGPT `yandexgpt/latest` или конкретная версия).
- `temperature`, `top_p` — управление случайностью.
- `max_tokens` — максимальная длина ответа в токенах.
- `stream` — включает потоковую передачу ответа (SSE). В нашем блоке пока используем `false`.

### Практика

#### 1. Установка зависимостей
Мы будем использовать два клиента: `openai` (совместим с DeepSeek) и `@yandex-cloud/nodejs-sdk` для YandexGPT. Инициализируем проект и установим пакеты:

```bash
mkdir ai-fundamentals && cd ai-fundamentals
npm init -y
npm install openai @yandex-cloud/nodejs-sdk typescript ts-node @types/node --save
npx tsc --init --target ES2020 --module commonjs --outDir dist
```

Создай файл `.env` (не коммить его!):
```
DEEPSEEK_API_KEY=sk-your-deepseek-key
YANDEX_API_KEY=AQVN...  # IAM-токен или API-ключ сервисного аккаунта
YANDEX_FOLDER_ID=b1g... # ID каталога в Яндекс Облаке
```

Для загрузки переменных окружения удобно использовать `dotenv`:
```bash
npm install dotenv
```

В корне проекта создай `load-env.ts`:
```typescript
import dotenv from 'dotenv';
dotenv.config();
```

#### 2. Первый запрос к DeepSeek API
Создадим файл `src/deepseek-hello.ts`. Подключаем клиент OpenAI с кастомным базовым URL:

```typescript
import OpenAI from 'openai';
import '../load-env';

const deepseek = new OpenAI({
  baseURL: 'https://api.deepseek.com',
  apiKey: process.env.DEEPSEEK_API_KEY!,
});

async function main() {
  const response = await deepseek.chat.completions.create({
    model: 'deepseek-chat',
    messages: [
      { role: 'system', content: 'Ты — ассистент инженера. Отвечай кратко, по делу.' },
      { role: 'user', content: 'Что такое токен в контексте LLM?' },
    ],
    temperature: 0.2,
    max_tokens: 200,
  });

  console.log('Ответ модели:');
  console.log(response.choices[0]?.message?.content);
  
  console.log('\nИспользование токенов:');
  console.log(`  Промпт: ${response.usage?.prompt_tokens}`);
  console.log(`  Завершение: ${response.usage?.completion_tokens}`);
  console.log(`  Всего: ${response.usage?.total_tokens}`);
  
  console.log('Причина остановки:', response.choices[0]?.finish_reason);
}

main().catch(console.error);
```

Запусти:
```bash
npx ts-node src/deepseek-hello.ts
```

Типичные ошибки:
- **401 Unauthorized** — проверь API-ключ, возможно, он пустой или неверный.
- **403 Forbidden** — исчерпаны кредиты на аккаунте DeepSeek, пополни баланс.
- **Connection timeout** — проверь сетевые ограничения, VPN при необходимости.

Анализируем поля:
- `choices[0].message.content` — собственно текст ответа.
- `usage.prompt_tokens` — сколько токенов съел твой запрос вместе с system и историей.
- `completion_tokens` — сколько токенов сгенерировала модель.
- `finish_reason` — `"stop"` (модель закончила сама), `"length"` (упёрлась в `max_tokens`), `"content_filter"` (сработал фильтр безопасности).

#### 3. Аналогичный запрос к YandexGPT
У Yandex Cloud более многословный API. Воспользуемся официальным SDK. Создадим `src/yandexgpt-hello.ts`:

```typescript
import { YandexCloud } from '@yandex-cloud/nodejs-sdk';
import '../load-env';

async function main() {
  const session = new YandexCloud.Session({
    oauthToken: process.env.YANDEX_API_KEY, // или IAM-токен
  });
  const client = session.client(YandexCloud.FoundationModelsService);
  
  const response = await client.completion({
    modelUri: `gpt://${process.env.YANDEX_FOLDER_ID}/yandexgpt/latest`,
    completionOptions: {
      stream: false,
      temperature: 0.2,
      maxTokens: 200,
    },
    messages: [
      { role: 'system', text: 'Ты — ассистент инженера. Отвечай кратко, по делу.' },
      { role: 'user', text: 'Что такое токен в контексте LLM?' },
    ],
  });

  // В ответе лежит массив alternatives
  const alternative = response.alternatives?.[0];
  console.log('Ответ модели:');
  console.log(alternative?.message?.text);
  
  console.log('\nИспользование токенов:');
  console.log(`  Промпт: ${response.usage?.inputTextTokens}`);
  console.log(`  Завершение: ${response.usage?.completionTokens}`);
  console.log(`  Всего: ${response.usage?.totalTokens}`);
  
  console.log('Статус:', alternative?.status);
}

main().catch(console.error);
```

Обрати внимание:
- YandexGPT требует `modelUri` вида `gpt://<folder-id>/yandexgpt/latest`.
- Роли называются так же, но поле с текстом — `text`, а не `content`.
- Статус `ALTERNATIVE_STATUS_FINAL` означает успешное завершение.

Типичные ошибки:
- **`modelUri is invalid`** — проверь folder ID, он должен быть в формате `b1g...`.
- **Аутентификация** — если используешь API-ключ сервисного аккаунта, передавай его в `session.serviceAccountKey`. OAuth токен подходит для личного аккаунта. В нашем курсе рекомендуется создать сервисный аккаунт с ролью `ai.languageModels.user` и использовать API-ключ.
- **Пустой ответ** — возможно, сработал фильтр модерации. Попробуй сменить промпт.

#### Самостоятельное задание (Модуль 1.1)
Создай файл `src/compare-models.ts`, который отправляет один и тот же промпт «Объясни принцип работы шарико-винтовой передачи, но так, будто я инженер-механик с 10-летним стажем» в DeepSeek и YandexGPT. Выведи в консоль:
- Ответ каждой модели (обрезанный до 500 символов).
- Расход токенов для каждой.
- Сделай вывод, какая модель отвечает лаконичнее (меньше токенов завершения при сравнимом качестве).

---

### Вопросы для самопроверки (1.1)
1. Чем AI-инженер отличается от ML-инженера? Почему TypeScript-бэкендер может легко войти в эту роль?
2. Что такое контекстное окно и что произойдёт, если сумма токенов промпта и истории превысит его размер?
3. Как температура влияет на генерацию кода? Какое значение ты выберешь для извлечения структурированных данных из ответа LLM?
4. Почему в ответе DeepSeek поле `finish_reason` может быть `"length"` и как это исправить?
5. Назови основные отличия формата сообщений YandexGPT от DeepSeek (OpenAI-совместимого).

---

## Модуль 1.2. Prompt Engineering для инженеров

### Теория

Промпт-инжиниринг — это не магия и не искусство, а инженерная дисциплина: мы проектируем входные данные так, чтобы получить от LLM предсказуемый, структурированный и полезный результат. Ты управляешь моделью через текст, и этот текст нужно собирать как программный компонент.

#### Структура промпта
Любой запрос к LLM состоит из контекстной настройки и задачи. Мы используем три основные роли:

- **System** — «прошивка» модели. Здесь мы фиксируем роль, стиль общения, границы дозволенного и формат вывода. Например: «Ты — инженер-конструктор в КБ. Все ответы даёшь в формате JSON: {“answer”: “...”}. Не используешь Markdown».
- **User** — конкретная инструкция, вопрос, входные данные. Может содержать динамические переменные, которые мы подставляем из кода.
- **Assistant** — история предыдущих ответов. Передавая её, мы поддерживаем диалог. Когда ты добавляешь результат tool call, он тоже попадает в историю.

#### Зачем нужен system-промпт
Правильный system-промпт радикально меняет поведение. Без него модель может растекаться мыслью, добавлять лишнюю информацию, игнорировать желаемый формат. Он работает как «подсказка уровня операционной системы» и имеет высокий приоритет. Инженерный подход: относись к system-промпту как к конфигурации модуля.

Примеры:
- *Без system*: «Рассчитай нагрузку на балку» → ответ с предположениями, общими словами.
- *System: «Ты прочностной аналитик. Используй только данные, указанные пользователем. Вычисления оформляй как формулу → расчёт → результат.»* → конкретика.

#### Основные техники
1. **Zero-shot** — даём только описание задачи без примеров. Подходит для простых команд и когда формат очевиден.
   > User: Переведи “bearing” на русский техническим языком.
2. **Few-shot** — добавляем 2–3 примера «запрос → идеальный ответ». Модель схватывает паттерн и воспроизводит его для нового запроса. Мощный инструмент для структурированных задач.
3. **Chain-of-Thought (CoT)** — просим модель думать по шагам перед финальным ответом. Фраза «Давай рассуждать пошагово» или «Explain your reasoning step by step» заставляет модель генерировать промежуточные рассуждения, что резко повышает точность в логических и математических задачах.

#### Шаблонизация промптов в TypeScript
Промпт должен быть типобезопасным. Мы создадим функцию, которая принимает строго типизированные параметры и возвращает готовый массив сообщений. Используем template literals, обязательно экранируем специальные символы, если модель может их неверно интерпретировать (например, JSON внутри промпта).

#### Контекстное окно и управление историей
История диалога — это массив сообщений. При каждом запросе ты можешь отправлять всю историю или её часть. Следи за суммарным количеством токенов. Для расчёта можно использовать библиотеку `tiktoken` (для OpenAI-подобных моделей) или приблизительную оценку. Пока мы будем укладываться в лимит, но при долгих диалогах придётся обрезать старые сообщения.

### Практика

#### 1. Создаём модуль шаблонов `src/prompt-templates.ts`
Реализуем функцию для инженера-конструктора, принимающую название детали и возвращающую массив сообщений с system и user.

```typescript
export interface PromptParams {
  partName: string;
  material?: string;
}

export function generateConstructionPrompt(params: PromptParams) {
  const { partName, material = 'сталь 40Х' } = params;
  
  const systemPrompt = `Ты — инженер-конструктор машиностроительного КБ.
Твоя задача — спроектировать деталь, указанную пользователем.
Ответь в формате JSON:
{
  "название": "...",
  "материал": "...",
  "геометрические_параметры": {
    "диаметр_мм": <число>,
    "длина_мм": <число>,
    "особенности": "перечисление"
  },
  "допуски": {
    "квалитет": "...",
    "посадка": "..."
  },
  "технические_требования": "строка"
}
Никаких пояснений вне JSON.`;

  const userPrompt = `Спроектируй деталь: ${partName}.
Материал по умолчанию: ${material}.
Если деталь нестандартная, предложи ближайший аналог.`;

  return {
    systemPrompt,
    userPrompt,
    messages: [
      { role: 'system', content: systemPrompt },
      { role: 'user', content: userPrompt },
    ],
  };
}
```

Протестируем, создав `src/test-template.ts`:
```typescript
import { generateConstructionPrompt } from './prompt-templates';
import OpenAI from 'openai';
import '../load-env';

const deepseek = new OpenAI({
  baseURL: 'https://api.deepseek.com',
  apiKey: process.env.DEEPSEEK_API_KEY!,
});

async function main() {
  const { messages } = generateConstructionPrompt({ partName: 'вал-шестерня' });
  
  const resp = await deepseek.chat.completions.create({
    model: 'deepseek-chat',
    messages: messages as any,
    temperature: 0.1,
    response_format: { type: 'json_object' }, // DeepSeek поддерживает JSON mode
  });
  
  const json = JSON.parse(resp.choices[0].message.content!);
  console.log('Спроектированная деталь:');
  console.log(JSON.stringify(json, null, 2));
}

main().catch(console.error);
```

**Типичные ошибки и решения:**
- *Модель возвращает Markdown-обёртку вокруг JSON* — добавь в system «Не используй форматирование Markdown, только чистый JSON».
- *Некорректный JSON* — уменьши температуру до 0, используй `response_format: { type: 'json_object' }` (DeepSeek и OpenAI поддерживают). Для YandexGPT такой опции нет, придётся парсить с обработкой ошибок.

#### 2. Экспериментируем с системным промптом
Создадим скрипт `src/system-experiment.ts`, который отправляет один и тот же user-запрос «Расскажи про сварку трением» трем разным system-настройкам:
1. «Ты — научный сотрудник» (подробно, сложно).
2. «Ты — мастер цеха» (простыми словами, практично).
3. «Ты — менеджер по продажам оборудования» (акцент на выгоды, стоимость).

Сравни длину и стиль ответов.

#### 3. Применяем Few-shot
Задача: модель должна конвертировать инженерные обозначения в читаемый вид. Создадим `src/few-shot.ts`:

```typescript
const fewShotSystem = `Ты — транслятор обозначений. Преобразуй входную строку в человекочитаемое описание.
Примеры:
Вход: "М16×1.5-6H"
Выход: "Метрическая резьба, номинальный диаметр 16 мм, шаг 1.5 мм, поле допуска гайки 6H"

Вход: "Подшипник 6204-2RS"
Выход: "Шариковый радиальный подшипник, внутренний диаметр 20 мм, наружный 47 мм, ширина 14 мм, с двухсторонним уплотнением"

Теперь обработай новый запрос.`;

const userPrompt = "М16×1.5-6H"; // уже встречался в примере? Нет, другой запрос: "Вал-шестерня z24 m3"
```

Модель, увидев два примера, научится извлекать сущности и форматировать ответ даже без точного знания ГОСТа. Это и есть few-shot.

#### 4. Chain-of-Thought: думаем шагами
Задача: подобрать квалитет и допуск для вала с номиналом 25 мм и требуемой посадкой H7/k6. Просим модель размышлять последовательно.

Создадим `src/cot.ts` с промптом:
```
Ты инженер-метролог. Действуй по шагам:
1. Определи номинальный размер.
2. По обозначению посадки найди отклонения для отверстия H7 и вала k6.
3. Рассчитай предельные размеры и допуски.
4. Выведи результат в виде таблицы.

Запрос: посадка 25 H7/k6.
```

Модель DeepSeek-R1 (reasoner) умеет такое из коробки, но мы можем заставить и обычный `deepseek-chat` делать CoT, добавив «Давай рассуждать по шагам» в system.

---

### Домашнее задание (Модуль 1.2)
Создай три промпта для трёх инженерных сценариев:
1. **Конструктор** — генерация спецификации на сборочный чертёж (список деталей в JSON).
2. **Технолог** — подбор режимов резания для токарной операции (использовать few-shot с примерами).
3. **Оператор станка с ЧПУ** — перевести текстовое описание «фрезеровать контур прямоугольника 100x50» в G-код (цепочка размышлений).

Протестируй их на DeepSeek и YandexGPT. Оформи отчёт в Markdown со скриншотами консоли, анализом расхода токенов и наблюдениями: где какая модель справилась лучше. Приложи код.

---

### Вопросы для самопроверки (1.2)
1. Почему system-промпт часто важнее самого user-запроса? Приведи пример.
2. Как с помощью few-shot примеров заставить модель выдавать ответ в строго определённом JSON-формате без использования `response_format`?
3. В чём преимущество Chain-of-Thought перед прямым запросом для сложных расчётов? Когда CoT может быть вреден?
4. Ты отправил модель длинный диалог, и ответ стал обрезаться. Что нужно проверить в первую очередь и как исправить?
5. Как типобезопасная шаблонизация промптов помогает избежать ошибок в production?

---

## Модуль 1.3. Function Calling (Tool Use) на практике

### Теория

Function Calling (он же Tool Use) — это способность LLM не просто генерировать текст, а запрашивать вызов внешней функции с конкретными аргументами. Модель сама решает, *какую* функцию вызвать и с какими параметрами, основываясь на промпте пользователя и описании доступных инструментов.

Это ключевой механизм для создания AI-агентов, которые могут взаимодействовать с реальным миром: читать файлы, управлять оборудованием, обращаться к базам данных.

#### Как это работает (цикл)
1. **Ты отправляешь модели** массив сообщений + массив определений инструментов (`tools`). Определение инструмента содержит имя, описание и JSON Schema параметров.
2. **Модель возвращает** либо обычный текстовый ответ (если инструменты не нужны), либо объект `tool_calls` в своём сообщении (роль `assistant`). Внутри — имя функции и аргументы (строка JSON, которую надо распарсить).
3. **Ты в своём коде** выполняешь реальную функцию с этими аргументами и получаешь результат.
4. **Ты добавляешь в историю** сообщение с ролью `tool`, где указываешь `tool_call_id` и результат в виде строки.
5. **Ты снова отправляешь запрос модели** с обновлённой историей. Модель видит результат и генерирует финальный текстовый ответ, возможно, запрашивая ещё вызовы.

#### Описание инструментов (JSON Schema)
Формат, совместимый с OpenAI и DeepSeek:
```json
{
  "type": "function",
  "function": {
    "name": "get_weather",
    "description": "Получает текущую погоду в городе",
    "parameters": {
      "type": "object",
      "properties": {
        "city": {
          "type": "string",
          "description": "Название города"
        }
      },
      "required": ["city"]
    }
  }
}
```
Чем точнее и понятнее `description`, тем лучше модель выбирает функцию и генерирует аргументы. Избегай двусмысленности.

#### Особенности YandexGPT
YandexGPT поддерживает Function Calling в формате, близком к OpenAI, но есть нюансы: вместо `tools` используется поле `functionCall` в запросе (устаревший подход OpenAI). На момент написания актуальный SDK для Node.js мог использовать `FunctionCall` внутри `completionOptions`. Мы покажем вариант с прямым REST-запросом, который более гибкий.

### Практика

#### 1. Создаём функции-заглушки
Нам понадобятся три инструмента. Создадим файл `src/tools.ts`:

```typescript
// read_cad_file - эмуляция чтения CAD-файла
export async function readCadFile(filepath: string): Promise<string> {
  // В реальном мире здесь был бы парсер STEP/IGES
  const mockData: Record<string, string> = {
    'bracket.step': 'Деталь "Кронштейн": материал АМг6, габариты 120x80x15 мм, масса 0.34 кг.',
    'shaft.igs': 'Вал: сталь 45, диаметр 40 мм, длина 250 мм.',
  };
  return mockData[filepath] ?? 'Файл не найден или формат не поддерживается.';
}

// get_machine_status - опрос станка по имени
export async function getMachineStatus(machine: string): Promise<string> {
  const statuses: Record<string, string> = {
    'laser_1': 'RUNNING',
    'laser_2': 'IDLE',
    'lathe_1': 'ERROR: перегрев шпинделя',
  };
  return statuses[machine] ?? 'UNKNOWN: станок не найден в сети';
}

// convert_units - конвертер единиц измерения
export async function convertUnits(value: number, from: string, to: string): Promise<string> {
  // Примитивная реализация для демонстрации
  const conversions: Record<string, number> = {
    'mm_to_cm': 0.1,
    'cm_to_mm': 10,
    'mm_to_inch': 0.0393701,
    'inch_to_mm': 25.4,
  };
  const key = `${from}_to_${to}`;
  const factor = conversions[key];
  if (factor === undefined) return `Конвертация ${from} -> ${to} не поддерживается.`;
  return String(value * factor);
}
```

#### 2. Описываем инструменты для DeepSeek
Создадим `src/deepseek-tool-calling.ts`. Определим массив `tools` в формате OpenAI:

```typescript
const tools = [
  {
    type: 'function' as const,
    function: {
      name: 'read_cad_file',
      description: 'Читает содержимое CAD-файла (STEP, IGES) и возвращает информацию о детали: материал, размеры, массу.',
      parameters: {
        type: 'object',
        properties: {
          filepath: {
            type: 'string',
            description: 'Имя файла, например "bracket.step"',
          },
        },
        required: ['filepath'],
      },
    },
  },
  {
    type: 'function' as const,
    function: {
      name: 'get_machine_status',
      description: 'Возвращает текущий статус станка: IDLE, RUNNING, ERROR или UNKNOWN.',
      parameters: {
        type: 'object',
        properties: {
          machine: {
            type: 'string',
            description: 'Идентификатор станка, например "laser_1"',
          },
        },
        required: ['machine'],
      },
    },
  },
];
```

#### 3. Реализуем полный цикл вызова инструментов
Основной алгоритм:

```typescript
import OpenAI from 'openai';
import { readCadFile, getMachineStatus } from './tools';
import '../load-env';

const deepseek = new OpenAI({
  baseURL: 'https://api.deepseek.com',
  apiKey: process.env.DEEPSEEK_API_KEY!,
});

async function main() {
  const messages: OpenAI.Chat.ChatCompletionMessageParam[] = [
    {
      role: 'system',
      content: 'Ты — помощник начальника цеха. Используй доступные инструменты для ответа на вопросы.',
    },
    {
      role: 'user',
      content: 'Дай сводку по файлу bracket.step и проверь, готов ли лазерный станок laser_1.',
    },
  ];

  // Первый запрос
  let response = await deepseek.chat.completions.create({
    model: 'deepseek-chat',
    messages,
    tools,
    temperature: 0.1,
  });

  const responseMessage = response.choices[0].message;
  
  // Если есть tool_calls — обрабатываем их
  const toolCalls = responseMessage.tool_calls;
  if (toolCalls) {
    console.log(`Модель запросила ${toolCalls.length} вызовов инструментов.`);
    
    // Добавляем ответ ассистента с tool_calls в историю
    messages.push(responseMessage);

    // Выполняем каждый запрошенный инструмент
    for (const toolCall of toolCalls) {
      const functionName = toolCall.function.name;
      const functionArgs = JSON.parse(toolCall.function.arguments);
      
      console.log(`Вызываю ${functionName} с аргументами:`, functionArgs);
      
      let functionResult: string;
      try {
        switch (functionName) {
          case 'read_cad_file':
            functionResult = await readCadFile(functionArgs.filepath);
            break;
          case 'get_machine_status':
            functionResult = await getMachineStatus(functionArgs.machine);
            break;
          default:
            functionResult = `Ошибка: неизвестная функция ${functionName}`;
        }
      } catch (err: any) {
        functionResult = `Ошибка выполнения: ${err.message}`;
      }
      
      console.log(`Результат: ${functionResult}`);
      
      // Добавляем результат в историю как сообщение роли tool
      messages.push({
        role: 'tool',
        tool_call_id: toolCall.id,
        content: functionResult,
      });
    }

    // Второй запрос с результатами
    response = await deepseek.chat.completions.create({
      model: 'deepseek-chat',
      messages,
      tools, // инструменты можно передавать снова или не передавать, если они больше не нужны
      temperature: 0.1,
    });

    console.log('\nФинальный ответ модели:');
    console.log(response.choices[0].message.content);
  } else {
    // Нет tool_calls, просто выводим текст
    console.log('Ответ модели без вызова инструментов:');
    console.log(responseMessage.content);
  }
}

main().catch(console.error);
```

**Типичные ошибки и их решение:**
- *Модель не вызывает инструмент, хотя он нужен* — проверь описание инструмента: возможно, оно недостаточно ясное. Добавь в system-промпт «Если вопрос касается содержимого файла, используй read_cad_file».
- *Аргументы приходят с неверными типами* — например, число приходит как строка. Это редкость для DeepSeek, но если случается, добавь валидацию при парсинге аргументов и возвращай ошибку в результате.
- *Модель вызывает несуществующую функцию* — ты должен перехватить `default` в switch и вернуть сообщение об ошибке. Модель увидит ошибку и попробует исправиться.
- *Бесконечный цикл вызовов* — модель может продолжать вызывать инструменты. Ограничь количество итераций (например, не более 5) в коде.

#### 4. YandexGPT и Function Calling
На момент написания YandexGPT поддерживает вызов функций, но интерфейс может отличаться. Покажем пример через REST API с использованием `axios` (установи `npm install axios`). Создадим `src/yandexgpt-tool-calling.ts`:

```typescript
import axios from 'axios';
import { readCadFile, getMachineStatus } from './tools';
import '../load-env';

const FOLDER_ID = process.env.YANDEX_FOLDER_ID!;
const API_KEY = process.env.YANDEX_API_KEY!; // API-ключ сервисного аккаунта
const BASE_URL = 'https://llm.api.cloud.yandex.net/foundationModels/v1/completion';

interface YandexTool {
  name: string;
  description: string;
  parameters: any;
}

async function main() {
  const tools: YandexTool[] = [
    {
      name: 'readCadFile',
      description: 'Читает CAD-файл и возвращает информацию о детали',
      parameters: {
        type: 'object',
        properties: {
          filepath: { type: 'string', description: 'путь к файлу' }
        },
        required: ['filepath']
      }
    },
    // getMachineStatus аналогично
  ];

  const messages = [
    { role: 'system', text: 'Ты — помощник начальника цеха.' },
    { role: 'user', text: 'Дай сводку по файлу bracket.step и статус лазера laser_1' }
  ];

  let response = await axios.post(BASE_URL, {
    modelUri: `gpt://${FOLDER_ID}/yandexgpt/latest`,
    completionOptions: {
      stream: false,
      temperature: 0.1,
      maxTokens: 500,
    },
    messages,
    tools, // YandexGPT ожидает массив tools
  }, {
    headers: {
      'Authorization': `Api-Key ${API_KEY}`,
      'Content-Type': 'application/json',
    }
  });

  let reply = response.data.result.alternatives[0].message;
  while (reply.toolCalls && reply.toolCalls.length > 0) {
    messages.push(reply); // добавляем сообщение ассистента с toolCalls
    for (const call of reply.toolCalls) {
      const funcName = call.name;
      const args = call.args; // уже объект
      let result: string;
      if (funcName === 'readCadFile') result = await readCadFile(args.filepath);
      else if (funcName === 'getMachineStatus') result = await getMachineStatus(args.machine);
      else result = `Неизвестная функция ${funcName}`;
      
      messages.push({
        role: 'tool',
        toolCallId: call.id,
        text: result,
      });
    }
    // Отправляем снова
    response = await axios.post(BASE_URL, {
      modelUri: `gpt://${FOLDER_ID}/yandexgpt/latest`,
      completionOptions: { stream: false, temperature: 0.1 },
      messages,
      tools,
    }, {
      headers: { 'Authorization': `Api-Key ${API_KEY}`, 'Content-Type': 'application/json' },
    });
    reply = response.data.result.alternatives[0].message;
  }
  
  console.log('Финальный ответ:', reply.text);
}

main().catch(console.error);
```

Обрати внимание: структура `toolCalls` в YandexGPT отличается: поле `name` и `args` (уже JSON-объект, а не строка). Формат может меняться, сверяйся с документацией [YandexGPT API](https://yandex.cloud/ru/docs/foundation-models/concepts/tool-call).

**Типичные ошибки YandexGPT:**
- *400 Bad Request* — проверь, что в `messages` role `tool` имеет поле `toolCallId` (именно так, CamelCase). В их документации возможны варианты.
- *Модель игнорирует инструменты* — YandexGPT может быть менее охотной до вызовов, чем DeepSeek. Попробуй усилить system-промпт: «Ты ОБЯЗАН использовать инструменты, если информация недоступна в тексте запроса».

#### 5. Добавляем третий инструмент convert_units
Расширим наш список инструментов. В `tools.ts` добавим описание и реализацию (уже есть). Покажем сценарий с несколькими вызовами в одном ответе. Модель может запросить конвертацию дважды за один запрос.

Пример user-сообщения: «Переведи 25.4 мм в дюймы и 10 см в мм». DeepSeek должен вернуть два `tool_calls` в одном ответе. Цикл выше это обработает: все вызовы выполняются последовательно (или параллельно при желании), результаты добавляются в историю, и модель даёт финальный ответ.

---

### Задание с самопроверкой (Модуль 1.3)
Реализуй скрипт, который использует три инструмента (`readCadFile`, `getMachineStatus`, `convertUnits`) для обработки сложного запроса:
> «Файл bracket.step: переведи массу из кг в граммы. И проверь лазерный станок laser_1. Если он в ERROR, скажи, что делать.»

Демонстрируй полный цикл, включая обработку ошибок (например, неверный ID станка, несуществующий файл). Выведи подробный лог, чтобы было видно все шаги.

---

### Вопросы для самопроверки (1.3)
1. Зачем в определении инструмента нужно подробное `description`? Может ли модель вызвать функцию, не глядя на него?
2. Что произойдёт, если модель вызовет инструмент, а мы вернём ошибку? Нужно ли прерывать диалог?
3. Почему результаты вызова инструментов мы добавляем в историю с ролью `tool`, а не `user` или `assistant`?
4. В чём ключевое различие формата `tool_calls` у DeepSeek (OpenAI-совместимый) и YandexGPT?
5. Как ограничить количество последовательных вызовов инструментов в коде, чтобы избежать бесконечного цикла?

---

## Домашнее задание по Блоку 1

Создай мини-ассистента «Цеховой мастер» (`src/shop-assistant.ts`), который использует не менее трёх инструментов (можно придумать свои, например: `get_inventory`, `create_work_order`, `send_alert`). Требования:
1. Поддерживает диалог с пользователем, запоминая историю.
2. Инструменты описаны строго типизированными объектами.
3. Реализован безопасный цикл вызова инструментов с ограничением в 5 итераций.
4. Обрабатываются ошибки инструментов: модель должна видеть сообщение об ошибке и предлагать альтернативу.
5. Ассистент должен работать как с DeepSeek, так и с YandexGPT (выбор через переменную окружения `LLM_PROVIDER`).

Подготовь отчёт в формате Markdown:
- Краткое описание архитектуры.
- Скриншоты консоли с примерами диалогов для обоих провайдеров.
- Анализ различий в поведении моделей при вызове инструментов.
- Выводы: какую модель рекомендовал бы для production с интенсивным Tool Calling и почему.

---

**Готово!** Ты заложил фундамент. Теперь ты умеешь подключать LLM, создавать умные промпты и превращать модель в активного участника системы, способного выполнять действия. В следующем блоке мы унифицируем работу с инструментами через MCP-серверы и начнём строить настоящий оркестратор.