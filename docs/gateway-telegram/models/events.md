# Модели событий (Gateway → Backend)

## Описание

Все события публикуются в RabbitMQ exchange `listener.events` (тип topic) с routing key по шаблону `telegram.<category>.<action>`. Каждое событие сериализуется из Pydantic-модели в JSON.

## Базовая модель

```python
class BaseEvent(BaseModel):
    """Базовый класс для всех исходящих событий."""
    event: str                      # Тип события (wire name для backend)
    session_id: str                 # UUID сессии в backend
    messenger_type: str = "telegram"
    correlation_id: str | None = None  # UUID корреляции (есть для ответов на команды)
    timestamp: str                  # ISO 8601 UTC
```

---

## AuthCodeRequested

Код авторизации успешно отправлен пользователю в Telegram.

**Wire name (для backend)**: `AuthCodeSent`
**Routing key**: `telegram.auth.code_requested`

### Поля

| Поле | Тип | Описание |
|------|-----|----------|
| `event` | string | `"AuthCodeSent"` |
| `sessionId` | string (UUID) | ID сессии в backend |
| `messengerType` | string | `"telegram"` |
| `codeLength` | int | Длина ожидаемого кода (обычно 5) |
| `correlationId` | string (UUID) | ID корреляции из исходной команды |
| `timestamp` | string (ISO 8601) | Время генерации события |

### JSON-пример

```json
{
    "event": "AuthCodeSent",
    "sessionId": "550e8400-e29b-41d4-a716-446655440000",
    "messengerType": "telegram",
    "codeLength": 5,
    "correlationId": "7c9e6679-7425-40de-944b-e07fc1f90ae7",
    "timestamp": "2024-01-15T10:30:00Z"
}
```

### Когда генерируется

После успешного вызова `client.send_code_request(phone)`. Telethon возвращает `SentCode` с длиной кода.

### Pydantic-модель

```python
class AuthCodeRequestedEvent(BaseEvent):
    event: str = "AuthCodeSent"
    code_length: int = Field(alias="codeLength")
```

---

## Auth2FARequired

Telegram запросил пароль двухфакторной аутентификации.

**Wire name (для backend)**: `Auth2FARequired`
**Routing key**: `telegram.auth.2fa_required`

### Поля

| Поле | Тип | Описание |
|------|-----|----------|
| `event` | string | `"Auth2FARequired"` |
| `sessionId` | string (UUID) | ID сессии в backend |
| `messengerType` | string | `"telegram"` |
| `hint` | string | Подсказка пароля (например `"***word"`) |
| `correlationId` | string (UUID) | ID корреляции |
| `timestamp` | string (ISO 8601) | Время генерации |

### JSON-пример

```json
{
    "event": "Auth2FARequired",
    "sessionId": "550e8400-e29b-41d4-a716-446655440000",
    "messengerType": "telegram",
    "hint": "***word",
    "correlationId": "7c9e6679-7425-40de-944b-e07fc1f90ae7",
    "timestamp": "2024-01-15T10:31:00Z"
}
```

### Когда генерируется

Когда `client.sign_in(code=...)` выбрасывает `SessionPasswordNeededError`. Hint извлекается из `client(GetPasswordRequest())`.

### Pydantic-модель

```python
class Auth2FARequiredEvent(BaseEvent):
    event: str = "Auth2FARequired"
    hint: str
```

---

## SessionAuthorized

Сессия успешно авторизована в Telegram.

**Wire name (для backend)**: `AuthSuccess`
**Routing key**: `telegram.auth.session_authorized`

### Поля

| Поле | Тип | Описание |
|------|-----|----------|
| `event` | string | `"AuthSuccess"` |
| `sessionId` | string (UUID) | ID сессии в backend |
| `messengerType` | string | `"telegram"` |
| `externalSessionId` | string | ID сессии в gateway (имя .session файла) |
| `displayName` | string | Имя пользователя в Telegram |
| `correlationId` | string (UUID) | ID корреляции |
| `timestamp` | string (ISO 8601) | Время генерации |

### JSON-пример

```json
{
    "event": "AuthSuccess",
    "sessionId": "550e8400-e29b-41d4-a716-446655440000",
    "messengerType": "telegram",
    "externalSessionId": "a1b2c3d4-e5f6-7890-abcd-ef1234567890",
    "displayName": "Иван Петров",
    "correlationId": "7c9e6679-7425-40de-944b-e07fc1f90ae7",
    "timestamp": "2024-01-15T10:32:00Z"
}
```

### Когда генерируется

После успешного `client.sign_in(code=...)` или `client.sign_in(password=...)`. `displayName` извлекается из `client.get_me()` как `f"{first_name} {last_name}"`.

