# Событие: SessionAuthorized

## Описание

Сессия успешно авторизована в Telegram. Backend получает `externalSessionId` для дальнейших операций.

**Wire name (для backend)**: `AuthSuccess`
**Routing key**: `telegram.auth.session_authorized`

## Когда генерируется

- После успешного `client.sign_in(code=...)` (без 2FA)
- После успешного `client.sign_in(password=...)` (с 2FA)
- После успешного `StartSession` (reconnect ранее авторизованной сессии)

## Поля

| Поле | Тип | Описание |
|------|-----|----------|
| `event` | string | `"AuthSuccess"` |
| `sessionId` | string (UUID) | ID сессии в backend |
| `messengerType` | string | `"telegram"` |
| `externalSessionId` | string | ID сессии в gateway (имя .session файла) |
| `displayName` | string | Имя пользователя в Telegram |
| `correlationId` | string (UUID) | ID корреляции |
| `timestamp` | string (ISO 8601) | Время генерации |

## JSON-пример

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

## Действия backend при получении

1. Обновить MonitoringSession:
   - status → AUTHORIZED
   - externalSessionId = полученный ID
   - displayName = полученное имя
2. Уведомить пользователя через Bot Context: «Авторизация успешна»
3. Теперь можно отправлять JoinGroup с этим externalSessionId

## Связанные документы

* [Models — Events](../../models/events.md#sessionauthorized)
* [Команда SubmitAuthCode](../commands/submit-auth-code.md)
* [Команда Submit2FA](../commands/submit-2fa.md)
* [Команда StartSession](../commands/start-session.md)
