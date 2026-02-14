# Событие: AuthCodeRequested

## Описание

Код авторизации успешно отправлен пользователю в Telegram.

**Wire name (для backend)**: `AuthCodeSent`
**Routing key**: `telegram.auth.code_requested`

## Когда генерируется

После успешного вызова `client.send_code_request(phone)` в обработчике команды `RequestAuthCode`.

## Поля

| Поле | Тип | Описание |
|------|-----|----------|
| `event` | string | `"AuthCodeSent"` |
| `sessionId` | string (UUID) | ID сессии в backend |
| `messengerType` | string | `"telegram"` |
| `codeLength` | int | Длина ожидаемого кода (обычно 5) |
| `correlationId` | string (UUID) | ID корреляции из команды |
| `timestamp` | string (ISO 8601) | Время генерации |

## JSON-пример

```json
{
    "event": "AuthCodeSent",
    "sessionId": "550e8400-e29b-41d4-a716-446655440000",
    "messengerType": "telegram",
    "codeLength": 5,
    "correlationId": "7c9e6679-7425-40de-944b-e07fc1f90ae7",
    "timestamp": "2024-01-15T10:30:00Z"
}
```

## Действия backend при получении

1. Обновить MonitoringSession: status → AUTHORIZING
2. Уведомить пользователя через Bot Context: «Введите код из Telegram»
3. Ожидать SubmitAuthCode от пользователя

## Связанные документы

* [Models — Events](../../models/events.md#authcoderequested)
* [Команда RequestAuthCode](../commands/request-auth-code.md)
