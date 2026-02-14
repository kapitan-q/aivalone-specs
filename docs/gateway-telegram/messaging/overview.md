# Messaging — Обзор

## Назначение

Модуль `messaging` обеспечивает всю коммуникацию с backend через RabbitMQ: приём команд, публикацию событий, сериализацию/десериализацию, маршрутизацию к обработчикам.

## Зона ответственности

### Отвечает за:

* Подключение к RabbitMQ (robust connection с автоматическим reconnect)
* Декларацию exchanges и queues при старте
* Подписку на очереди команд
* Десериализацию JSON → Pydantic-модель
* Маршрутизацию команд к обработчикам по полю `command`
* Сериализацию Pydantic-модель → JSON для публикации событий
* ACK/NACK управление сообщениями
* Structured logging каждого входящего/исходящего сообщения

### НЕ отвечает за:

* Бизнес-логику обработки команд (делегируется модулям auth, groups, sessions)
* Управление Telegram-клиентами
* Хранение состояния

## Компоненты

| Компонент | Файл | Описание |
|-----------|------|----------|
| `Consumer` | `src/messaging/consumer.py` | Подписка на queues, десериализация, маршрутизация |
| `Publisher` | `src/messaging/publisher.py` | Публикация событий в exchange с routing key |
| `Router` | `src/messaging/consumer.py` | Маппинг `command` name → (model, handler) |
| `Schemas` | `src/messaging/schemas.py` | Pydantic-модели команд и событий |

## RabbitMQ Topology

### Exchanges

| Exchange | Тип | Durable | Создаётся | Описание |
|----------|-----|---------|-----------|----------|
| `monitoring.commands` | direct | да | Backend | Команды от backend |
| `listener.events` | topic | да | Gateway | События от gateway |

> Gateway при старте **декларирует** оба exchange (idempotent operation). Если exchange уже существует — ничего не происходит.

### Queues

| Queue | Exchange | Binding Key | Durable | Описание |
|-------|----------|-------------|---------|----------|
| `gateway-telegram.auth` | `monitoring.commands` | `telegram.auth` | да | Команды авторизации |
| `gateway-telegram.listener` | `monitoring.commands` | `telegram.listener` | да | Команды управления группами и сессиями |

### Routing Keys для публикации событий

| Routing Key | Событие | Категория |
|-------------|---------|-----------|
| `telegram.auth.code_requested` | AuthCodeSent | auth |
| `telegram.auth.2fa_required` | Auth2FARequired | auth |
| `telegram.auth.session_authorized` | AuthSuccess | auth |
| `telegram.auth.session_failed` | AuthFailed | auth |
| `telegram.auth.session_expired` | SessionExpired | sessions |
| `telegram.group.joined` | GroupJoined | groups |
| `telegram.group.join_failed` | GroupJoinFailed | groups |
| `telegram.message.received` | MessageReceived | groups |

## Формат сообщений

### Входящие (команды)

```json
{
    "command": "InitAuth",
    "sessionId": "uuid",
    "messengerType": "telegram",
    "correlationId": "uuid",
    ... // command-specific fields
}
```

Поле `command` определяет тип. Consumer использует его для выбора Pydantic-модели и обработчика.

### Исходящие (события)

```json
{
    "event": "AuthCodeSent",
    "sessionId": "uuid",
    "messengerType": "telegram",
    "correlationId": "uuid",
    "timestamp": "2024-01-15T10:30:00Z",
    ... // event-specific fields
}
```

Все события включают `timestamp` (ISO 8601 UTC). `correlationId` присутствует для ответов на команды, отсутствует для push-событий (`SessionExpired`, `MessageReceived`).

### AMQP Properties

| Property | Значение | Описание |
|----------|---------|----------|
| `content_type` | `application/json` | Формат тела |
| `delivery_mode` | `2` (persistent) | Сохранять при перезапуске RabbitMQ |
| `correlation_id` | UUID | Дублирует `correlationId` из тела (для AMQP-совместимости) |
| `message_id` | UUID | Уникальный ID сообщения |
| `timestamp` | Unix timestamp | Время публикации |

## Consumer — архитектура

### Инициализация

