# Session Lifecycle

## Описание

Жизненный цикл сессии в gateway: от создания `.session` файла до удаления. Описывает процессы `StartSession`, `StopSession`, и обработку неожиданных отключений.

## Диаграмма состояний

```
                 RequestAuthCode
                      │
                      ▼
              ┌───────────────┐
              │  .session     │
              │  файл создан  │
              │  (auth flow)  │
              └───────┬───────┘
                      │
            AuthSuccess / Auth завершена
                      │
                      ▼
              ┌───────────────┐
              │  Авторизован  │◄────── StartSession
              │  (в пуле)     │        (reconnect)
              └───────┬───────┘
                      │
          ┌───────────┼────────────┐
          │           │            │
     StopSession   SessionRevoked  Disconnect
          │           │            │
          ▼           ▼            ▼
   ┌──────────┐ ┌──────────┐ ┌──────────┐
   │  Logout  │ │ Expired  │ │  Lost    │
   │ (.session│ │ (.session│ │ (reconnect│
   │  удалён) │ │  удалён) │ │  или exp)│
   └──────────┘ └──────────┘ └──────────┘
```

## Процесс: StartSession

```
1. Получение команды StartSession
   └── externalSessionId → имя .session файла

2. Проверка: клиент уже в пуле?
   └── Да → логируем info, отправляем AuthSuccess (idempotent)

3. Проверка: файл {externalSessionId}.session существует?
   └── Нет → SessionFailed (SESSION_NOT_FOUND)

4. Создание клиента:
   client = client_pool.create_client(externalSessionId)

5. Подключение:
   await client.connect()
   └── Ошибка → SessionFailed (CONNECTION_FAILED)

6. Проверка авторизации:
   authorized = await client.is_user_authorized()
   └── Нет → SessionExpired (session_revoked), удаляем .session

7. Получение информации:
   me = await client.get_me()
   displayName = f"{me.first_name} {me.last_name or ''}".strip()

8. Добавление в пул:
   client_pool.add(ActiveClient(
       external_session_id=externalSessionId,
       session_id=command.sessionId,
       client=client,
       is_service_account=False,
   ))

9. Регистрация disconnect callback

10. Публикация AuthSuccess (externalSessionId, displayName)
```

## Процесс: StopSession

```
1. Получение команды StopSession
   └── externalSessionId → какой клиент остановить

2. Снятие всех group subscriptions для этого клиента:
   subscriptions = group_registry.get_by_client_key(externalSessionId)
   for sub in subscriptions:
       client.remove_event_handler(sub.listener_handle)
       group_registry.unregister(sub.group_id)

3. Logout из Telegram:
   await client.log_out()
   └── Ошибка → логируем, продолжаем

4. Disconnect:
   await client.disconnect()

5. Удаление из пула:
   client_pool.remove(externalSessionId)

6. Удаление .session файла:
   Path(session_dir / f"{externalSessionId}.session").unlink(missing_ok=True)
```

## Процесс: Неожиданное отключение

Telethon может потерять соединение по разным причинам. Обработка:

### SessionRevokedError / AuthKeyDuplicatedError

```
1. Telethon callback: on_disconnect или error handler
2. Снять все group subscriptions
3. Удалить из пула
4. Удалить .session файл (сессия недействительна)
5. Публикация SessionExpired (reason=session_revoked | auth_key_duplicated)
```

### Временная потеря связи

```
1. Telethon автоматически пытается reconnect
2. Если reconnect успешен → ничего не делаем
3. Если reconnect не удался (post-MVP: после N попыток):
   → SessionExpired (reason=connection_lost)
```

## .session файлы

### Формат

Telethon создаёт SQLite файлы с расширением `.session`. Внутри хранятся:

* Auth key (ключ авторизации)
* DC ID (дата-центр Telegram)
* Server address
* Metadata

### Именование

```
{TELEGRAM_SESSION_DIR}/{externalSessionId}.session
```

Где `externalSessionId` — UUID4, сгенерированный при `RequestAuthCode`.

### Пример

```
/app/sessions/a1b2c3d4-e5f6-7890-abcd-ef1234567890.session
/app/sessions/service_account.session
```

### Безопасность

* Файлы хранятся на Docker volume `gateway_telegram_sessions`
* Volume не монтируется в другие контейнеры
* При `StopSession` и `SessionExpired` — файл удаляется
* Доступ только из процесса gateway

## Graceful Shutdown

При получении SIGTERM/SIGINT:

```
1. Остановить consumer (прекратить приём новых команд)
2. Дождаться завершения текущих handler-ов
3. client_pool.disconnect_all()
   - Для каждого клиента: disconnect (без logout — сессия остаётся валидной)
4. Закрыть RabbitMQ connection
```

> **Важно**: при graceful shutdown НЕ делаем `log_out()` — сессии остаются валидными для последующего `StartSession`.

## Связанные документы

* [Sessions Overview](overview.md)
* [Client Pool](client-pool.md)
* [Service Account](service-account.md)
* [Команда StartSession](../messaging/commands/start-session.md)
* [Команда StopSession](../messaging/commands/stop-session.md)
* [Событие SessionExpired](../messaging/events/session-expired.md)
