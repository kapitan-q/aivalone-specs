# Команда: LeaveGroup

## Описание

Отписка от Telegram-группы или канала, прекращение прослушивания сообщений.

**Wire name (backend)**: `LeaveGroup`
**Queue**: `gateway-telegram.listener`
**Модуль-обработчик**: groups

## Поля

| Поле | Тип | Обязательно | Описание |
|------|-----|-------------|----------|
| `command` | string | да | `"LeaveGroup"` |
| `groupId` | string (UUID) | да | ID группы в backend |
| `externalGroupId` | string | да | `@username` или chat ID |
| `messengerType` | string | да | `"telegram"` |
| `sessionId` | string \| null | нет | `externalSessionId` или `null` |
| `correlationId` | string (UUID) | да | ID корреляции |

## JSON-пример

```json
{
    "command": "LeaveGroup",
    "groupId": "660e8400-e29b-41d4-a716-446655440001",
    "externalGroupId": "@cryptonews",
    "messengerType": "telegram",
    "sessionId": null,
    "correlationId": "8d9e6679-7425-40de-944b-e07fc1f90ae8"
}
```

## Handler Flow

```
1. Найти GroupSubscription по groupId в GroupRegistry
2. Если подписка не найдена → логируем warning, завершаем
3. Снять event handler: client.remove_event_handler(subscription.listener_handle)
4. Удалить подписку из GroupRegistry
5. Определить клиент по subscription.client_key
6. Покинуть группу: await client(LeaveChannelRequest(chat_id))
7. (Нет ответного события — LeaveGroup fire-and-forget)
```

## Ответные события

Команда `LeaveGroup` не генерирует ответных событий. Backend считает группу отписанной после отправки команды.

## Обработка ошибок

| Ситуация | Действие |
|----------|---------|
| Подписка не найдена | Логируем warning, завершаем (idempotent) |
| Ошибка при leave | Логируем, всё равно удаляем подписку из registry |
| Клиент не найден | Логируем, удаляем подписку |

## Связанные документы

* [Models — Commands](../../models/commands.md#leavegroup)
* [Join/Leave Flow](../../groups/join-leave-flow.md)
* [Group Registry](../../groups/group-registry.md)
