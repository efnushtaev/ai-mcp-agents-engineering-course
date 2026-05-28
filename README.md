# 🤖 AI MCP Agents Engineering Course

**Практический курс по созданию AI-агентов с использованием MCP-серверов, оркестрации и промышленного деплоя.**

> От первого запроса к LLM до production-ready системы с Docker, мониторингом и CI.

---

## 📋 О курсе

Этот курс проведёт вас через все этапы создания AI-агентов — от базового знакомства с LLM до развёртывания полноценной системы в продакшн. Вы научитесь проектировать MCP-серверы, интегрировать их с языковыми моделями (DeepSeek, YandexGPT), писать эмуляторы реального оборудования и оркестрировать работу агентов.

### Чему вы научитесь

- **Работа с LLM**: DeepSeek и YandexGPT, Function Calling, промпт-инжиниринг
- **MCP (Model Context Protocol)**: создание кастомных серверов, инструментов и ресурсов
- **Архитектура агентов**: универсальный `IToolCallingClient`, фабрики, DI
- **Эмуляция оборудования**: Kompas-3D (CAD), CarverAll 15Pro (лазерный гравёр)
- **Оркестрация**: OpenCode, системные промпты, параллельные вызовы, обработка ошибок
- **Тестирование**: Vitest, MSW, MCP Inspector, E2E-тесты, CI-скрипты
- **DevOps**: Docker multi-stage сборка, docker-compose, pino-логирование, health-check

---

## 🏗 Архитектура

```
┌─────────────────────────────────────────────────────────────┐
│                     OpenCode Orchestrator                   │
│  (opencode.json, system prompt, parallel calls, fallbacks)  │
└──────┬──────────────┬──────────────┬────────────────┬───────┘
       │              │              │                │
       ▼              ▼              ▼                ▼
┌────────────┐ ┌────────────┐ ┌────────────┐ ┌────────────────┐
│ kompas3d-  │ │ carverall- │ │  toolbox-  │ │   file-manager │
│ mcp-server │ │ mcp-server │ │ mcp-server │ │   mcp-server   │
│  (CAD)     │ │ (лазер)    │ │ (echo,add, │ │   (файловые    │
│            │ │            │ │  multiply) │ │   операции)    │
└─────┬──────┘ └─────┬──────┘ └────────────┘ └────────────────┘
      │              │
      ▼              ▼
┌────────────┐ ┌────────────┐
│ Kompas-3D  │ │ CarverAll  │
│ Emulator   │ │ 15Pro      │
│ (Fastify)  │ │ Emulator   │
│            │ │ (Fastify)  │
└────────────┘ └────────────┘

┌─────────────────────────────────────────────────────────────┐
│                  IToolCallingClient (единый интерфейс)      │
├───────────────────────┬─────────────────────────────────────┤
│   DeepSeekClient      │       YandexGPTClient               │
│   (deepseek-chat)     │       (YandexGPT API)               │
└───────────────────────┴─────────────────────────────────────┘
```

---

## 🗺 Структура курса

| Уровень | Тема | Ключевые технологии |
|---------|------|---------------------|
| **[Блок 0](course/level-0/README.md)** | Введение и настройка инструментов | Node.js, OpenCode, DeepSeek API, YandexGPT API, `.env` |
| **[Блок 1](course/level-1/README.md)** | Фундамент: LLM, промпты и вызов инструментов | Prompt engineering, Function Calling, JSON Schema |
| **[Блок 2](course/level-2/README.md)** | Единый интерфейс для Tool Calling | `IToolCallingClient`, DeepSeekClient, YandexGPTClient, Factory, MSW |
| **[Блок 3](course/level-3/README.md)** | Введение в MCP и первый кастомный сервер | MCP SDK, stdio/SSE, JSON-RPC 2.0, toolbox-mcp, ресурсы |
| **[Блок 4](course/level-4/README.md)** | MCP-сервер для Компас-3D | Fastify, эмулятор CAD, kompas3d-mcp, агент-конструктор |
| **[Блок 5](course/level-5/README.md)** | MCP-сервер для лазерного гравировщика | CarverAll 15Pro, async jobs, carverall-mcp, агент-оператор |
| **[Блок 6](course/level-6/README.md)** | Оркестратор на OpenCode | `opencode.json`, system prompt, parallel calls, error handling |
| **[Блок 7](course/level-7/README.md)** | Тестирование и отладка | Vitest, MSW, MCP Inspector, E2E, CI-скрипт |
| **[Блок 8](course/level-8/README.md)** | Продакшн-сборка и деплой | Docker, docker-compose, pino, health-check, deploy.sh |

---

## 🚀 Быстрый старт

### Предварительные требования

- **Node.js** 20+ (рекомендуется 22 LTS)
- **Git**
- **Docker** (для Блока 8)
- **API-ключи**: DeepSeek и/или Yandex Cloud

### Установка

