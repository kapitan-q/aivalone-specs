# Событие: SessionFailed

## Описание

Ошибка авторизации сессии. Может быть retry-able (неверный код) или терминальная (номер забанен).

**Wire name (для backend)**: `AuthFailed`
**Routing key**: `telegram.auth.session_failed`

## Когда генерируется

При любой ошибке Telethon во время auth flow или при неудачном StartSession.

## Поля

| Поле | Тип | Описание |
|------|-----|----------|
| `event` | string | `"AuthFailed"` |
| `sessionId` | string (UUID) | ID сессии в backend |
| `messengerType` | string | `"telegram"` |
| `reason` | string | Код ошибки |
| `message` | string | Человекочитаемое описание |
| `canRetry` | boolean | Можно ли повторить операцию |
| `correlationId` | string (UUID) | ID корреляции |
| `timestamp` | string (ISO 8601) | Время генерации |

## JSON-примеры

### Retry-able ошибка

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

### Терминальная ошибка

```json
{
    "event": "AuthFailed",
    "sessionId": "550e8400-e29b-41d4-a716-446655440000",
    "messengerType": "telegram",
    "reason": "PHONE_NUMBER_BANNED",
    "message": "Номер телефона заблокирован",
    "canRetry": false,
    "correlationId": "7c9e6679-7425-40de-944b-e07fc1f90ae7",
    "timestamp": "2024-01-15T10:33:00Z"
}
```

## Действия backend при получении

### canRetry == true
1. Уведомить пользователя: «Неверный код/пароль, попробуйте снова»
2. Ожидать повторную попытку

### canRetry == false
1. Обновить MonitoringSession: status → FAILED
2. Уведомить пользователя: сообщение с причиной
3. Очистить данные авторизации

## Возможные reason коды

Полный список: [errors.md](../../models/errors.md)

## Связанные документы

* [Models — Events](../../models/events.md#sessionfailed)
* [Models — Errors](../../models/errors.md)
