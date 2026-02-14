# Auth State

## Описание

`AuthState` — модель промежуточного состояния auth flow. Хранится в памяти на время авторизации (от `RequestAuthCode` до `AuthSuccess` или терминальной `SessionFailed`).

## Файл

`src/auth/auth_state.py`

## Поля

| Поле | Тип | Описание |
|------|-----|----------|
| `session_id` | str (UUID) | ID сессии в backend |
| `external_session_id` | str | ID сессии в gateway (имя .session файла) |
| `phone_number` | str | Номер телефона (хранится до завершения auth) |
| `phone_code_hash` | str | Hash от `send_code_request` (нужен для `sign_in`) |
| `state` | AuthFlowState | Текущее состояние flow |
| `correlation_id` | str | ID корреляции последней команды |
| `created_at` | datetime | Время создания |

## AuthFlowState

```python
class AuthFlowState(str, Enum):
    AWAITING_CODE = "awaiting_code"
    AWAITING_2FA = "awaiting_2fa"
    COMPLETED = "completed"
```

## Хранение

```python
class AuthStateStore:
    def __init__(self, ttl_seconds: int = 900):  # 15 минут
        self._states: dict[str, AuthState] = {}
        self._ttl = ttl_seconds

    def set(self, session_id: str, state: AuthState) -> None:
        self._states[session_id] = state

    def get(self, session_id: str) -> AuthState | None:
        state = self._states.get(session_id)
        if state and self._is_expired(state):
            del self._states[session_id]
            return None
        return state

    def delete(self, session_id: str) -> None:
        self._states.pop(session_id, None)

    def _is_expired(self, state: AuthState) -> bool:
        return (datetime.now(UTC) - state.created_at).total_seconds() > self._ttl
```

## Жизненный цикл

```
RequestAuthCode → set(sessionId, AuthState(AWAITING_CODE))
                      │
                SubmitAuthCode
                      │
              ┌───────┴────────┐
              │                │
         sign_in OK      2FA Required
              │                │
         delete(id)      state = AWAITING_2FA
         (COMPLETED)           │
                          Submit2FA
                               │
                        ┌──────┴──────┐
                        │             │
                   sign_in OK    Error (canRetry)
                        │             │
                   delete(id)    state stays
                   (COMPLETED)   AWAITING_2FA
                                      │
                                 Error (terminal)
                                      │
                                 delete(id)
```

## TTL и очистка

* TTL = **15 минут** от `created_at`
* Проверяется при каждом `get()` (lazy expiration)
* Периодическая очистка (опционально, post-MVP): asyncio task каждые 5 минут
* При истечении TTL:
  1. AuthState удаляется
  2. Клиент из пула отключается
  3. `.session` файл удаляется
  4. Backend получит timeout (отсутствие события)

## Чувствительные данные

| Поле | Логируется | Описание |
|------|-----------|----------|
| `session_id` | ✅ | Безопасно — UUID |
| `external_session_id` | ✅ | Безопасно — UUID |
| `phone_number` | ❌ **НИКОГДА** | Персональные данные |
| `phone_code_hash` | ❌ **НИКОГДА** | Секретное значение Telethon |
| `state` | ✅ | Безопасно — enum |

## Связанные документы

* [Auth Overview](overview.md)
* [Auth Handler](auth-handler.md)
* [Models — Internal](../models/internal.md#authstate)
