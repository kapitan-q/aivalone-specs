# Команда: StartSession

## Описание

Запуск ранее авторизованной сессии — подключение TelegramClient к Telegram серверам.

**Wire name (backend)**: `StartSession`
**Queue**: `gateway-telegram.listener`
**Модуль-обработчик**: sessions

## Поля

| Поле | Тип | Обязательно | Описание |
|------|-----|-------------|----------|
| `command` | string | да | `"StartSession"` |
| `sessionId` | string (UUID) | да | ID сессии в backend |
| `externalSessionId` | string | да | ID сессии в gateway |
| `messengerType` | string | да | `"telegram"` |
| `correlationId` | string (UUID) | да | ID корреляции |

## JSON-пример

```json
{
    "command": "StartSession",
    "sessionId": "550e8400-e29b-41d4-a716-446655440000",
    "externalSessionId": "a1b2c3d4-e5f6-7890-abcd-ef1234567890",
    "messengerType": "telegram",
    "correlationId": "7c9e6679-7425-40de-944b-e07fc1f90ae7"
}
```

## Handler Flow

```
1. Проверить: клиент уже в пуле? Если да — уже запущен, игнорируем
2. Проверить: файл {externalSessionId}.session существует?
   - Нет → SessionFailed (reason=SESSION_NOT_FOUND)
3. Создать TelegramClient с session файлом
4. await client.connect()
5. Проверить авторизацию: await client.is_user_authorized()
   - Нет → SessionExpired (reason=session_revoked), удаляем .session
6. me = await client.get_me()
7. Добавить ActiveClient в ClientPool:
   - external_session_id, session_id, client, is_service_account=false
8. Зарегистрировать disconnect callback (для отправки SessionExpired)
9. Публикация AuthSuccess (externalSessionId, displayName)
```

## Ответные события

| Результат | Событие |
|-----------|---------|
| Клиент подключён и авторизован | `AuthSuccess` |
| .session файл не найден | `SessionFailed` (reason=SESSION_NOT_FOUND) |
| Сессия отозвана | `SessionExpired` (reason=session_revoked) |
| Ошибка подключения | `SessionFailed` (reason=CONNECTION_FAILED) |

## Связанные документы

* [Models — Commands](../../models/commands.md#startsession)
* [Session Lifecycle](../../sessions/session-lifecycle.md)
* [Client Pool](../../sessions/client-pool.md)