### Pydantic-модель

```python
class SessionAuthorizedEvent(BaseEvent):
    event: str = "AuthSuccess"
    external_session_id: str = Field(alias="externalSessionId")
    display_name: str = Field(alias="displayName")
```

---

## SessionFailed

Ошибка авторизации сессии.

**Wire name (для backend)**: `AuthFailed`
**Routing key**: `telegram.auth.session_failed`

### Поля

| Поле | Тип | Описание |
|------|-----|----------|
| `event` | string | `"AuthFailed"` |
| `sessionId` | string (UUID) | ID сессии в backend |
| `messengerType` | string | `"telegram"` |
| `reason` | string | Код ошибки (см. [errors.md](errors.md)) |
| `message` | string | Человекочитаемое описание ошибки |
| `canRetry` | boolean | Можно ли повторить операцию |
| `correlationId` | string (UUID) | ID корреляции |
| `timestamp` | string (ISO 8601) | Время генерации |

### JSON-пример

```json
{
    "event": "AuthFailed",
    "sessionId": "550e8400-e29b-41d4-a716-446655440000",
    "messengerType": "telegram",
    "reason": "PHONE_CODE_INVALID",
    "message": "Неверный код авторизации",
    "canRetry": true,
    "correlationId": "7c9e6679-7425-40de-944b-e07fc1f90ae7",
    "timestamp": "2024-01-15T10:33:00Z"
}
```

### Когда генерируется

При любой ошибке Telethon во время авторизации. Подробный маппинг ошибок — [errors.md](errors.md).

### Pydantic-модель

```python
class SessionFailedEvent(BaseEvent):
    event: str = "AuthFailed"
    reason: str
    message: str
    can_retry: bool = Field(alias="canRetry")
```

---

## SessionExpired

Ранее авторизованная сессия истекла или была отозвана.

**Wire name (для backend)**: `SessionExpired`
**Routing key**: `telegram.auth.session_expired`

### Поля

| Поле | Тип | Описание |
|------|-----|----------|
| `event` | string | `"SessionExpired"` |
| `sessionId` | string (UUID) | ID сессии в backend |
| `messengerType` | string | `"telegram"` |
| `reason` | string | Причина: `"session_revoked"`, `"auth_key_duplicated"`, `"connection_lost"` |
| `timestamp` | string (ISO 8601) | Время генерации |

### JSON-пример

```json
{
    "event": "SessionExpired",
    "sessionId": "550e8400-e29b-41d4-a716-446655440000",
    "messengerType": "telegram",
    "reason": "session_revoked",
    "timestamp": "2024-01-15T14:00:00Z"
}
```

### Когда генерируется

* При получении `AuthKeyDuplicatedError` или `SessionRevokedError` от Telethon
* При невозможности reconnect к Telegram (post-MVP: health monitor)

> **Примечание**: `correlationId` отсутствует — это событие инициировано Telegram, а не backend.

### Pydantic-модель

```python
class SessionExpiredEvent(BaseEvent):
    event: str = "SessionExpired"
    reason: str
    correlation_id: str | None = None
```

---

## GroupJoined

Успешно присоединились к группе/каналу.

**Wire name (для backend)**: `GroupJoined`
**Routing key**: `telegram.group.joined`

### Поля

| Поле | Тип | Описание |
|------|-----|----------|
| `event` | string | `"GroupJoined"` |
| `groupId` | string (UUID) | ID группы в backend |
| `externalGroupId` | string | `@username` или chat ID |
| `messengerType` | string | `"telegram"` |
| `groupTitle` | string | Название группы/канала |
| `memberCount` | int | Количество участников |
| `correlationId` | string (UUID) | ID корреляции |
| `timestamp` | string (ISO 8601) | Время генерации |

### JSON-пример

```json
{
    "event": "GroupJoined",
    "groupId": "660e8400-e29b-41d4-a716-446655440001",
    "externalGroupId": "@cryptonews",
    "messengerType": "telegram",
    "groupTitle": "Crypto News Channel",
    "memberCount": 50000,
    "correlationId": "8d9e6679-7425-40de-944b-e07fc1f90ae8",
    "timestamp": "2024-01-15T11:00:00Z"
}
```

### Когда генерируется

После успешного `client.join_chat(external_group_id)` или получения информации о группе через `GetFullChannelRequest`. `groupTitle` и `memberCount` извлекаются из ответа Telegram API.

### Pydantic-модель

```python
class GroupJoinedEvent(BaseEvent):
    event: str = "GroupJoined"
    group_id: str = Field(alias="groupId")
    external_group_id: str = Field(alias="externalGroupId")
    group_title: str = Field(alias="groupTitle")
    member_count: int = Field(alias="memberCount")
```

