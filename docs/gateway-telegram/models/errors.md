# Коды ошибок и маппинг исключений

## Описание

Gateway Telegram транслирует исключения Telethon в коды ошибок (`reason`), которые включаются в события `SessionFailed` и `GroupJoinFailed`. Backend использует `reason` для определения дальнейших действий и формирования сообщений пользователю.

## Ошибки авторизации

Используются в событии `SessionFailed` (`AuthFailed`).

### Маппинг Telethon → reason

| Telethon Exception | reason | message | canRetry | Описание |
|-------------------|--------|---------|----------|----------|
| `PhoneCodeInvalidError` | `PHONE_CODE_INVALID` | Неверный код авторизации | `true` | Пользователь ввёл неправильный код |
| `PhoneCodeExpiredError` | `PHONE_CODE_EXPIRED` | Код авторизации истёк | `false` | Нужен новый `RequestAuthCode` |
| `PasswordHashInvalidError` | `PASSWORD_HASH_INVALID` | Неверный пароль 2FA | `true` | Пользователь ввёл неправильный 2FA |
| `PhoneNumberBannedError` | `PHONE_NUMBER_BANNED` | Номер телефона заблокирован | `false` | Терминальная ошибка |
| `PhoneNumberInvalidError` | `PHONE_NUMBER_INVALID` | Неверный номер телефона | `false` | Формат номера некорректен |
| `PhoneNumberFloodError` | `PHONE_NUMBER_FLOOD` | Слишком много попыток | `false` | Rate limit на номер |
| `FloodWaitError` | `FLOOD_WAIT` | Превышен лимит запросов, ожидайте N сек | `false` | `message` содержит время ожидания |
| `SessionRevokedError` | `SESSION_REVOKED` | Сессия отозвана | `false` | Пользователь завершил сессию с другого устройства |
| `AuthKeyDuplicatedError` | `AUTH_KEY_DUPLICATED` | Дублирование ключа авторизации | `false` | Конфликт сессий |
| `ApiIdInvalidError` | `API_ID_INVALID` | Неверный API ID приложения | `false` | Ошибка конфигурации gateway |
| (любое другое) | `UNKNOWN_ERROR` | Неизвестная ошибка авторизации | `false` | Fallback для неожиданных ошибок |

### Логика canRetry

* `true` — пользователь может повторить ввод (код, пароль), backend отправит новую команду
* `false` — повтор невозможен, нужен новый auth flow или устранение проблемы

### Pseudocode маппинга

```python
from telethon.errors import (
    PhoneCodeInvalidError, PhoneCodeExpiredError,
    PasswordHashInvalidError, PhoneNumberBannedError,
    PhoneNumberInvalidError, PhoneNumberFloodError,
    FloodWaitError, SessionRevokedError,
    AuthKeyDuplicatedError, ApiIdInvalidError,
)

AUTH_ERROR_MAP: dict[type, tuple[str, str, bool]] = {
    PhoneCodeInvalidError:   ("PHONE_CODE_INVALID",   "Неверный код авторизации",              True),
    PhoneCodeExpiredError:   ("PHONE_CODE_EXPIRED",   "Код авторизации истёк",                 False),
    PasswordHashInvalidError:("PASSWORD_HASH_INVALID","Неверный пароль 2FA",                   True),
    PhoneNumberBannedError:  ("PHONE_NUMBER_BANNED",  "Номер телефона заблокирован",            False),
    PhoneNumberInvalidError: ("PHONE_NUMBER_INVALID", "Неверный номер телефона",                False),
    PhoneNumberFloodError:   ("PHONE_NUMBER_FLOOD",   "Слишком много попыток",                  False),
    FloodWaitError:          ("FLOOD_WAIT",           "Превышен лимит запросов",                False),
    SessionRevokedError:     ("SESSION_REVOKED",      "Сессия отозвана",                        False),
    AuthKeyDuplicatedError:  ("AUTH_KEY_DUPLICATED",  "Дублирование ключа авторизации",         False),
    ApiIdInvalidError:       ("API_ID_INVALID",       "Неверный API ID приложения",             False),
}

def map_auth_error(exc: Exception) -> tuple[str, str, bool]:
    """Возвращает (reason, message, canRetry)."""
    if isinstance(exc, FloodWaitError):
        return ("FLOOD_WAIT", f"Превышен лимит запросов, ожидайте {exc.seconds} сек", False)

    for exc_type, (reason, message, can_retry) in AUTH_ERROR_MAP.items():
        if isinstance(exc, exc_type):
            return (reason, message, can_retry)

    return ("UNKNOWN_ERROR", f"Неизвестная ошибка: {type(exc).__name__}", False)
```

---

## Ошибки групп

Используются в событии `GroupJoinFailed`.

### Маппинг Telethon → reason