```bash
# Клонировать репозиторий
git clone https://github.com/your-username/ai-mcp-agents-engineering-course.git
cd ai-mcp-agents-engineering-course

# Установить зависимости (для каждого блока — свои)
# Пример для Блока 1:
cd course/level-1 && npm install
```

### Настройка окружения

Создайте файл `.env` в корне проекта:

```env
DEEPSEEK_API_KEY=sk-xxxxxxxxxxxxxxxx
YANDEX_GPT_API_KEY=xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx
YANDEX_GPT_FOLDER_ID=b1gxxxxxxxxxxxxxxxx
```

### Запуск OpenCode

```bash
npx @openagent/open-code
```

Подробная инструкция — в [Блоке 0](course/level-0/README.md).

---

## 🧩 Ключевые компоненты

### MCP-серверы

| Сервер | Назначение | Транспорт |
|--------|-----------|-----------|
| [`toolbox-mcp`](course/level-3/README.md) | Демо: echo, add, multiply + ресурс `clock://current` | stdio |
| [`kompas3d-mcp`](course/level-4/README.md) | Управление Kompas-3D: создание деталей, эскизов, выдавливание, экспорт | stdio / HTTP |
| [`carverall-mcp`](course/level-5/README.md) | Управление лазерным гравёром: гравировка, резка, мониторинг заданий | stdio / HTTP |
| [`file-manager`](course/level-3/README.md) | Файловые операции (домашнее задание) | stdio |

### Эмуляторы

- **Kompas-3D Emulator** — HTTP-сервер на Fastify, эмулирующий REST API CAD-системы
- **CarverAll 15Pro Emulator** — HTTP-сервер на Fastify с асинхронными заданиями (polling)

### Клиенты LLM

- **`IToolCallingClient`** — универсальный интерфейс для вызова инструментов
- **`DeepSeekClient`** — реализация для DeepSeek API (deepseek-chat)
- **`YandexGPTClient`** — реализация для YandexGPT API (с поддержкой Function Calling)
- **Factory** — создание клиента по строке конфигурации

---

## 🛠 Технологический стек

| Категория | Технологии |
|-----------|-----------|
| **Язык** | TypeScript, Node.js 20+ |
| **LLM** | DeepSeek API, YandexGPT API |
| **Протокол** | MCP (Model Context Protocol), JSON-RPC 2.0 |
| **Оркестратор** | OpenCode |
| **HTTP** | Fastify, Axios |
| **Тестирование** | Vitest, MSW (Mock Service Worker), MCP Inspector |
| **Контейнеризация** | Docker (multi-stage), docker-compose |
| **Логирование** | pino, pino-pretty |
| **Валидация** | Zod, JSON Schema |

---

## 📁 Структура проекта

```
ai-mcp-agents-engineering-course/
├── README.md
├── course/
│   ├── level-0/README.md          # Введение, OpenCode, API-ключи
│   ├── level-1/README.md          # LLM, промпты, Function Calling
│   ├── level-2/README.md          # IToolCallingClient, DeepSeek, YandexGPT
│   ├── level-3/README.md          # MCP, toolbox-mcp-server
│   ├── level-4/README.md          # Kompas-3D эмулятор + MCP-сервер
│   ├── level-5/README.md          # CarverAll эмулятор + MCP-сервер
│   ├── level-6/README.md          # OpenCode оркестратор
│   ├── level-7/README.md          # Тестирование и CI
│   └── level-8/README.md          # Docker, деплой, мониторинг
```

Каждый блок содержит:
- Теоретический материал с диаграммами
- Пошаговые практические руководства
- Код для самостоятельной реализации
- Вопросы для самопроверки
- Домашние задания

---

## 🎯 Для кого этот курс

- **Backend-разработчики**, переходящие в AI/ML
- **Full-stack инженеры**, интересующиеся AI-агентами
- **DevOps-инженеры**, желающие понять деплой AI-систем
- **Все**, кто хочет научиться создавать production-ready AI-агентов

### Необходимые знания

- TypeScript / JavaScript (продвинутый уровень)
- Опыт работы с REST API и HTTP
- Базовое понимание Docker и Linux
- Знакомство с Git

---

## 📄 Лицензия

MIT License. Смотрите файл `LICENSE` для подробностей.

---

## 🤝 Вклад в проект

Предложения и улучшения приветствуются! Создавайте Issue или Pull Request.

---

## 📚 Дополнительные материалы

- [MCP Specification](https://spec.modelcontextprotocol.io)
- [OpenCode Documentation](https://github.com/opencode-ai/opencode)
- [DeepSeek API Docs](https://platform.deepseek.com/api-docs)
- [YandexGPT API Docs](https://yandex.cloud/ru/docs/yandexgpt)
- [Fastify Documentation](https://fastify.dev)
- [Docker Multi-stage Builds](https://docs.docker.com/build/building/multi-stage/)