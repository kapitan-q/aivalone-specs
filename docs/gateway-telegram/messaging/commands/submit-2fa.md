# Команда: Submit2FA

## Описание

Отправка пароля двухфакторной аутентификации.

**Wire name (backend)**: `Submit2FA`
**Queue**: `gateway-telegram.auth`
**Модуль-обработчик**: auth

## Поля

| Поле | Тип | Обязательно | Описание |
|------|-----|-------------|----------|
| `command` | string | да | `"Submit2FA"` |
| `sessionId` | string (UUID) | да | ID сессии в backend |
| `password` | string | да | 2FA пароль |
| `correlationId` | string (UUID) | да | ID корреляции |

## JSON-пример

```json
{
    "command": "Submit2FA",
    "sessionId": "550e8400-e29b-41d4-a716-446655440000",
    "password": "my_secure_password",
    "correlationId": "7c9e6679-7425-40de-944b-e07fc1f90ae7"
}
```

## Handler Flow

```
1. Найти AuthState по sessionId
2. Проверить state == AWAITING_2FA
3. Получить client из ClientPool по externalSessionId
4. Вызов: await client.sign_in(password=command.password)
5а. Успех:
    - me = await client.get_me()
    - displayName = f"{me.first_name} {me.last_name or ''}".strip()
    - Публикация AuthSuccess (externalSessionId, displayName)
    - Удаление AuthState
5б. PasswordHashInvalidError:
    - Публикация SessionFailed (reason=PASSWORD_HASH_INVALID, canRetry=true)
5в. Другая ошибка:
    - Маппинг через map_auth_error()
    - Публикация SessionFailed
    - Удаление AuthState, disconnect client
```

## Ответные события

| Результат | Событие |
|-----------|---------|
| Пароль верный | `AuthSuccess` |
| Пароль неверный | `SessionFailed` (canRetry=true) |
| Другая ошибка | `SessionFailed` (canRetry=false) |

## Связанные документы

* [Models — Commands](../../models/commands.md#submit2fa)
* [Auth Flow](../../auth/auth-flow.md)
* [Errors](../../models/errors.md)
