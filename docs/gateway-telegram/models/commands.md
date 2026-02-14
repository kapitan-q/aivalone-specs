# Модели команд (Backend → Gateway)

## Описание

Все команды поступают из RabbitMQ exchange `monitoring.commands` в формате JSON. Каждая команда десериализуется в соответствующую Pydantic-модель и маршрутизируется к обработчику.

## Базовая модель

```python
class BaseCommand(BaseModel):
    """Базовый класс для всех входящих команд."""
    command: str                    # Тип команды (wire name из backend)
    session_id: str                 # UUID сессии в backend (MonitoringSession.id)
    messenger_type: str             # Тип мессенджера ("telegram")
    correlation_id: str             # UUID для связи команды с ответным событием
```

> **Примечание**: поле `command` содержит wire name из backend (например `InitAuth`), которое используется для маршрутизации к правильному обработчику.

---

## RequestAuthCode

Запрос отправки кода авторизации на номер телефона.

**Wire name (backend)**: `InitAuth`

### Поля

| Поле | Тип | Обязательно | Описание |
|------|-----|-------------|----------|
| `command` | string | да | `"InitAuth"` |
| `sessionId` | string (UUID) | да | ID сессии в backend |
| `messengerType` | string | да | `"telegram"` |
| `authData.phoneNumber` | string | да | Номер телефона в формате `+7XXXXXXXXXX` |
| `correlationId` | string (UUID) | да | ID корреляции |

### JSON-пример

```json
{
    "command": "InitAuth",
    "sessionId": "550e8400-e29b-41d4-a716-446655440000",
    "messengerType": "telegram",
    "authData": {
        "phoneNumber": "+79001234567"
    },
    "correlationId": "7c9e6679-7425-40de-944b-e07fc1f90ae7"
}
```

### Pydantic-модель

```python
class AuthData(BaseModel):
    phone_number: str = Field(alias="phoneNumber", pattern=r"^\+\d{10,15}$")

class RequestAuthCodeCommand(BaseCommand):
    auth_data: AuthData = Field(alias="authData")
```

### Валидация

* `phoneNumber` — строка, начинается с `+`, 10-15 цифр
* `messengerType` — строго `"telegram"`
* `sessionId` — валидный UUID

---

## SubmitAuthCode

Отправка кода подтверждения, полученного пользователем в Telegram.

**Wire name (backend)**: `SubmitCode`

### Поля

| Поле | Тип | Обязательно | Описание |
|------|-----|-------------|----------|
| `command` | string | да | `"SubmitCode"` |
| `sessionId` | string (UUID) | да | ID сессии в backend |
| `code` | string | да | Код подтверждения (5 цифр) |
| `correlationId` | string (UUID) | да | ID корреляции |

### JSON-пример

```json
{
    "command": "SubmitCode",
    "sessionId": "550e8400-e29b-41d4-a716-446655440000",
    "code": "12345",
    "correlationId": "7c9e6679-7425-40de-944b-e07fc1f90ae7"
}
```

### Pydantic-модель

```python
class SubmitAuthCodeCommand(BaseCommand):
    code: str = Field(min_length=4, max_length=8)
```

### Валидация

* `code` — строка, 4-8 символов (Telegram может менять длину кода)
* `sessionId` должен соответствовать активному auth flow в `AuthState`

---

## Submit2FA

Отправка пароля двухфакторной аутентификации.

**Wire name (backend)**: `Submit2FA`

### Поля

| Поле | Тип | Обязательно | Описание |
|------|-----|-------------|----------|
| `command` | string | да | `"Submit2FA"` |
| `sessionId` | string (UUID) | да | ID сессии в backend |
| `password` | string | да | 2FA пароль |
| `correlationId` | string (UUID) | да | ID корреляции |

### JSON-пример

```json
{
    "command": "Submit2FA",
    "sessionId": "550e8400-e29b-41d4-a716-446655440000",
    "password": "my_secure_password",
    "correlationId": "7c9e6679-7425-40de-944b-e07fc1f90ae7"
}
```

### Pydantic-модель

```python
class Submit2FACommand(BaseCommand):
    password: str = Field(min_length=1)
```

### Валидация

* `password` — непустая строка
* `sessionId` должен соответствовать auth flow в состоянии `AWAITING_2FA`

---

## StartSession

Запуск ранее авторизованной сессии (подключение TelegramClient).

**Wire name (backend)**: `StartSession`

### Поля

| Поле | Тип | Обязательно | Описание |
|------|-----|-------------|----------|
| `command` | string | да | `"StartSession"` |
| `sessionId` | string (UUID) | да | ID сессии в backend |
| `externalSessionId` | string | да | ID сессии в gateway (имя .session файла) |
| `messengerType` | string | да | `"telegram"` |
| `correlationId` | string (UUID) | да | ID корреляции |

### JSON-пример

```json
{
    "command": "StartSession",
    "sessionId": "550e8400-e29b-41d4-a716-446655440000",
    "externalSessionId": "a1b2c3d4-e5f6-7890-abcd-ef1234567890",
    "messengerType": "telegram",
    "correlationId": "7c9e6679-7425-40de-944b-e07fc1f90ae7"
}
```

### Pydantic-модель

```python
class StartSessionCommand(BaseCommand):
    external_session_id: str = Field(alias="externalSessionId")
```

### Валидация

* `externalSessionId` — непустая строка, должен соответствовать существующему `.session` файлу
* Файл `{externalSessionId}.session` должен существовать в `TELEGRAM_SESSION_DIR`

---

## StopSession

Остановка сессии: logout из Telegram, удаление клиента из пула.

