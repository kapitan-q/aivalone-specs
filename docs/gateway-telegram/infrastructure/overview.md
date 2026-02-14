# Инфраструктура Gateway Telegram

## Структура проекта

```
gateway-telegram/
├── src/
│   ├── __init__.py
│   ├── main.py                          # Точка входа: запуск consumers и event loop
│   ├── config.py                        # pydantic-settings: типизированные настройки из env
│   │
│   ├── auth/                            # Модуль авторизации
│   │   ├── __init__.py
│   │   ├── handler.py                   # Обработчики: RequestAuthCode, SubmitAuthCode, Submit2FA
│   │   └── session_manager.py           # Создание/удаление .session файлов, генерация externalSessionId
│   │
│   ├── groups/                          # Модуль управления группами
│   │   ├── __init__.py
│   │   ├── handler.py                   # Обработчики: JoinGroup, LeaveGroup
│   │   └── listener.py                  # Event handler для новых сообщений в группах
│   │
│   ├── messaging/                       # RabbitMQ адаптеры
│   │   ├── __init__.py
│   │   ├── consumer.py                  # Подписка на очереди команд, десериализация, роутинг
│   │   ├── publisher.py                 # Публикация событий в exchange
│   │   └── schemas.py                   # Pydantic-модели для всех команд и событий
│   │
│   └── telegram/                        # Telethon-обёртки
│       ├── __init__.py
│       └── client_pool.py               # Пул TelegramClient: создание, кеширование, disconnect
│
├── tests/
│   ├── __init__.py
│   ├── unit/                            # Unit-тесты (мокированный Telethon и RabbitMQ)
│   │   ├── __init__.py
│   │   ├── test_auth_handler.py
│   │   ├── test_group_handler.py
│   │   └── test_session_manager.py
│   └── integration/                     # Integration-тесты (реальный RabbitMQ, мокированный Telegram)
│       └── __init__.py
│
├── sessions/                            # Volume для .session файлов (gitignored)
│   └── .gitkeep
│
├── Dockerfile
├── pyproject.toml                       # Зависимости, metadata, настройки ruff/mypy
├── .env.example                         # Пример переменных окружения
└── README.md
```

## Зависимости

### Runtime

| Пакет | Версия | Назначение |
|-------|--------|------------|
| `telethon` | >= 1.36 | MTProto клиент для Telegram API |
| `aio-pika` | >= 9.4 | Асинхронный AMQP клиент для RabbitMQ |
| `pydantic` | >= 2.0 | Валидация и сериализация данных (команды, события) |
| `pydantic-settings` | >= 2.0 | Типизированная конфигурация из переменных окружения |
| `structlog` | >= 24.0 | Структурированное логирование (JSON формат) |

### Development

| Пакет | Версия | Назначение |
|-------|--------|------------|
| `pytest` | >= 8.0 | Фреймворк тестирования |
| `pytest-asyncio` | >= 0.24 | Поддержка async тестов |
| `ruff` | latest | Линтинг и форматирование |
| `mypy` | latest | Статическая типизация |

## Docker

### Dockerfile

```dockerfile
FROM python:3.12-slim

WORKDIR /app

# Установка зависимостей
COPY pyproject.toml ./
RUN pip install --no-cache-dir .

# Копирование исходников
COPY src/ ./src/

# Каталог для session-файлов (монтируется как volume)
RUN mkdir -p /app/sessions

CMD ["python", "-m", "src.main"]
```

### Интеграция в docker-compose backend

Gateway Telegram добавляется в `docker-compose.yaml` backend-проекта как дополнительный сервис:

```yaml
services:
  gateway-telegram:
    build:
      context: ../gateway-telegram
      dockerfile: Dockerfile
    depends_on:
      rabbitmq:
        condition: service_healthy
    environment:
      - RABBITMQ_HOST=rabbitmq
      - RABBITMQ_PORT=5672
      - RABBITMQ_USER=${RABBITMQ_USER:-guest}
      - RABBITMQ_PASS=${RABBITMQ_PASS:-guest}
      - TELEGRAM_API_ID=${TELEGRAM_API_ID}
      - TELEGRAM_API_HASH=${TELEGRAM_API_HASH}
      - TELEGRAM_SESSION_DIR=/app/sessions
      - SERVICE_ACCOUNT_PHONE=${SERVICE_ACCOUNT_PHONE}
      - LOG_LEVEL=${GATEWAY_LOG_LEVEL:-INFO}
    volumes:
      - gateway_telegram_sessions:/app/sessions
    networks:
      - aivalone-network
    restart: unless-stopped

volumes:
  gateway_telegram_sessions:
    driver: local
```

**Ключевые моменты**:

* `depends_on: rabbitmq` — gateway стартует после RabbitMQ
* `aivalone-network` — та же сеть, что у backend (доступ к rabbitmq по имени сервиса)
* `gateway_telegram_sessions` — persistent volume для `.session` файлов
* Нет зависимости от PostgreSQL или Redis — gateway не использует БД

## Конфигурация (переменные окружения)

### RabbitMQ

