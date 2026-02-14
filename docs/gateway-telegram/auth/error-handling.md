# Auth — Error Handling

## Описание

Маппинг исключений Telethon в auth flow на события `SessionFailed`. Определяет `reason`, `message` и `canRetry` для каждого типа ошибки.

## Общий принцип

```
Telethon Exception → map_auth_error() → (reason, message, canRetry)
                                              │
                                              ▼
                                     SessionFailed event
                                              │
                              ┌───────────────┼───────────────┐
                              │                               │
                        canRetry=true                   canRetry=false
                              │                               │
                     AuthState сохраняется          AuthState удаляется
                     (ждём повторную попытку)        Client disconnect
                                                    .session удаляется
```

## Ошибки RequestAuthCode

| Telethon Exception | reason | canRetry | Действие |
|-------------------|--------|----------|---------|
| `PhoneNumberInvalidError` | `PHONE_NUMBER_INVALID` | false | Cleanup |
| `PhoneNumberBannedError` | `PHONE_NUMBER_BANNED` | false | Cleanup |
| `PhoneNumberFloodError` | `PHONE_NUMBER_FLOOD` | false | Cleanup |
| `FloodWaitError` | `FLOOD_WAIT` | false | Cleanup, message с временем ожидания |
| `ApiIdInvalidError` | `API_ID_INVALID` | false | Cleanup, CRITICAL log |
| Connection error | `CONNECTION_FAILED` | false | Cleanup |
| Другое | `UNKNOWN_ERROR` | false | Cleanup |

## Ошибки SubmitAuthCode

| Telethon Exception | reason | canRetry | Действие |
|-------------------|--------|----------|---------|
| `PhoneCodeInvalidError` | `PHONE_CODE_INVALID` | **true** | AuthState сохраняется |
| `PhoneCodeExpiredError` | `PHONE_CODE_EXPIRED` | false | Cleanup, нужен новый RequestAuthCode |
| `SessionPasswordNeededError` | — | — | **Не ошибка**: переход в AWAITING_2FA |
| `FloodWaitError` | `FLOOD_WAIT` | false | Cleanup |
| `SessionRevokedError` | `SESSION_REVOKED` | false | Cleanup |
| Другое | `UNKNOWN_ERROR` | false | Cleanup |

## Ошибки Submit2FA

| Telethon Exception | reason | canRetry | Действие |
|-------------------|--------|----------|---------|
| `PasswordHashInvalidError` | `PASSWORD_HASH_INVALID` | **true** | AuthState сохраняется |
| `FloodWaitError` | `FLOOD_WAIT` | false | Cleanup |
| `SessionRevokedError` | `SESSION_REVOKED` | false | Cleanup |
| Другое | `UNKNOWN_ERROR` | false | Cleanup |

## Cleanup процедура

При терминальной ошибке (`canRetry=false`):

```python
async def _cleanup_client(self, external_session_id: str) -> None:
    # 1. Удалить из пула (с disconnect)
    await self._client_pool.remove(external_session_id)
    # 2. Удалить .session файл
    self._session_manager.delete_session(external_session_id)
```

## FloodWaitError — особая обработка

`FloodWaitError` содержит количество секунд ожидания:

```python
if isinstance(exc, FloodWaitError):
    reason = "FLOOD_WAIT"
    message = f"Превышен лимит запросов, ожидайте {exc.seconds} сек"
    can_retry = False
```

Backend может использовать `message` для уведомления пользователя о времени ожидания.

## Логирование

```python
logger.error(
    "auth_error",
    session_id=session_id,
    error_reason=reason,
    can_retry=can_retry,
    telethon_error=type(exc).__name__,
    # NEVER: phone_number, code, password, phone_code_hash
)
```

## Связанные документы

* [Auth Overview](overview.md)
* [Auth Handler](auth-handler.md)
* [Models — Errors](../models/errors.md)
