# Команда: StopSession

## Описание

Остановка сессии: logout из Telegram, удаление клиента из пула, опциональное удаление .session файла.

**Wire name (backend)**: `StopSession`
**Queue**: `gateway-telegram.listener`
**Модуль-обработчик**: sessions

## Поля

| Поле | Тип | Обязательно | Описание |
|------|-----|-------------|----------|
| `command` | string | да | `"StopSession"` |
| `sessionId` | string (UUID) | да | ID сессии в backend |
| `externalSessionId` | string | да | ID сессии в gateway |
| `messengerType` | string | да | `"telegram"` |
| `correlationId` | string (UUID) | да | ID корреляции |

## JSON-пример

```json
{
    "command": "StopSession",
    "sessionId": "550e8400-e29b-41d4-a716-446655440000",
    "externalSessionId": "a1b2c3d4-e5f6-7890-abcd-ef1234567890",
    "messengerType": "telegram",
    "correlationId": "7c9e6679-7425-40de-944b-e07fc1f90ae7"
}
```

## Handler Flow

```
1. Найти все GroupSubscription для данного client_key
2. Для каждой подписки: снять event handler (unregister listener)
3. Получить client из ClientPool по externalSessionId
4. await client.log_out()  # Завершить Telegram-сессию
5. await client.disconnect()
6. Удалить ActiveClient из ClientPool
7. Удалить .session файл
8. (Нет ответного события — StopSession fire-and-forget)
```

## Ответные события

Команда `StopSession` не генерирует ответных событий. Backend считает сессию остановленной после отправки команды.

## Обработка ошибок

| Ситуация | Действие |
|----------|---------|
| Клиент не найден в пуле | Логируем warning, пытаемся удалить .session файл |
| Ошибка при logout | Логируем, всё равно disconnect и удаляем |
| .session файл не найден | Логируем warning, продолжаем |

## Связанные документы

* [Models — Commands](../../models/commands.md#stopsession)
* [Session Lifecycle](../../sessions/session-lifecycle.md)
* [Group Registry](../../groups/group-registry.md)