---

## GroupJoinFailed

Ошибка присоединения к группе/каналу.

**Wire name (для backend)**: `GroupJoinFailed`
**Routing key**: `telegram.group.join_failed`

### Поля

| Поле | Тип | Описание |
|------|-----|----------|
| `event` | string | `"GroupJoinFailed"` |
| `groupId` | string (UUID) | ID группы в backend |
| `externalGroupId` | string | `@username` или chat ID |
| `messengerType` | string | `"telegram"` |
| `reason` | string | Код ошибки (см. [errors.md](errors.md)) |
| `message` | string | Человекочитаемое описание |
| `correlationId` | string (UUID) | ID корреляции |
| `timestamp` | string (ISO 8601) | Время генерации |

### JSON-пример

```json
{
    "event": "GroupJoinFailed",
    "groupId": "660e8400-e29b-41d4-a716-446655440001",
    "externalGroupId": "@cryptonews",
    "messengerType": "telegram",
    "reason": "CHAT_INVALID",
    "message": "Группа не существует или недоступна",
    "correlationId": "8d9e6679-7425-40de-944b-e07fc1f90ae8",
    "timestamp": "2024-01-15T11:00:00Z"
}
```

### Pydantic-модель

```python
class GroupJoinFailedEvent(BaseEvent):
    event: str = "GroupJoinFailed"
    group_id: str = Field(alias="groupId")
    external_group_id: str = Field(alias="externalGroupId")
    reason: str
    message: str
```

---

## MessageReceived

Получено новое сообщение из отслеживаемой группы. Отправляется **одно событие на одно сообщение** — backend сам определяет, какие groupId привязаны к этому чату.

**Wire name (для backend)**: `MessageReceived`
**Routing key**: `telegram.message.received`

### Поля

| Поле | Тип | Описание |
|------|-----|----------|
| `event` | string | `"MessageReceived"` |
| `messengerType` | string | `"telegram"` |
| `chatId` | int | Числовой Telegram chat ID |
| `externalGroupId` | string | Строковый chat ID (для совместимости) |
| `messageId` | string | ID сообщения в Telegram |
| `content` | string | Текст сообщения |
| `senderName` | string | Имя отправителя |
| `sentAt` | string (ISO 8601) | Время отправки в Telegram |
| `metadata` | object | Дополнительная информация |
| `metadata.hasMedia` | boolean | Содержит ли сообщение медиа |
| `metadata.replyToMessageId` | string \| null | ID сообщения, на которое это ответ |
| `timestamp` | string (ISO 8601) | Время генерации события |

> **`group_id` отсутствует!** Backend определяет группы по `chatId`/`externalGroupId`. Это позволяет gateway отправлять одно событие даже если 10 пользователей мониторят одну группу.

> **`correlationId` и `sessionId` отсутствуют** — это push-событие.

### JSON-пример

```json
{
    "event": "MessageReceived",
    "messengerType": "telegram",
    "chatId": -1001234567890,
    "externalGroupId": "-1001234567890",
    "messageId": "123456",
    "content": "Bitcoin reached new ATH today!",
    "senderName": "CryptoBot",
    "sentAt": "2024-01-15T14:30:00Z",
    "metadata": {
        "hasMedia": false,
        "replyToMessageId": null
    },
    "timestamp": "2024-01-15T14:30:01Z"
}
```

### Когда генерируется

При срабатывании Telethon event handler `events.NewMessage` для группы, зарегистрированной в MonitoringRegistry (`is_monitored(chat_id) == True`). Данные извлекаются из `event.message`.

### Pydantic-модель

```python
class MessageMetadata(BaseModel):
    has_media: bool = Field(alias="hasMedia", default=False)
    reply_to_message_id: str | None = Field(alias="replyToMessageId", default=None)

class MessageReceivedEvent(BaseModel):
    event: str = "MessageReceived"
    messenger_type: str = Field(alias="messengerType", default="telegram")
    chat_id: int = Field(alias="chatId")
    external_group_id: str = Field(alias="externalGroupId")
    message_id: str = Field(alias="messageId")
    content: str
    sender_name: str = Field(alias="senderName")
    sent_at: str = Field(alias="sentAt")
    metadata: MessageMetadata
    timestamp: str
```

---

## Связанные документы

* [Models Overview](overview.md)
* [Commands](commands.md)
* [Errors](errors.md)
* [Messaging Overview](../messaging/overview.md)
* [Backend Listener Event Consumer](../../backend/monitoring/infrastructure/listener-event-consumer.md)