| Переменная | Обязательна | По умолчанию | Описание |
|-----------|-------------|-------------|----------|
| `RABBITMQ_HOST` | нет | `rabbitmq` | Хост RabbitMQ |
| `RABBITMQ_PORT` | нет | `5672` | Порт AMQP |
| `RABBITMQ_USER` | нет | `guest` | Пользователь |
| `RABBITMQ_PASS` | нет | `guest` | Пароль |
| `RABBITMQ_VHOST` | нет | `/` | Virtual host |

### Telegram API

| Переменная | Обязательна | По умолчанию | Описание |
|-----------|-------------|-------------|----------|
| `TELEGRAM_API_ID` | **да** | — | API ID приложения (my.telegram.org) |
| `TELEGRAM_API_HASH` | **да** | — | API Hash приложения |
| `TELEGRAM_SESSION_DIR` | нет | `/app/sessions` | Путь к каталогу `.session` файлов |

### Сервисный аккаунт

| Переменная | Обязательна | По умолчанию | Описание |
|-----------|-------------|-------------|----------|
| `SERVICE_ACCOUNT_PHONE` | **да** | — | Номер телефона сервисного аккаунта |
| `SERVICE_ACCOUNT_SESSION` | нет | `service_account` | Имя `.session` файла сервисного аккаунта |

### Общие

| Переменная | Обязательна | По умолчанию | Описание |
|-----------|-------------|-------------|----------|
| `LOG_LEVEL` | нет | `INFO` | Уровень логирования (DEBUG, INFO, WARNING, ERROR) |
| `LOG_FORMAT` | нет | `json` | Формат логов (`json` для production, `console` для разработки) |

## RabbitMQ Topology

Gateway при старте декларирует необходимые exchanges и queues:

### Exchanges

| Exchange | Тип | Durable | Описание |
|----------|-----|---------|----------|
| `monitoring.commands` | direct | да | Команды от backend (gateway подписывается) |
| `listener.events` | topic | да | События от gateway (gateway публикует) |

### Queues

| Queue | Exchange | Routing Key | Описание |
|-------|----------|------------|----------|
| `gateway-telegram.auth.commands` | `monitoring.commands` | `telegram.auth` | Команды авторизации |
| `gateway-telegram.listener.commands` | `monitoring.commands` | `telegram.listener` | Команды управления группами |

### Routing keys для событий (публикация)

| Routing Key | Событие |
|-------------|---------|
| `telegram.auth.code_requested` | AuthCodeRequested |
| `telegram.auth.2fa_required` | Auth2FARequired |
| `telegram.auth.session_authorized` | SessionAuthorized |
| `telegram.auth.session_failed` | SessionFailed |
| `telegram.auth.session_expired` | SessionExpired |
| `telegram.group.joined` | GroupJoined |
| `telegram.group.join_failed` | GroupJoinFailed |
| `telegram.message.received` | MessageReceived |

## Хранение данных

### `.session` файлы

* Telethon создаёт SQLite-файлы с расширением `.session` для каждого авторизованного клиента
* Файлы хранятся в Docker volume `gateway_telegram_sessions`
* Имя файла = `externalSessionId` (генерируется gateway)
* Сервисный аккаунт использует фиксированное имя файла (env `SERVICE_ACCOUNT_SESSION`)

### Нет БД

Gateway **не использует базу данных**. Всё состояние:

* **Сессии** — `.session` файлы на volume
* **Активные клиенты** — in-memory `dict` в `client_pool.py`
* **Подписки на группы** — in-memory `dict` (восстанавливаются при старте через backend команды)

При перезапуске gateway, backend должен повторно отправить `StartSession` и `JoinGroup` для восстановления активных подписок.

## Сетевая схема

```
┌──────────────────────────────────────────────────────┐
│                  aivalone-network                      │
│                                                        │
│  ┌────────────┐    ┌────────────┐    ┌──────────────┐  │
│  │  php-fpm   │    │  rabbitmq  │    │   gateway-   │  │
│  │  (backend) │───→│  :5672     │←───│   telegram   │  │
│  └────────────┘    └────────────┘    └──────────────┘  │
│  ┌────────────┐    ┌────────────┐                      │
│  │  postgres  │    │   redis    │                      │
│  │  :5432     │    │   :6379    │                      │
│  └────────────┘    └────────────┘                      │
│                                                        │
└──────────────────────────────────────────────────────┘
                                            │
                                            │ MTProto
                                            ▼
                                   ┌──────────────────┐
                                   │  Telegram Servers │
                                   └──────────────────┘
```

Gateway общается только с RabbitMQ (внутри Docker-сети) и Telegram серверами (внешний трафик). Прямого доступа к PostgreSQL или Redis не требуется.

## Связанные документы

* [Gateway Telegram Overview](../overview.md)
* [Backend Infrastructure](../../backend/monitoring/infrastructure/overview.md)
* [Backend Docker Compose](../../../services/aivalone-backend/docker-compose.yaml)

## Статус реализации

* [ ] Спецификация (этот файл)
* [ ] Создание репозитория
* [ ] `pyproject.toml` с зависимостями
* [ ] `Dockerfile`
* [ ] Интеграция в docker-compose backend
* [ ] `config.py` (pydantic-settings)
* [ ] `main.py` (точка входа)
* [ ] RabbitMQ consumer/publisher
* [ ] Telegram client pool
