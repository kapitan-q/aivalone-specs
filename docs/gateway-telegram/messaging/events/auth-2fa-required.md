# Событие: Auth2FARequired

## Описание

Telegram запросил пароль двухфакторной аутентификации после успешного ввода кода.

**Wire name (для backend)**: `Auth2FARequired`
**Routing key**: `telegram.auth.2fa_required`

## Когда генерируется

Когда `client.sign_in(code=...)` выбрасывает `SessionPasswordNeededError` в обработчике `SubmitAuthCode`.

## Поля

| Поле | Тип | Описание |
|------|-----|----------|
| `event` | string | `"Auth2FARequired"` |
| `sessionId` | string (UUID) | ID сессии в backend |
| `messengerType` | string | `"telegram"` |
| `hint` | string | Подсказка пароля (например `"***word"`) |
| `correlationId` | string (UUID) | ID корреляции |
| `timestamp` | string (ISO 8601) | Время генерации |

## JSON-пример

```json
{
    "event": "Auth2FARequired",
    "sessionId": "550e8400-e29b-41d4-a716-446655440000",
    "messengerType": "telegram",
    "hint": "***word",
    "correlationId": "7c9e6679-7425-40de-944b-e07fc1f90ae7",
    "timestamp": "2024-01-15T10:31:00Z"
}
```

## Действия backend при получении

1. Уведомить пользователя через Bot Context: «Введите 2FA пароль» с hint
2. Ожидать Submit2FA от пользователя

## Связанные документы

* [Models — Events](../../models/events.md#auth2farequired)
* [Команда SubmitAuthCode](../commands/submit-auth-code.md)
* [Команда Submit2FA](../commands/submit-2fa.md)
