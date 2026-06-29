# Airline API

Микросервис с REST API для управления профилями клиентов авиакомпании и состоянием бронирований.

## 📖 Содержание

- [Описание](#-описание)  
- [Архитектура](#️-архитектура)  
- [Быстрый старт](#-быстрый-старт)  
- [Развертывание в Yandex Cloud](#-развертывание-в-yandex-cloud-подробная-инструкция)  
  - [Шаг 1: Сборка Docker-образа](#шаг-1-сборка-docker-образа)  
  - [Шаг 2: Container Registry](#шаг-2-создание-container-registry-и-загрузка-образа)  
  - [Шаг 3: YDB Document API (опционально)](#шаг-3-создание-базы-данных-ydb-document-api-опционально)  
  - [Шаг 4: Сервисный аккаунт](#шаг-4-создание-сервисного-аккаунта-с-ролями)  
  - [Шаг 5: API-ключ](#шаг-5-создание-api-ключа)  
  - [Шаг 6: Lockbox секрет](#шаг-6-создание-секрета-lockbox)  
  - [Шаг 7: MCP Gateway](#шаг-7-создание-mcp-gateway-опционально)  
  - [Шаг 8: Деплой контейнера](#шаг-8-создание-serverless-container-и-деплой)  
- [API Endpoints](#-api-endpoints)  
- [Переменные окружения](#-переменные-окружения)  
- [Интеграция с MCP](#-интеграция-с-mcp)  
- [Полезные ссылки](#-полезные-ссылки)

## 📋 Описание

Airline API — это микросервис для управления состоянием клиентов авиакомпании:

- **REST API** для CRUD-операций с профилями клиентов  
- **YDB Document API** (опционально) для постоянного хранения данных  
- **In-Memory-хранилище** для быстрой разработки и тестирования  
- **интеграция с MCP Gateway** — для использования в качестве инструментов AI-агентов  
- **Независимый микросервис**

**Основные возможности:**

- управление профилями клиентов (имя, email, телефон, статус лояльности)  
- управление сегментами рейсов (origin, destination, seat, status)  
- изменение мест в самолёте  
- отмена бронирований  
- управление багажом  
- установка предпочтений по питанию  
- запросы на специальную помощь  
- временная шкала действий (timeline) по каждому профилю

## 🏗️ Архитектура

┌─────────────────────────────────────────────────────────────┐

│                  Frontend / ChatKit Agent                   │

│                       или A2A Agent                         │

└──────────────────────┬──────────────────────────────────────┘

                       │ HTTP REST API

                       ▼

┌─────────────────────────────────────────────────────────────┐

│         Serverless Container (Airline API)                  │

│  ┌──────────────────────────────────────────────────────┐   │

│  │  FastAPI Server (main.py)                            │   │

│  │  \- GET  /profile/{profile\_id}                        │   │

│  │  \- POST /seat (изменить место)                       │   │

│  │  \- POST /cancel (отменить бронирование)              │   │

│  │  \- POST /bag (добавить багаж)                        │   │

│  │  \- POST /meal (предпочтения питания)                 │   │

│  │  \- POST /assistance (запрос помощи)                  │   │

│  └────────────┬─────────────────────────────────────────┘   │

│               │                                             │

│  ┌────────────▼─────────────────────────────────────────┐   │

│  │  Airline State Manager (airline\_state.py)            │   │

│  │  \- CustomerProfile (модель данных)                   │   │

│  │  \- FlightSegment (модель сегмента рейса)             │   │

│  │  \- Бизнес-логика операций                            │   │

│  └────────────┬─────────────────────────────────────────┘   │

│               │                                             │

│       ┌───────┴────────┐                                    │

│       ▼                ▼                                    │

│  ┌─────────┐    ┌──────────────┐                            │

│  │ Memory  │    │   DynamoDB   │                            │

│  │ Storage │    │ (YDB Doc API)│                            │

│  └─────────┘    └──────────────┘                            │

└─────────────────────────────────────────────────────────────┘

                       │

                       ▼

              ┌─────────────────┐

              │   MCP Gateway   │

              │  (AI Studio)    │

              │  \- SSE Stream   │

              │  \- Tool calling │

              └─────────────────┘

### Компоненты

1. **FastAPI Server** — REST API с эндпоинтами для управления профилями  
2. **Airline State Manager** — бизнес-логика и модели данных  
3. **Storage Layer** — выбор между in-memory и DynamoDB (YDB Document API)  
4. **MCP Gateway** — интеграция с AI Studio для оборачивания в MCP-сервер

### Модели данных

**CustomerProfile:**

- `customer_id` — уникальный идентификатор  
- `name` — имя клиента  
- `loyalty_status` — статус программы лояльности  
- `loyalty_id` — ID в программе лояльности  
- `email` — электронная почта  
- `phone` — номер телефона  
- `tier_benefits` — список привилегий  
- `segments` — список сегментов рейсов  
- `bags_checked` — количество зарегистрированного багажа  
- `meal_preference` — предпочтения по питанию  
- `special_assistance` — заметки о специальной помощи  
- `timeline` — временная шкала действий

**FlightSegment:**

- `flight_number` — номер рейса  
- `date` — дата рейса  
- `origin` — аэропорт отправления  
- `destination` — аэропорт назначения  
- `departure_time` — время вылета  
- `arrival_time` — время прибытия  
- `seat` — место в самолёте  
- `status` — статус рейса (Scheduled/Cancelled)

## ⚡ Быстрый старт

### Локальная разработка

\# 1\. Установка зависимостей

cd airline-api

uv sync

\# 2\. Запуск с in-memory-хранилищем (для разработки)

export USE\_MEMORY\_STORE=true

export PORT=8001

uv run uvicorn app.main:app \--host 0.0.0.0 \--port 8001 \--reload

\# 3\. Проверка работы

curl http://localhost:8001/health

curl http://localhost:8001/profile/demo\_default\_thread

### С использованием Docker

\# Сборка образа

docker build \-t airline-api:latest .

\# Запуск контейнера

docker run \-p 8001:8001 \\

  \-e PORT=8001 \\

  \-e USE\_MEMORY\_STORE=true \\

  airline-api:latest

## 🚀 Развертывание в Yandex Cloud (подробная инструкция)

Эта инструкция описывает полный процесс развертывания Airline API в Yandex Cloud Serverless Containers с интеграцией через MCP Gateway.

### Предварительные требования

- Установленный [Yandex Cloud CLI](https://cloud.yandex.ru/docs/cli/quickstart)  
- Docker для сборки образа  
- Права на создание ресурсов в Yandex Cloud

### Шаг 1: Сборка Docker-образа

Соберите Docker-образ API:

cd airline-api

\# Сборка образа

docker build \-t airline-api:latest .

### Шаг 2: Создание Container Registry и загрузка образа

Создайте реестр Container Registry и загрузите в него образ:

\# Создание реестра (если ещё не создан)

yc container registry create \--name my-registry

\# Получение ID реестра

REGISTRY\_ID=$(yc container registry get \--name my-registry \--format json | jq \-r '.id')

\# Авторизация в Container Registry

yc container registry configure-docker

\# Тегирование образа

docker tag airline-api:latest cr.yandex/${REGISTRY\_ID}/airline-api:latest

\# Загрузка образа

docker push cr.yandex/${REGISTRY\_ID}/airline-api:latest

### Шаг 3: Создание базы данных YDB Document API (опционально)

💡  Для разработки можно использовать in-memory-хранилище (`USE_MEMORY_STORE=true`). Для продакшена рекомендуется YDB.

Создайте базу данных Serverless YDB с поддержкой Document API:

\# Создание Serverless YDB

yc ydb database create \\

  \--name airline-db \\

  \--serverless

\# Получение эндпоинта Document API

DOCUMENT\_API\_ENDPOINT=$(yc ydb database get airline-db \--format json | jq \-r '.document\_api\_endpoint')

echo "Document API Endpoint: ${DOCUMENT\_API\_ENDPOINT}"

💡 Сохраните значение `DOCUMENT_API_ENDPOINT`. Оно понадобится для настройки переменных окружения.

### Шаг 4: Создание сервисного аккаунта с ролями

Создайте сервисный аккаунт и назначьте ему необходимые роли:

\# Создание сервисного аккаунта

yc iam service-account create \--name airline-api-sa \--description "Service account for Airline API"

\# Получение ID сервисного аккаунта

SA\_ID=$(yc iam service-account get airline-api-sa \--format json | jq \-r '.id')

\# Получение ID каталога

FOLDER\_ID=$(yc config get folder-id)

\# Назначение ролей (одной командой)

yc resourcemanager folder add-access-bindings ${FOLDER\_ID} \\

  \--access-binding role=lockbox.payloadViewer,subject=serviceAccount:${SA\_ID} \\

  \--access-binding role=container-registry.images.puller,subject=serviceAccount:${SA\_ID} \\

  \--access-binding role=serverless.containers.invoker,subject=serviceAccount:${SA\_ID} \\

  \--access-binding role=ydb.editor,subject=serviceAccount:${SA\_ID}

### Шаг 5: Создание API-ключа

Создайте API-ключ для сервисного аккаунта:

\# Создание API-ключа и извлечение секрета

API\_KEY=$(yc iam api-key create \\

  \--service-account-id ${SA\_ID} \\

  \--description "API key for Airline API" \\

  \--format json | jq \-r '.secret')

echo "API Key: ${API\_KEY}"

💡 Сохраните `API_KEY` в безопасном месте.

### Шаг 6: Создание секрета Lockbox

Создайте секрет в Yandex Lockbox для хранения API-ключа:

\# Создание секрета с API-ключом

yc lockbox secret create \\

  \--name airline-api-key \\

  \--description "API key for Airline API" \\

  \--payload "\[{'key': 'API\_KEY', 'text\_value': '${API\_KEY}'}\]"

\# Получение ID секрета

SECRET\_ID=$(yc lockbox secret get airline-api-key \--format json | jq \-r '.id')

VERSION\_ID=$(yc lockbox secret get airline-api-key \--format json | jq \-r '.current\_version.id')

echo "Secret ID: ${SECRET\_ID}"

### Шаг 7: Создание MCP Gateway (опционально)

MCP Gateway позволяет AI-агентам использовать Airline API как набор инструментов:

\# Сначала создайте контейнер (см. шаг 8), а затем MCP Gateway

\# Получение URL контейнера

CONTAINER\_URL=$(yc serverless container get airline-api \--format json | jq \-r '.url')

\# Создание MCP Gateway

yc ai mcp-gateway create \\

  \--name airline-tools \\

  \--service-account-id ${SA\_ID} \\

  \--sse-url ${CONTAINER\_URL}/sse \\

  \--sse-headers "Authorization: Api-Key ${API\_KEY}"

\# Получение ID MCP Gateway

MCP\_GATEWAY\_ID=$(yc ai mcp-gateway get airline-tools \--format json | jq \-r '.id')

\# URL для использования в агентах

MCP\_SERVER\_URL="https://${MCP\_GATEWAY\_ID}.6q7pzfrg.mcpgw.serverless.yandexcloud.net/sse"

echo "MCP Server URL: ${MCP\_SERVER\_URL}"

💡 MCP Gateway нужен, только если вы хотите использовать API как инструменты для AI-агентов — через MCP Hub в AI Studio.

### Шаг 8: Создание и развертывание Serverless Container

Создайте публичный Serverless Container и разверните ревизию:

\# Получение переменных из предыдущих шагов

FOLDER\_ID=$(yc config get folder-id)

REGISTRY\_ID=$(yc container registry get \--name my-registry \--format json | jq \-r '.id')

SA\_ID=$(yc iam service-account get airline-api-sa \--format json | jq \-r '.id')

DOCUMENT\_API\_ENDPOINT=$(yc ydb database get airline-db \--format json | jq \-r '.document\_api\_endpoint')

SECRET\_ID=$(yc lockbox secret get airline-api-key \--format json | jq \-r '.id')

VERSION\_ID=$(yc lockbox secret get airline-api-key \--format json | jq \-r '.current\_version.id')

\# Создание контейнера

yc serverless container create \--name airline-api

\# Вариант 1: Деплой с in-memory-хранилищем (для разработки)

yc serverless container revision deploy \\

  \--container-name airline-api \\

  \--image cr.yandex/${REGISTRY\_ID}/airline-api:latest \\

  \--service-account-id ${SA\_ID} \\

  \--memory 512MB \\

  \--cores 1 \\

  \--execution-timeout 30s \\

  \--concurrency 4 \\

  \--environment PORT=8001 \\

  \--environment USE\_MEMORY\_STORE=true \\

  \--secret environment-variable=API\_KEY,id=${SECRET\_ID},version-id=${VERSION\_ID},key=API\_KEY

\# Вариант 2: Деплой с YDB Document API (для production)

yc serverless container revision deploy \\

  \--container-name airline-api \\

  \--image cr.yandex/${REGISTRY\_ID}/airline-api:latest \\

  \--service-account-id ${SA\_ID} \\

  \--memory 512MB \\

  \--cores 1 \\

  \--execution-timeout 30s \\

  \--concurrency 4 \\

  \--environment PORT=8001 \\

  \--environment USE\_MEMORY\_STORE=false \\

  \--environment AWS\_REGION=ru-central1 \\

  \--environment DYNAMODB\_ENDPOINT\_URL=${DOCUMENT\_API\_ENDPOINT} \\

  \--environment DYNAMODB\_TABLE\_PREFIX=airline \\

  \--environment AUTO\_CREATE\_TABLES=true \\

  \--secret environment-variable=API\_KEY,id=${SECRET\_ID},version-id=${VERSION\_ID},key=API\_KEY

\# Сделать контейнер публичным

yc serverless container allow-unauthenticated-invoke airline-api

\# Получить URL контейнера

CONTAINER\_URL=$(yc serverless container get airline-api \--format json | jq \-r '.url')

echo "Container URL: ${CONTAINER\_URL}"

### Проверка развертывания

Проверьте, что API работает корректно:

\# Health check

curl ${CONTAINER\_URL}/health

\# Получение профиля

curl ${CONTAINER\_URL}/profile/demo\_default\_thread

\# Список всех эндпоинтов

curl ${CONTAINER\_URL}/

\# Изменение места

curl \-X POST ${CONTAINER\_URL}/seat \\

  \-H "Content-Type: application/json" \\

  \-H "Authorization: Api-Key ${API\_KEY}" \\

  \-d '{

    "profile\_id": "demo\_default\_thread",

    "flight\_number": "OA476",

    "seat": "14C"

  }'

## 📡 API эндпоинтов

### GET /health

Сделать Health check эндпоинтов.

curl http://localhost:8001/health

Response:

{

  "status": "healthy",

  "service": "airline-state-management"

}

### GET /profile/{profile\_id}

Получить профиль клиента по profile\_id.

curl http://localhost:8001/profile/demo\_default\_thread

Response:

{

  "success": true,

  "profile": {

    "customer\_id": "demo\_default\_thread",

    "name": "Jordan Miles",

    "loyalty\_status": "Aviator Platinum",

    "loyalty\_id": "APL-204981",

    "email": "jordan.miles@example.com",

    "phone": "+1 (415) 555-9214",

    "tier\_benefits": \[

      "Complimentary upgrades when available",

      "Unlimited lounge access",

      "Priority boarding group 1"

    \],

    "segments": \[...\],

    "bags\_checked": 0,

    "meal\_preference": null,

    "special\_assistance": null,

    "timeline": \[...\]

  }

}

### POST /seat

Изменить место клиента на рейсе.

curl \-X POST http://localhost:8001/seat \\

  \-H "Content-Type: application/json" \\

  \-d '{

    "profile\_id": "demo\_default\_thread",

    "flight\_number": "OA476",

    "seat": "14C"

  }'

Response:

{

  "success": true,

  "message": "Seat updated to 14C on flight OA476."

}

### POST /cancel

Отменить поездку клиента.

curl \-X POST http://localhost:8001/cancel \\

  \-H "Content-Type: application/json" \\

  \-d '{

    "profile\_id": "demo\_default\_thread"

  }'

Response:

{

  "success": true,

  "message": "The reservation has been cancelled. Refund processing will begin immediately."

}

### POST /bag

Добавить багаж для клиента.

curl \-X POST http://localhost:8001/bag \\

  \-H "Content-Type: application/json" \\

  \-d '{

    "profile\_id": "demo\_default\_thread"

  }'

Response:

{

  "success": true,

  "message": "Checked bag added. You now have 1 bag(s) checked."

}

### POST /meal

Установить предпочтение по еде для клиента.

curl \-X POST http://localhost:8001/meal \\

  \-H "Content-Type: application/json" \\

  \-d '{

    "profile\_id": "demo\_default\_thread",

    "meal": "vegetarian"

  }'

Response:

{

  "success": true,

  "message": "We'll note vegetarian as the meal preference."

}

### POST /assistance

Запросить специальную помощь для клиента.

curl \-X POST http://localhost:8001/assistance \\

  \-H "Content-Type: application/json" \\

  \-d '{

    "profile\_id": "demo\_default\_thread",

    "note": "Wheelchair assistance needed"

  }'

Response:

{

  "success": true,

  "message": "Assistance request recorded. Airport staff will be notified."

}

### GET /

Корневой эндпоинт с информацией о сервисе.

curl http://localhost:8001/

Response:

{

  "service": "Airline State Management API",

  "version": "1.0.0",

  "endpoints": {

    "health": "GET /health",

    "get\_profile": "GET /profile/{profile\_id}",

    "change\_seat": "POST /seat",

    "cancel\_trip": "POST /cancel",

    "add\_bag": "POST /bag",

    "set\_meal": "POST /meal",

    "request\_assistance": "POST /assistance"

  }

}

## 📝 Переменные окружения

Полный список переменных окружения для Serverless Container:

| Переменная | Обязательная | Значение по умолчанию | Описание |
| :---- | :---- | :---- | :---- |
| `PORT` | Да | \- | Порт для HTTP сервера (обычно 8001\) |
| `USE_MEMORY_STORE` | Да | `true` | `true` для in-memory, `false` для DynamoDB |
| `AWS_REGION` | Нет\* | `us-east-1` | Регион для DynamoDB (для Yandex Cloud: `ru-central1`) |
| `DYNAMODB_ENDPOINT_URL` | Нет\* | \- | URL Document API endpoint (YDB) |
| `DYNAMODB_TABLE_PREFIX` | Нет | `airline` | Префикс для таблиц DynamoDB |
| `AUTO_CREATE_TABLES` | Нет | `false` | Автоматическое создание таблиц при запуске |
| `API_KEY` | Нет\*\* | \- | API-ключ для аутентификации запросов |

\* Обязательные если `USE_MEMORY_STORE=false`  
\*\* Обязательный если требуется аутентификация или для использования с MCP Gateway

## 🔧 Интеграция с MCP

Airline API можно использовать как MCP-сервер для AI агентов через Yandex AI Studio MCP Hub.

### Что такое MCP?

Model Context Protocol (MCP) — это протокол для интеграции внешних инструментов (tools) с AI-агентами. Airline API предоставляет следующие инструменты:

1. **get\_customer\_profile** — получить профиль клиента  
2. **change\_seat** — изменить место на рейсе  
3. **cancel\_trip** — отменить бронирование  
4. **add\_checked\_bag** — добавить багаж  
5. **set\_meal\_preference** — установить предпочтения питания  
6. **request\_assistance** — запросить специальную помощь

### Использование через MCP Gateway

После создания MCP Gateway используйте его URL в агентах:

from agents.mcp import MCPServerSse

\# URL MCP Gateway

MCP\_SERVER\_URL \= "https://\<gateway-id\>.mcpgw.serverless.yandexcloud.net/sse"

\# Создание MCP клиента

mcp\_server \= MCPServerSse(

    params={

        "url": MCP\_SERVER\_URL,

        "headers": {

            "Authorization": f"Api-Key {API\_KEY}"

        }

    },

    name="Airline Tools"

)

\# Использование в агенте

await mcp\_server.connect()

tools \= await mcp\_server.list\_tools()

## Прямое использование как MCP-сервера

Кроме работы через MCP Gateway, Airline API можно запустить и самостоятельно — как отдельный MCP-сервер по транспорту SSE. Пример подключения к SSE-эндпоинту:

import httpx

\# Подключение к SSE-эндпоинту

async with httpx.AsyncClient() as client:

    async with client.stream(

        "GET",

        f"{AIRLINE\_API\_URL}/sse",

        headers={"Authorization": f"Api-Key {API\_KEY}"}

    ) as response:

        async for line in response.aiter\_lines():

            \# Обработка SSE-событий

            print(line)

## 🔐 Аутентификация

Airline API поддерживает аутентификацию через API-ключи:

\# Запросы с API-ключом

curl http://localhost:8001/profile/test \\

  \-H "Authorization: Api-Key your\_api\_key\_here"

Для локальной разработки можно отключить проверку API-ключей (по умолчанию отключена).

## 🧪 Тестирование

### Ручное тестирование

\# Получить профиль

curl http://localhost:8001/profile/test\_user

\# Изменить место

curl \-X POST http://localhost:8001/seat \\

  \-H "Content-Type: application/json" \\

  \-d '{"profile\_id": "test\_user", "flight\_number": "OA476", "seat": "15F"}'

\# Проверить изменения

curl http://localhost:8001/profile/test\_user

### Управление DynamoDB

Если используется YDB Document API, управляйте таблицами с помощью скрипта:

cd airline-api/app/dynamodb

\# Создать таблицы

python manage\_airline\_db.py \--create

\# Проверить статус

python manage\_airline\_db.py \--status

\# Список профилей

python manage\_airline\_db.py \--list-profiles

\# Получить конкретный профиль

python manage\_airline\_db.py \--get-profile demo\_default\_thread

\# Удалить все данные (⚠️ ОСТОРОЖНО)

python manage\_airline\_db.py \--delete

## 📊 Мониторинг и логи

### Просмотр логов контейнера

\# Получить ID последней ревизии

REVISION\_ID=$(yc serverless container revision list \\

  \--container-name airline-api \\

  \--format json | jq \-r '.\[0\].id')

\# Просмотреть логи

yc serverless container revision logs ${REVISION\_ID}

\# Следить за логами в реальном времени

yc serverless container revision logs ${REVISION\_ID} \--follow

### Метрики

Yandex Cloud автоматически собирает метрики:

- количество запросов  
- время выполнения  
- ошибки  
- использование ресурсов

💡 Просмотр метрик в [консоли Yandex Cloud](https://console.cloud.yandex.ru/).

## 🚨 Устранение неполадок

### CORS-ошибки

Если фронтенд получает CORS-ошибки при обращении к Airline API в Yandex Cloud, используйте прокси через chatkit-agent вместо прямого обращения:

// Неправильно — прямое обращение к Airline API

const response \= await fetch('https://airline-api.yandexcloud.net/profile/user');

// Правильно — через chatkit-agent proxy

const response \= await fetch('http://localhost:8000/profiles/user');

### Проблемы с YDB Document API

\# Проверьте эндпоинт

echo $DYNAMODB\_ENDPOINT\_URL

\# Проверьте роли сервисного аккаунта

yc iam service-account list-access-bindings airline-api-sa

\# Проверьте таблицы

python app/dynamodb/manage\_airline\_db.py \--status

### Ошибки аутентификации

\# Проверьте API-ключ в Lockbox

yc lockbox secret get airline-api-key

\# Проверьте переменные окружения контейнера

yc serverless container revision get \<REVISION\_ID\>

## 📚 Полезные ссылки

### Документация Yandex Cloud

- [Serverless Containers](https://cloud.yandex.ru/docs/serverless-containers/)  
- [YDB Document API](https://cloud.yandex.ru/docs/ydb/docapi/)  
- [Container Registry](https://cloud.yandex.ru/docs/container-registry/)  
- [Lockbox](https://cloud.yandex.ru/docs/lockbox/)  
- [IAM-токены](https://cloud.yandex.ru/docs/iam/concepts/authorization/iam-token)  
- [Сервисные аккаунты](https://cloud.yandex.ru/docs/iam/concepts/users/service-accounts)

### Документация Yandex AI Studio

- [MCP Hub](https://yandex.cloud/ru/docs/ai-studio/concepts/mcp-hub/)  
- [MCP Gateway](https://yandex.cloud/ru/docs/ai-studio/operations/mcp-gateway/)  
- [AI-агенты документация](https://yandex.cloud/ru/docs/ai-studio/concepts/agents/)

### Документация FastAPI

- [FastAPI](https://fastapi.tiangolo.com/)  
- [Pydantic](https://docs.pydantic.dev/)  
- [Uvicorn](https://www.uvicorn.org/)

### Документация MCP

- [Model Context Protocol](https://modelcontextprotocol.io/)  
- [MCP Specification](https://spec.modelcontextprotocol.io/)

## 🤝 Поддержка

При возникновении проблем:

1. Проверьте логи контейнера: `yc serverless container revision logs <REVISION_ID>`  
2. Убедитесь, что все переменные окружения заданы правильно  
3. Проверьте доступность YDB Document API (если используется)  
4. Убедитесь, что сервисный аккаунт имеет все необходимые роли  
5. Проверьте CORS-настройки для фронтенд-интеграции

