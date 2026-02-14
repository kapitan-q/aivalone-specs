# Команда: RequestAuthCode

## Описание

Запрос отправки кода авторизации на номер телефона пользователя в Telegram.

**Wire name (backend)**: `InitAuth`
**Queue**: `gateway-telegram.auth`
**Модуль-обработчик**: auth

## Поля

| Поле | Тип | Обязательно | Описание |
|------|-----|-------------|----------|
| `command` | string | да | `"InitAuth"` |
| `sessionId` | string (UUID) | да | ID сессии в backend |
| `messengerType` | string | да | `"telegram"` |
| `authData.phoneNumber` | string | да | Номер телефона `+7XXXXXXXXXX` |
| `correlationId` | string (UUID) | да | ID корреляции |

## JSON-пример

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

## Handler Flow

```
1. Валидация: phoneNumber формат, messengerType == "telegram"
2. Генерация externalSessionId (UUID4)
3. Создание TelegramClient с session файлом = externalSessionId
4. Подключение: await client.connect()
5. Отправка кода: result = await client.send_code_request(phone_number)
6. Сохранение AuthState:
   - session_id = command.sessionId
   - external_session_id = generated UUID
   - phone_code_hash = result.phone_code_hash
   - state = AWAITING_CODE
7. Добавление client в ClientPool
8. Публикация события AuthCodeSent:
   - codeLength = result.type.length (обычно 5)
   - correlationId = command.correlationId
```

## Ответные события

| Результат | Событие | Описание |
|-----------|---------|----------|
| Успех | `AuthCodeSent` | Код отправлен, codeLength |
| Ошибка | `SessionFailed` | reason, message, canRetry |

## Обработка ошибок

| Ошибка Telethon | reason | canRetry |
|-----------------|--------|----------|
| `PhoneNumberInvalidError` | `PHONE_NUMBER_INVALID` | false |
| `PhoneNumberBannedError` | `PHONE_NUMBER_BANNED` | false |
| `PhoneNumberFloodError` | `PHONE_NUMBER_FLOOD` | false |
| `FloodWaitError` | `FLOOD_WAIT` | false |
| `ApiIdInvalidError` | `API_ID_INVALID` | false |

При ошибке: удаляем AuthState, disconnectим client, удаляем .session файл.

## Связанные документы

* [Models — Commands](../../models/commands.md#requestauthcode)
* [Auth Flow](../../auth/auth-flow.md)
* [Auth Handler](../../auth/auth-handler.md)
* [Errors](../../models/errors.md)