```python
async def start(self):
    connection = await aio_pika.connect_robust(settings.rabbitmq_url)
    channel = await connection.channel()
    await channel.set_qos(prefetch_count=10)

    # Declare exchanges
    commands_exchange = await channel.declare_exchange(
        "monitoring.commands", aio_pika.ExchangeType.DIRECT, durable=True
    )
    self._events_exchange = await channel.declare_exchange(
        "listener.events", aio_pika.ExchangeType.TOPIC, durable=True
    )

    # Declare and bind queues
    auth_queue = await channel.declare_queue("gateway-telegram.auth", durable=True)
    await auth_queue.bind(commands_exchange, routing_key="telegram.auth")
    await auth_queue.consume(self._on_message)

    listener_queue = await channel.declare_queue("gateway-telegram.listener", durable=True)
    await listener_queue.bind(commands_exchange, routing_key="telegram.listener")
    await listener_queue.consume(self._on_message)
```

### Обработка сообщения

```python
async def _on_message(self, message: aio_pika.IncomingMessage):
    async with message.process():  # auto-ACK при успехе, NACK при исключении
        body = json.loads(message.body)
        command_type = body.get("command")

        route = COMMAND_ROUTES.get(command_type)
        if not route:
            logger.warning("unknown_command", command=command_type)
            return  # ACK и пропускаем

        model_class, handler = route
        try:
            command = model_class.model_validate(body)
        except ValidationError as e:
            logger.error("validation_error", command=command_type, errors=str(e))
            return  # ACK — невалидное сообщение не имеет смысла повторять

        await handler(command)
```

### Маршрутизация

```python
COMMAND_ROUTES = {
    # Auth commands (queue: gateway-telegram.auth)
    "InitAuth":     (RequestAuthCodeCommand, auth_handler.handle_request_auth_code),
    "SubmitCode":   (SubmitAuthCodeCommand,  auth_handler.handle_submit_auth_code),
    "Submit2FA":    (Submit2FACommand,       auth_handler.handle_submit_2fa),

    # Session commands (queue: gateway-telegram.listener)
    "StartSession": (StartSessionCommand,    session_handler.handle_start_session),
    "StopSession":  (StopSessionCommand,     session_handler.handle_stop_session),

    # Group commands (queue: gateway-telegram.listener)
    "JoinGroup":    (JoinGroupCommand,       group_handler.handle_join_group),
    "LeaveGroup":   (LeaveGroupCommand,      group_handler.handle_leave_group),
}
```

## Publisher — архитектура

### Публикация события

```python
class Publisher:
    async def publish(self, event: BaseEvent, routing_key: str):
        body = event.model_dump_json(by_alias=True)
        message = aio_pika.Message(
            body=body.encode(),
            content_type="application/json",
            delivery_mode=aio_pika.DeliveryMode.PERSISTENT,
            correlation_id=event.correlation_id,
            message_id=str(uuid4()),
            timestamp=datetime.now(UTC),
        )
        await self._events_exchange.publish(message, routing_key=routing_key)
        logger.info("event_published", event=event.event, routing_key=routing_key)
```

### Маппинг событий → routing keys

```python
EVENT_ROUTING_KEYS = {
    "AuthCodeSent":     "telegram.auth.code_requested",
    "Auth2FARequired":  "telegram.auth.2fa_required",
    "AuthSuccess":      "telegram.auth.session_authorized",
    "AuthFailed":       "telegram.auth.session_failed",
    "SessionExpired":   "telegram.auth.session_expired",
    "GroupJoined":      "telegram.group.joined",
    "GroupJoinFailed":  "telegram.group.join_failed",
    "MessageReceived":  "telegram.message.received",
}
```

## Обработка ошибок

| Ситуация | Действие |
|----------|---------|
| RabbitMQ connection lost | `aio_pika.connect_robust` автоматически reconnect |
| Невалидный JSON | ACK + логирование (повтор бессмыслен) |
| Неизвестная команда | ACK + логирование |
| Ошибка валидации Pydantic | ACK + логирование |
| Ошибка в handler | NACK + requeue (сообщение вернётся в очередь) |
| Ошибка публикации события | Retry с exponential backoff, затем логирование |

## Связанные документы

* [Gateway Telegram Overview](../overview.md)
* [Models — Commands](../models/commands.md)
* [Models — Events](../models/events.md)
* [Backend Monitoring Infrastructure](../../backend/monitoring/infrastructure/overview.md)
