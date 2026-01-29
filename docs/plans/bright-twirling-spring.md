# План: Python микросервис aivalone-proxy-telegram

## Цель
Создать Python микросервис для мониторинга Telegram чатов через MTProto API с интеграцией в существующую архитектуру через RabbitMQ.

## Технологический стек
- **Python 3.12**
- **Telethon** - MTProto клиент для Telegram
- **aio-pika** - асинхронный RabbitMQ клиент
- **aiohttp** - HTTP сервер для health checks
- **pydantic-settings** - конфигурация
- **structlog** - JSON логирование

---

## 1. Структура проекта

```
services/aivalone-proxy-telegram/
├── src/
│   ├── __init__.py
│   ├── main.py                         # Точка входа
│   ├── config/
│   │   ├── __init__.py
│   │   └── settings.py                 # Конфигурация из ENV
│   ├── domain/
│   │   ├── __init__.py
│   │   ├── models/
│   │   │   ├── __init__.py
│   │   │   ├── session.py              # SessionData value object
│   │   │   └── chat_monitor.py         # Состояние мониторинга
│   │   └── events/
│   │       ├── __init__.py
│   │       ├── incoming_message.py     # IncomingTelegramMessageEvent
│   │       └── access_lost.py          # AccessLostEvent
│   ├── application/
│   │   ├── __init__.py
│   │   ├── commands/
│   │   │   ├── __init__.py
│   │   │   ├── start_monitoring.py     # StartMonitoringChatCommand handler
│   │   │   ├── stop_monitoring.py      # StopMonitoringChatCommand handler
│   │   │   ├── join_chat.py            # JoinChatCommand handler
│   │   │   └── switch_source.py        # SwitchMonitoringSourceCommand handler
│   │   └── services/
│   │       ├── __init__.py
│   │       ├── monitor_manager.py      # Управление активными мониторами
│   │       ├── session_manager.py      # Управление Telegram сессиями
│   │       └── failover_service.py     # Логика failover
│   ├── infrastructure/
│   │   ├── __init__.py
│   │   ├── telegram/
│   │   │   ├── __init__.py
│   │   │   └── telethon_client.py      # Адаптер Telethon
│   │   ├── rabbitmq/
│   │   │   ├── __init__.py
│   │   │   ├── consumer.py             # Потребитель команд
│   │   │   └── publisher.py            # Отправитель событий
│   │   └── health/
│   │       ├── __init__.py
│   │       └── http_server.py          # Health check сервер
│   └── presentation/
│       ├── __init__.py
│       └── command_dispatcher.py       # Диспетчер команд
├── tests/
│   ├── __init__.py
│   └── unit/
├── Dockerfile
├── requirements.txt
├── .env.example
└── README.md
```

---

## 2. Интеграция с RabbitMQ

### Конфигурация (из messenger.yaml бекенда)
```yaml
exchange: aivalone_python_bridge (direct)
queues:
  - python_bridge_commands: binding_keys: [command]
  - python_bridge_events: binding_keys: [event]
```

### Входящие команды (JSON)

| Команда | Поля |
|---------|------|
| `start_monitoring_chat` | `chat_id`, `primary_session`, `backup_sessions_pool` |
| `stop_monitoring_chat` | `chat_id` |
| `join_chat` | `chat_id`, `session` |
| `switch_monitoring_source` | `chat_id`, `new_primary_session`, `backup_sessions_pool` |

### Исходящие события (JSON)

| Событие | Поля |
|---------|------|
| `incoming_telegram_message` | `chat_id`, `message_id`, `date`, `text`, `metadata` |
| `access_lost` | `chat_id`, `session_id`, `reason` |

### Формат SessionData
```json
{
  "id": "uuid",
  "api_hash": "xxx",
  "session_string": "xxx"
}
```

---

## 3. Основные компоненты

### MonitorManager
- Хранит `dict[chat_id, ChatMonitor]` активных мониторов
- Методы: `start_monitoring()`, `stop_monitoring()`, `switch_source()`

### SessionManager
- Хранит `dict[session_id, TelegramClient]` подключённых сессий
- Методы: `get_or_create_client()`, `disconnect_session()`, `disconnect_all()`

### FailoverService
- При `AccessLostEvent` переключается на backup сессию
- Если все сессии недоступны - останавливает мониторинг

### CommandDispatcher
- Маршрутизирует команды из очереди к обработчикам

---

## 4. Алгоритм мониторинга

```
1. Получаем StartMonitoringChatCommand
2. Создаём/получаем Telethon клиент для primary_session
3. Подписываемся на NewMessage events для chat_id
4. При получении сообщения:
   - Формируем IncomingTelegramMessageEvent
   - Публикуем в очередь python_bridge_events
5. При потере доступа:
   - Публикуем AccessLostEvent
   - Пытаемся переключиться на backup сессию
```

---

## 5. Файлы для создания/изменения

