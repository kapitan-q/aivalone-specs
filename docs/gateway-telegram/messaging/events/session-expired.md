# Событие: SessionExpired

## Описание

Ранее авторизованная сессия истекла или была отозвана. Это push-событие, инициированное Telegram, а не ответ на команду backend.

**Wire name (для backend)**: `SessionExpired`
**Routing key**: `telegram.auth.session_expired`

## Когда генерируется

- `SessionRevokedError` — пользователь завершил сессию с другого устройства
- `AuthKeyDuplicatedError` — конфликт ключей авторизации
- Невозможность reconnect после потери связи (post-MVP)

## Поля

| Поле | Тип | Описание |
|------|-----|----------|
| `event` | string | `"SessionExpired"` |
| `sessionId` | string (UUID) | ID сессии в backend |
| `messengerType` | string | `"telegram"` |
| `reason` | string | `"session_revoked"`, `"auth_key_duplicated"`, `"connection_lost"` |
| `timestamp` | string (ISO 8601) | Время генерации |

> **correlationId отсутствует** — это push-событие.

## JSON-пример

```json
{
    "event": "SessionExpired",
    "sessionId": "550e8400-e29b-41d4-a716-446655440000",
    "messengerType": "telegram",
    "reason": "session_revoked",
    "timestamp": "2024-01-15T14:00:00Z"
}
```

## Действия backend при получении

1. Обновить MonitoringSession: status → EXPIRED, isActive → false
2. Деактивировать все приватные группы, привязанные к этой сессии
3. Уведомить пользователя: «Сессия Telegram истекла, авторизуйтесь заново»

## Действия gateway при генерации

1. Снять все event handlers для групп этой сессии (через GroupRegistry)
2. Disconnect клиент
3. Удалить ActiveClient из ClientPool
4. Удалить .session файл (сессия недействительна)

## Связанные документы

* [Models — Events](../../models/events.md#sessionexpired)
* [Session Lifecycle](../../sessions/session-lifecycle.md)
* [Group Registry](../../groups/group-registry.md)