| Telethon Exception | reason | message | Описание |
|-------------------|--------|---------|----------|
| `ChatInvalidError` | `CHAT_INVALID` | Группа не существует или недоступна | Неверный chat ID или username |
| `InviteHashInvalidError` | `INVITE_HASH_INVALID` | Недействительная ссылка приглашения | Invite link невалиден |
| `InviteHashExpiredError` | `INVITE_HASH_EXPIRED` | Ссылка приглашения истекла | Invite link просрочен |
| `ChannelInvalidError` | `CHANNEL_INVALID` | Канал не существует | Канал удалён или недоступен |
| `ChannelPrivateError` | `CHANNEL_PRIVATE` | Канал приватный, доступ запрещён | Нет прав для вступления |
| `UserBannedInChannelError` | `USER_BANNED` | Пользователь заблокирован в канале | Бан в группе |
| `ChatWriteForbiddenError` | `CHAT_WRITE_FORBIDDEN` | Нет прав на запись в чат | Ограничения прав |
| `FloodWaitError` | `FLOOD_WAIT` | Превышен лимит запросов, ожидайте N сек | Rate limit |
| `UserAlreadyParticipantError` | — | — | **Не ошибка**: генерируем `GroupJoined` |
| (любое другое) | `UNKNOWN_ERROR` | Неизвестная ошибка | Fallback |

### Pseudocode маппинга

```python
from telethon.errors import (
    ChatInvalidError, InviteHashInvalidError, InviteHashExpiredError,
    ChannelInvalidError, ChannelPrivateError, UserBannedInChannelError,
    ChatWriteForbiddenError, FloodWaitError, UserAlreadyParticipantError,
)

GROUP_ERROR_MAP: dict[type, tuple[str, str]] = {
    ChatInvalidError:           ("CHAT_INVALID",        "Группа не существует или недоступна"),
    InviteHashInvalidError:     ("INVITE_HASH_INVALID", "Недействительная ссылка приглашения"),
    InviteHashExpiredError:     ("INVITE_HASH_EXPIRED", "Ссылка приглашения истекла"),
    ChannelInvalidError:        ("CHANNEL_INVALID",     "Канал не существует"),
    ChannelPrivateError:        ("CHANNEL_PRIVATE",     "Канал приватный, доступ запрещён"),
    UserBannedInChannelError:   ("USER_BANNED",         "Пользователь заблокирован в канале"),
    ChatWriteForbiddenError:    ("CHAT_WRITE_FORBIDDEN","Нет прав на запись в чат"),
}

def map_group_error(exc: Exception) -> tuple[str, str]:
    """Возвращает (reason, message)."""
    if isinstance(exc, FloodWaitError):
        return ("FLOOD_WAIT", f"Превышен лимит запросов, ожидайте {exc.seconds} сек")

    for exc_type, (reason, message) in GROUP_ERROR_MAP.items():
        if isinstance(exc, exc_type):
            return (reason, message)

    return ("UNKNOWN_ERROR", f"Неизвестная ошибка: {type(exc).__name__}")
```

### Особый случай: UserAlreadyParticipantError

Если пользователь уже состоит в группе, это **не ошибка**. Gateway должен:

1. Получить информацию о группе (`groupTitle`, `memberCount`)
2. Опубликовать `GroupJoined` как при успешном join

---

## Ошибки сессий

Используются при `StartSession` / `StopSession` и при обнаружении проблем с сессиями.

| Ситуация | Обработка |
|----------|-----------|
| `.session` файл не найден | Публикуем `SessionFailed` с reason `SESSION_NOT_FOUND` |
| Клиент не может подключиться | Публикуем `SessionFailed` с reason `CONNECTION_FAILED` |
| Сессия отозвана при проверке | Публикуем `SessionExpired` с reason `session_revoked` |
| Auth key дублирована | Публикуем `SessionExpired` с reason `auth_key_duplicated` |

---

## Общие ошибки инфраструктуры

| Ситуация | Обработка |
|----------|-----------|
| RabbitMQ connection lost | Логируем, пытаемся reconnect (aio-pika robust connection) |
| Telegram connection lost | Telethon автоматически reconnect-ится; при неудаче — `SessionExpired` |
| Невалидный JSON в команде | Логируем ошибку, NACK сообщение, **не** публикуем событие |
| Неизвестный тип команды | Логируем ошибку, NACK сообщение |

---

## Логирование ошибок

Все ошибки логируются через structlog с контекстом:

```python
logger.error(
    "auth_error",
    session_id=session_id,
    error_reason=reason,
    telethon_error=type(exc).__name__,
    # NEVER log: phone_number, code, password, phone_code_hash
)
```

## Связанные документы

* [Models Overview](overview.md)
* [Auth Error Handling](../auth/error-handling.md)
* [Groups Error Handling](../groups/error-handling.md)
* [SessionFailed Event](events.md#sessionfailed)
* [GroupJoinFailed Event](events.md#groupjoinfailed)