### Новые файлы (services/aivalone-proxy-telegram/)
| Файл | Описание |
|------|----------|
| `src/main.py` | Точка входа, graceful shutdown |
| `src/config/settings.py` | Pydantic Settings |
| `src/domain/models/session.py` | SessionData dataclass |
| `src/domain/models/chat_monitor.py` | ChatMonitor dataclass |
| `src/domain/events/incoming_message.py` | IncomingTelegramMessageEvent |
| `src/domain/events/access_lost.py` | AccessLostEvent |
| `src/application/commands/*.py` | Обработчики 4 команд |
| `src/application/services/monitor_manager.py` | MonitorManager |
| `src/application/services/session_manager.py` | SessionManager |
| `src/application/services/failover_service.py` | FailoverService |
| `src/infrastructure/telegram/telethon_client.py` | TelethonAdapter |
| `src/infrastructure/rabbitmq/consumer.py` | RabbitMQConsumer |
| `src/infrastructure/rabbitmq/publisher.py` | RabbitMQPublisher |
| `src/infrastructure/health/http_server.py` | HealthCheckServer |
| `src/presentation/command_dispatcher.py` | CommandDispatcher |
| `Dockerfile` | Multi-stage build |
| `requirements.txt` | Зависимости |
| `.env.example` | Пример переменных окружения |
| `README.md` | Документация сервиса |

### Изменения в бекенде
| Файл | Изменение |
|------|-----------|
| `services/aivalone-backend/docker-compose.yaml` | Добавить RabbitMQ сервис |
| `services/aivalone-backend/.env.example` | Добавить RABBITMQ_* переменные |

### Документация (docs/)
| Файл | Описание |
|------|----------|
| `docs/proxy-telegram/overview.md` | Обзор сервиса |
| `docs/proxy-telegram/architecture.md` | Архитектура |
| `docs/proxy-telegram/commands.md` | Протокол команд |
| `docs/proxy-telegram/events.md` | Протокол событий |
| `docs/proxy-telegram/deployment.md` | Развёртывание |
| `docs/core/overview.md` | Добавить ссылку на proxy-telegram |
| `docs/backend/monitoring/overview.md` | Добавить ссылку на proxy-telegram |

---

## 6. Конфигурация

### .env.example
```env
TELEGRAM_API_ID=12345678
TELEGRAM_API_HASH=your_api_hash

RABBITMQ_URL=amqp://guest:guest@rabbitmq:5672/
RABBITMQ_EXCHANGE=aivalone_python_bridge

HEALTH_CHECK_HOST=0.0.0.0
HEALTH_CHECK_PORT=8080

LOG_LEVEL=INFO
LOG_FORMAT=json

FAILOVER_RETRY_DELAY_SECONDS=5
MAX_RECONNECT_ATTEMPTS=3
SHUTDOWN_TIMEOUT_SECONDS=30
```

---

## 7. requirements.txt

```txt
telethon==1.36.0
aio-pika==9.4.0
aiohttp==3.9.3
pydantic-settings==2.2.1
structlog==24.1.0
aiodns==3.1.1
```

---

## 8. Docker интеграция

### Добавить в docker-compose.yaml бекенда
```yaml
services:
  rabbitmq:
    image: rabbitmq:3.12-management-alpine
    container_name: aivalone-rabbitmq
    ports:
      - "5672:5672"
      - "15672:15672"
    environment:
      - RABBITMQ_DEFAULT_USER=${RABBITMQ_USER:-guest}
      - RABBITMQ_DEFAULT_PASS=${RABBITMQ_PASS:-guest}
    volumes:
      - rabbitmq_data:/var/lib/rabbitmq
    healthcheck:
      test: rabbitmq-diagnostics -q ping
      interval: 10s
      timeout: 5s
      retries: 5
    networks:
      - aivalone-network

  proxy-telegram:
    build:
      context: ../aivalone-proxy-telegram
      dockerfile: Dockerfile
    container_name: aivalone-proxy-telegram
    environment:
      - TELEGRAM_API_ID=${TELEGRAM_API_ID}
      - TELEGRAM_API_HASH=${TELEGRAM_API_HASH}
      - RABBITMQ_URL=amqp://${RABBITMQ_USER:-guest}:${RABBITMQ_PASS:-guest}@rabbitmq:5672/
    depends_on:
      rabbitmq:
        condition: service_healthy
    networks:
      - aivalone-network
    restart: unless-stopped

volumes:
  rabbitmq_data:
```

---

## 9. Верификация

### Тестирование интеграции
1. Запустить `docker-compose up -d`
2. Проверить health: `curl http://localhost:8080/health`
3. Проверить RabbitMQ UI: `http://localhost:15672`
4. Отправить тестовую команду через RabbitMQ Management
5. Проверить логи: `docker logs aivalone-proxy-telegram`

### Тестирование команд
1. Отправить `start_monitoring_chat` с тестовым chat_id
2. Проверить подключение к чату в логах
3. Отправить сообщение в тестовый чат
4. Проверить `incoming_telegram_message` event в очереди

### Unit тесты
```bash
cd services/aivalone-proxy-telegram
pip install -r requirements-dev.txt
pytest tests/
```

---

## 10. Порядок реализации

1. **Базовая структура** - создать директории и __init__.py
2. **Конфигурация** - settings.py, .env.example
3. **Domain models** - SessionData, ChatMonitor, Events
4. **Infrastructure/RabbitMQ** - consumer, publisher
5. **Infrastructure/Telegram** - TelethonAdapter
6. **Application services** - MonitorManager, SessionManager
7. **Command handlers** - 4 обработчика команд
8. **Presentation** - CommandDispatcher
9. **Main** - точка входа, graceful shutdown
10. **Health server** - HTTP endpoints
11. **Dockerfile** - multi-stage build
12. **Docker-compose** - интеграция с бекендом
13. **Документация** - docs/proxy-telegram/*
14. **Обновить ссылки** - docs/core/, docs/backend/monitoring/
