# Команда: SubmitAuthCode

## Описание

Отправка кода подтверждения, полученного пользователем в Telegram.

**Wire name (backend)**: `SubmitCode`
**Queue**: `gateway-telegram.auth`
**Модуль-обработчик**: auth

## Поля

| Поле | Тип | Обязательно | Описание |
|------|-----|-------------|----------|
| `command` | string | да | `"SubmitCode"` |
| `sessionId` | string (UUID) | да | ID сессии в backend |
| `code` | string | да | Код подтверждения (4-8 символов) |
| `correlationId` | string (UUID) | да | ID корреляции |

## JSON-пример

```json
{
    "command": "SubmitCode",
    "sessionId": "550e8400-e29b-41d4-a716-446655440000",
    "code": "12345",
    "correlationId": "7c9e6679-7425-40de-944b-e07fc1f90ae7"
}
```

## Handler Flow

```
1. Найти AuthState по sessionId
2. Проверить state == AWAITING_CODE
3. Получить client из ClientPool по externalSessionId
4. Вызов: await client.sign_in(
       phone=auth_state.phone_number,
       code=command.code,
       phone_code_hash=auth_state.phone_code_hash
   )
5а. Успех:
    - me = await client.get_me()
    - displayName = f"{me.first_name} {me.last_name or ''}".strip()
    - Публикация AuthSuccess (externalSessionId, displayName)
    - Удаление AuthState
    - Очистка phone_number из памяти
5б. SessionPasswordNeededError:
    - password_info = await client(GetPasswordRequest())
    - hint = password_info.hint or ""
    - Обновление AuthState: state = AWAITING_2FA
    - Публикация Auth2FARequired (hint)
5в. Другая ошибка:
    - Маппинг через map_auth_error()
    - Публикация SessionFailed (reason, message, canRetry)
    - Если canRetry == false: удаление AuthState, disconnect client
```

## Ответные события

| Результат | Событие |
|-----------|---------|
| Код верный, без 2FA | `AuthSuccess` |
| Код верный, нужен 2FA | `Auth2FARequired` |
| Код неверный | `SessionFailed` (canRetry=true) |
| Код истёк | `SessionFailed` (canRetry=false) |

## Связанные документы

* [Models — Commands](../../models/commands.md#submitauthcode)
* [Auth Flow](../../auth/auth-flow.md)
* [Errors](../../models/errors.md)
