# Команда: JoinGroup

## Описание

Начало мониторинга Telegram-группы или канала (MonitoringStart). Семантика: "начни мониторинг", а не "вступи в группу". Gateway дедуплицирует: если группа уже прослушивается — просто добавляет запись мониторинга.

**Wire name (backend)**: `JoinGroup`
**Queue**: `gateway-telegram.listener`
**Модуль-обработчик**: groups

## Поля

| Поле | Тип | Обязательно | Описание |
|------|-----|-------------|----------|
| `command` | string | да | `"JoinGroup"` |
| `groupId` | string (UUID) | да | ID группы в backend |
| `externalGroupId` | string | да | `@username` или chat ID |
| `messengerType` | string | да | `"telegram"` |
| `sessionId` | string \| null | нет | `externalSessionId` (приватная) или `null` (публичная) |
| `correlationId` | string (UUID) | да | ID корреляции |

## JSON-примеры

### Публичная группа (sessionId = null)

```json
{
    "command": "JoinGroup",
    "groupId": "660e8400-e29b-41d4-a716-446655440001",
    "externalGroupId": "@cryptonews",
    "messengerType": "telegram",
    "sessionId": null,
    "correlationId": "8d9e6679-7425-40de-944b-e07fc1f90ae8"
}
```

### Приватная группа (sessionId = externalSessionId)

```json
{
    "command": "JoinGroup",
    "groupId": "660e8400-e29b-41d4-a716-446655440001",
    "externalGroupId": "-1001234567890",
    "messengerType": "telegram",
    "sessionId": "a1b2c3d4-e5f6-7890-abcd-ef1234567890",
    "correlationId": "8d9e6679-7425-40de-944b-e07fc1f90ae8"
}
```

## Handler Flow

```
ПРИВАТНАЯ (sessionId != null):
1. Проверить сессию в ClientPool → SESSION_NOT_FOUND
2. Проверить авторизацию → SESSION_NOT_AUTHORIZED
3. Проверить доступ к группе (get_dialogs + GetParticipantRequest) → NOT_A_MEMBER
4. Проверить: уже мониторится (MonitoringRegistry)?
   - ДА → добавить MonitoringEntry → GroupJoined (без нового listener)
   - НЕТ → создать Telethon listener → GroupJoined

ПУБЛИЧНАЯ (sessionId == null):
1. Проверить: уже мониторится?
   - ДА → добавить MonitoringEntry → GroupJoined
   - НЕТ → выбрать SA (уже в группе или least-loaded)
2. JoinChannelRequest (если SA не в группе)
3. Создать Telethon listener → GroupJoined
```

> Подробный pseudocode: [Group Handler](../../groups/group-handler.md)

## Ответные события

| Результат | Событие |
|-----------|---------|
| Мониторинг начат (новый listener) | `GroupJoined` |
| Мониторинг начат (существующий listener) | `GroupJoined` |
| Не участник приватной группы | `GroupJoinFailed (NOT_A_MEMBER)` |
| Сессия не найдена | `GroupJoinFailed (SESSION_NOT_FOUND)` |
| Сессия не авторизована | `GroupJoinFailed (SESSION_NOT_AUTHORIZED)` |
| Ошибка Telegram API | `GroupJoinFailed` |

## Обработка ошибок

| Ошибка | reason | Контекст |
|--------|--------|----------|
| Сессия не найдена | `SESSION_NOT_FOUND` | Приватная |
| Сессия не авторизована | `SESSION_NOT_AUTHORIZED` | Приватная |
| Не участник группы | `NOT_A_MEMBER` | Приватная |
| `ChatInvalidError` | `CHAT_INVALID` | Любая |
| `ChannelInvalidError` | `CHANNEL_INVALID` | Любая |
| `ChannelPrivateError` | `CHANNEL_PRIVATE` | Публичная |
| `UserBannedInChannelError` | `USER_BANNED` | Любая |
| `FloodWaitError` | `FLOOD_WAIT` | Любая |

## Связанные документы

* [Models — Commands](../../models/commands.md#joingroup)
* [Join/Leave Flow](../../groups/join-leave-flow.md)
* [Message Listener](../../groups/message-listener.md)
* [Group Registry](../../groups/group-registry.md)
* [Errors](../../models/errors.md)
