# Событие: GroupJoined

## Описание

Успешно присоединились к Telegram-группе или каналу. Listener зарегистрирован.

**Wire name (для backend)**: `GroupJoined`
**Routing key**: `telegram.group.joined`

## Когда генерируется

- После успешного `client.join_chat(external_group_id)`
- Если пользователь уже участник группы (`UserAlreadyParticipantError` — не ошибка)

## Поля

| Поле | Тип | Описание |
|------|-----|----------|
| `event` | string | `"GroupJoined"` |
| `groupId` | string (UUID) | ID группы в backend |
| `externalGroupId` | string | `@username` или chat ID |
| `messengerType` | string | `"telegram"` |
| `groupTitle` | string | Название группы/канала |
| `memberCount` | int | Количество участников |
| `correlationId` | string (UUID) | ID корреляции |
| `timestamp` | string (ISO 8601) | Время генерации |

## JSON-пример

```json
{
    "event": "GroupJoined",
    "groupId": "660e8400-e29b-41d4-a716-446655440001",
    "externalGroupId": "@cryptonews",
    "messengerType": "telegram",
    "groupTitle": "Crypto News Channel",
    "memberCount": 50000,
    "correlationId": "8d9e6679-7425-40de-944b-e07fc1f90ae8",
    "timestamp": "2024-01-15T11:00:00Z"
}
```

## Действия backend при получении

1. Обновить MonitoredGroup: status → ACTIVE, groupTitle = полученное название
2. Группа готова к получению сообщений

## Связанные документы

* [Models — Events](../../models/events.md#groupjoined)
* [Команда JoinGroup](../commands/join-group.md)
