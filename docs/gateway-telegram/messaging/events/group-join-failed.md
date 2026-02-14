# Событие: GroupJoinFailed

## Описание

Ошибка присоединения к группе или каналу.

**Wire name (для backend)**: `GroupJoinFailed`
**Routing key**: `telegram.group.join_failed`

## Когда генерируется

При ошибке Telethon в обработчике команды `JoinGroup`.

## Поля

| Поле | Тип | Описание |
|------|-----|----------|
| `event` | string | `"GroupJoinFailed"` |
| `groupId` | string (UUID) | ID группы в backend |
| `externalGroupId` | string | `@username` или chat ID |
| `messengerType` | string | `"telegram"` |
| `reason` | string | Код ошибки |
| `message` | string | Человекочитаемое описание |
| `correlationId` | string (UUID) | ID корреляции |
| `timestamp` | string (ISO 8601) | Время генерации |

## JSON-пример

```json
{
    "event": "GroupJoinFailed",
    "groupId": "660e8400-e29b-41d4-a716-446655440001",
    "externalGroupId": "@cryptonews",
    "messengerType": "telegram",
    "reason": "CHAT_INVALID",
    "message": "Группа не существует или недоступна",
    "correlationId": "8d9e6679-7425-40de-944b-e07fc1f90ae8",
    "timestamp": "2024-01-15T11:00:00Z"
}
```

## Действия backend при получении

1. Обновить MonitoredGroup: status → FAILED
2. Уведомить пользователя: сообщение с причиной

## Возможные reason коды

Полный список: [errors.md](../../models/errors.md)

## Связанные документы

* [Models — Events](../../models/events.md#groupjoinfailed)
* [Команда JoinGroup](../commands/join-group.md)
* [Models — Errors](../../models/errors.md)
