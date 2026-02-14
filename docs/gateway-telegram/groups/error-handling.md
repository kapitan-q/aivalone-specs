# Groups — Error Handling

## Описание

Маппинг исключений Telethon при работе с группами на события `GroupJoinFailed`.

## Ошибки JoinGroup

| Telethon Exception | reason | message | Контекст |
|-------------------|--------|---------|----------|
| `ChatInvalidError` | `CHAT_INVALID` | Группа не существует или недоступна | Любая |
| `ChannelInvalidError` | `CHANNEL_INVALID` | Канал не существует | Любая |
| `ChannelPrivateError` | `CHANNEL_PRIVATE` | Канал приватный, доступ запрещён | Публичная |
| `UserNotParticipantError` | `NOT_A_MEMBER` | Пользователь не является участником приватной группы | **Приватная** |
| `UserBannedInChannelError` | `USER_BANNED` | Пользователь заблокирован в канале | Любая |
| `ChatWriteForbiddenError` | `CHAT_WRITE_FORBIDDEN` | Нет прав в чате | Любая |
| `FloodWaitError` | `FLOOD_WAIT` | Превышен лимит запросов, ожидайте N сек | Любая |
| `UserAlreadyParticipantError` | — | **Не ошибка** (только публичные): публикуем GroupJoined | Публичная |
| Клиент не найден в пуле | `SESSION_NOT_FOUND` | Сессия не найдена в gateway | Приватная |
| Сессия не авторизована | `SESSION_NOT_AUTHORIZED` | Сессия не авторизована в Telegram | Приватная |
| Другое | `UNKNOWN_ERROR` | Неизвестная ошибка | Любая |

> **Важно**: `InviteHashInvalidError` и `InviteHashExpiredError` убраны — gateway не принимает invite links напрямую. Для публичных групп используется `@username`, для приватных — числовой chat ID.

## Pseudocode

```python
def map_group_error(exc: Exception) -> tuple[str, str]:
    if isinstance(exc, FloodWaitError):
        return ("FLOOD_WAIT", f"Превышен лимит запросов, ожидайте {exc.seconds} сек")

    for exc_type, (reason, message) in GROUP_ERROR_MAP.items():
        if isinstance(exc, exc_type):
            return (reason, message)

    return ("UNKNOWN_ERROR", f"Неизвестная ошибка: {type(exc).__name__}")
```

## Особые случаи

### UserAlreadyParticipantError (только публичные)

Сервисный аккаунт уже состоит в публичной группе. Это **не ошибка**:

```python
# Только для публичных групп (is_public = True)
try:
    await client(JoinChannelRequest(entity))
except UserAlreadyParticipantError:
    pass  # Продолжаем: получаем информацию и регистрируем listener
```

### UserNotParticipantError (только приватные)

Пользователь **не является участником** приватной группы. Gateway не может программно вступить в приватную группу — пользователь должен быть участником заранее:

```python
# Только для приватных групп (is_public = False)
try:
    await client(GetParticipantRequest(entity, 'me'))
except UserNotParticipantError:
    → GroupJoinFailed(reason="NOT_A_MEMBER",
        message="Пользователь не является участником приватной группы")
```

Backend должен сообщить пользователю, что нужно самостоятельно вступить в группу перед началом мониторинга.

### FloodWaitError

Содержит `exc.seconds` — время ожидания. Включается в `message`:

```python
message = f"Превышен лимит запросов, ожидайте {exc.seconds} сек"
```

Backend может показать пользователю время ожидания.

### Ошибка resolve (@username → entity)

Если `get_entity(externalGroupId)` не может resolve username:

```python
try:
    entity = await client.get_entity(external_group_id)
except (ValueError, UsernameNotOccupiedError):
    → GroupJoinFailed (reason=CHAT_INVALID)
```

## Логирование

```python
logger.error(
    "group_join_error",
    group_id=group_id,
    external_group_id=external_group_id,
    error_reason=reason,
    telethon_error=type(exc).__name__,
)
```

## Связанные документы

* [Groups Overview](overview.md)
* [Group Handler](group-handler.md)
* [Models — Errors](../models/errors.md)