**Wire name (backend)**: `StopSession`

### Поля

| Поле | Тип | Обязательно | Описание |
|------|-----|-------------|----------|
| `command` | string | да | `"StopSession"` |
| `sessionId` | string (UUID) | да | ID сессии в backend |
| `externalSessionId` | string | да | ID сессии в gateway |
| `messengerType` | string | да | `"telegram"` |
| `correlationId` | string (UUID) | да | ID корреляции |

### JSON-пример

```json
{
    "command": "StopSession",
    "sessionId": "550e8400-e29b-41d4-a716-446655440000",
    "externalSessionId": "a1b2c3d4-e5f6-7890-abcd-ef1234567890",
    "messengerType": "telegram",
    "correlationId": "7c9e6679-7425-40de-944b-e07fc1f90ae7"
}
```

### Pydantic-модель

```python
class StopSessionCommand(BaseCommand):
    external_session_id: str = Field(alias="externalSessionId")
```

---

## JoinGroup

Подписка на Telegram-группу/канал для получения сообщений.

**Wire name (backend)**: `JoinGroup`

### Поля

| Поле | Тип | Обязательно | Описание |
|------|-----|-------------|----------|
| `command` | string | да | `"JoinGroup"` |
| `groupId` | string (UUID) | да | ID группы в backend |
| `externalGroupId` | string | да | `@username` или chat ID |
| `messengerType` | string | да | `"telegram"` |
| `sessionId` | string \| null | нет | `externalSessionId` для приватных, `null` для публичных |
| `correlationId` | string (UUID) | да | ID корреляции |

### JSON-пример (публичная группа)

```json
{
    "command": "JoinGroup",
    "groupId": "660e8400-e29b-41d4-a716-446655440001",
    "externalGroupId": "@cryptonews",
    "messengerType": "telegram",
    "sessionId": null,
    "correlationId": "8d9e6679-7425-40de-944b-e07fc1f90ae8"
}
```

### JSON-пример (приватная группа)

```json
{
    "command": "JoinGroup",
    "groupId": "660e8400-e29b-41d4-a716-446655440001",
    "externalGroupId": "-1001234567890",
    "messengerType": "telegram",
    "sessionId": "a1b2c3d4-e5f6-7890-abcd-ef1234567890",
    "correlationId": "8d9e6679-7425-40de-944b-e07fc1f90ae8"
}
```

### Pydantic-модель

```python
class JoinGroupCommand(BaseModel):
    command: str
    group_id: str = Field(alias="groupId")
    external_group_id: str = Field(alias="externalGroupId")
    messenger_type: str = Field(alias="messengerType")
    session_id: str | None = Field(alias="sessionId", default=None)
    correlation_id: str = Field(alias="correlationId")
```

### Логика маршрутизации

* `sessionId == null` → использовать сервисный аккаунт (публичная группа)
* `sessionId != null` → найти клиент по `externalSessionId` в пуле (приватная группа)

### Валидация

* `externalGroupId` — непустая строка (может быть `@username`, invite link, или числовой chat ID)
* Если `sessionId != null` — клиент должен быть в пуле и авторизован

---

## LeaveGroup

Отписка от Telegram-группы/канала.

**Wire name (backend)**: `LeaveGroup`

### Поля

| Поле | Тип | Обязательно | Описание |
|------|-----|-------------|----------|
| `command` | string | да | `"LeaveGroup"` |
| `groupId` | string (UUID) | да | ID группы в backend |
| `externalGroupId` | string | да | `@username` или chat ID |
| `messengerType` | string | да | `"telegram"` |
| `sessionId` | string \| null | нет | `externalSessionId` или `null` |
| `correlationId` | string (UUID) | да | ID корреляции |

### JSON-пример

```json
{
    "command": "LeaveGroup",
    "groupId": "660e8400-e29b-41d4-a716-446655440001",
    "externalGroupId": "@cryptonews",
    "messengerType": "telegram",
    "sessionId": null,
    "correlationId": "8d9e6679-7425-40de-944b-e07fc1f90ae8"
}
```

### Pydantic-модель

```python
class LeaveGroupCommand(BaseModel):
    command: str
    group_id: str = Field(alias="groupId")
    external_group_id: str = Field(alias="externalGroupId")
    messenger_type: str = Field(alias="messengerType")
    session_id: str | None = Field(alias="sessionId", default=None)
    correlation_id: str = Field(alias="correlationId")
```

---

## Маршрутизация команд

Consumer десериализует JSON и по полю `command` определяет обработчик:

```python
COMMAND_ROUTES = {
    "InitAuth": ("auth", RequestAuthCodeCommand, auth_handler.handle_request_auth_code),
    "SubmitCode": ("auth", SubmitAuthCodeCommand, auth_handler.handle_submit_auth_code),
    "Submit2FA": ("auth", Submit2FACommand, auth_handler.handle_submit_2fa),
    "StartSession": ("sessions", StartSessionCommand, session_handler.handle_start_session),
    "StopSession": ("sessions", StopSessionCommand, session_handler.handle_stop_session),
    "JoinGroup": ("groups", JoinGroupCommand, group_handler.handle_join_group),
    "LeaveGroup": ("groups", LeaveGroupCommand, group_handler.handle_leave_group),
}
```

## Связанные документы

* [Models Overview](overview.md)
* [Events](events.md)
* [Messaging Overview](../messaging/overview.md)
* [Backend Authorize Session](../../backend/monitoring/processes/authorize-session.md)
* [Backend Subscribe to Group](../../backend/monitoring/processes/subscribe-to-group.md)
