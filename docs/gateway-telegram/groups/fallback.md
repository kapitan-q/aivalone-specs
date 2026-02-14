# Fallback при обрыве сессии

## Описание

Когда Telethon-сессия, через которую прослушивается группа (listener session), обрывается — gateway автоматически переключает прослушивание на другую доступную сессию. Это происходит без участия backend.

## Когда срабатывает

| Событие | Действие |
|---------|---------|
| **Disconnect callback** от Telethon | Автоматический fallback |
| **StopSession** (logout пользователя) | Fallback + удалить .session |
| **SessionExpired** (Telegram отозвал) | Fallback + удалить .session |

> Для сервисных аккаунтов fallback **не нужен**: SA не отзываются и имеют свой механизм reconnect.

## Алгоритм fallback

```
on_session_disconnect(disconnected_session_id):
    │
    ├─ 1. Получить все chat_ids где эта сессия — listener
    │     chats = monitoring_registry.get_chats_by_listener_session(disconnected_session_id)
    │
    ├─ 2. Для каждого chat_id:
    │     │
    │     ├─ Получить все MonitoringEntry этого chat_id
    │     │  entries = monitoring_registry.get_entries_for_chat(chat_id)
    │     │
    │     ├─ Найти fallback-сессию:
    │     │  Из всех entries выбрать requesting_session_id, который:
    │     │  a) ≠ disconnected_session_id
    │     │  b) Есть в ClientPool (активен)
    │     │  c) Имеет chat_id в user_groups кэше
    │     │
    │     ├─ НАШЛИ fallback:
    │     │  │
    │     │  ├─ Снять старый listener (уже не работает — сессия отключена)
    │     │  ├─ Создать новый handler на fallback-клиенте:
    │     │  │   handler = fallback_client.add_event_handler(on_message, NewMessage(chats=[chat_id]))
    │     │  ├─ Обновить registry:
    │     │  │   monitoring_registry.set_listener(chat_id, handler, fallback_session_id)
    │     │  └─ logger.info("fallback_activated", chat_id=chat_id,
    │     │                  from=disconnected_session_id, to=fallback_session_id)
    │     │
    │     └─ НЕ НАШЛИ fallback:
    │        │
    │        ├─ Удалить listener из registry
    │        ├─ Публикация ListenerLost event (если добавим) или просто лог
    │        └─ logger.warning("no_fallback_available", chat_id=chat_id)
    │            # Группа перестаёт мониториться до нового MonitoringStart от backend
    │
    └─ 3. Cleanup
          monitoring_registry.cleanup_session(disconnected_session_id)
          client_pool.remove(disconnected_session_id)
```

## Sequence Diagram

```
TelegramClient        DisconnectHandler       MonitoringRegistry      ClientPool
     │                        │                       │                    │
     │  disconnect            │                       │                    │
     │───────────────────────→│                       │                    │
     │                        │                       │                    │
     │                        │  get_chats_by_listener│                    │
     │                        │──────────────────────→│                    │
     │                        │  ◄ [chat_id_1, ...]   │                    │
     │                        │                       │                    │
     │                        │  # Для каждого chat_id                     │
     │                        │  get_entries_for_chat  │                    │
     │                        │──────────────────────→│                    │
     │                        │  ◄ entries             │                    │
     │                        │                       │                    │
     │                        │  # Найти fallback                          │
     │                        │                       │  get(session_id)   │
     │                        │──────────────────────────────────────────→│
     │                        │                       │  ◄ active_client   │
     │                        │                       │                    │
     │                        │  # Подписка на TG через fallback          │
  FallbackClient              │                       │                    │
     │◄───────────────────────│  add_event_handler    │                    │
     │                        │                       │                    │
     │                        │  set_listener(chat_id, handle, fallback)   │
     │                        │──────────────────────→│                    │
     │                        │                       │                    │
     │                        │  cleanup_session      │                    │
     │                        │──────────────────────→│                    │
```

## Pseudocode

```python
async def handle_session_disconnect(
    self,
    disconnected_session_id: str,
    monitoring_registry: MonitoringRegistry,
    client_pool: ClientPool,
    message_listener: MessageListener,
) -> None:
    # 1. Найти все chat_ids где эта сессия — listener
    affected_chats = monitoring_registry.get_chats_by_listener_session(
        disconnected_session_id
    )

    for chat_id in affected_chats:
        entries = monitoring_registry.get_entries_for_chat(chat_id)

        # 2. Найти fallback-сессию
        fallback_client = self._find_fallback(
            entries, disconnected_session_id, chat_id, client_pool
        )

        if fallback_client:
            # 3. Переключить listener
            handler = message_listener.register(
                fallback_client.client, chat_id
            )
            monitoring_registry.set_listener(
                chat_id, handler, fallback_client.external_session_id
            )
            logger.info(
                "fallback_activated",
                chat_id=chat_id,
                from_session=disconnected_session_id,
                to_session=fallback_client.external_session_id,
            )
        else:
            # 4. Нет fallback — группа перестаёт мониториться
            monitoring_registry.remove_listener(chat_id)
            logger.warning(
                "no_fallback_available",
                chat_id=chat_id,
                affected_groups=[e.group_id for e in entries],
            )

    # 5. Cleanup
    monitoring_registry.cleanup_session(disconnected_session_id)
    client_pool.remove(disconnected_session_id)


def _find_fallback(
    self,
    entries: list[MonitoringEntry],
    disconnected_session_id: str,
    chat_id: int,
    client_pool: ClientPool,
) -> ActiveClient | None:
    """Найти сессию, которая может стать новым listener для chat_id."""
    for entry in entries:
        session_id = entry.requesting_session_id
        if not session_id or session_id == disconnected_session_id:
            continue

        active_client = client_pool.get(session_id)
        if not active_client:
            continue

        # Проверить кэш групп: есть ли доступ к этому чату
        if active_client.user_groups and chat_id in active_client.user_groups.chat_ids:
            return active_client

    return None
```

## Обработка отзыва авторизации (StopSession)

StopSession — пользователь отзывает авторизацию:

```
1. Удалить .session файл
2. Удалить привязки групп к этой сессии (cleanup_session)
3. Если сессия была listener для каких-то групп:
   → Выполнить fallback (handle_session_disconnect)
4. Удалить client из pool
5. Отправить подтверждение backend (если нужно)
```

> Для приватных групп: fallback найдёт другую сессию.
> Для публичных: SA не зависят от пользовательских сессий — не затронуты.

## Edge Cases

### Нет fallback-сессии

Если для приватной группы нет другой сессии с доступом:
- Группа временно перестаёт мониториться
- Backend может заметить отсутствие MessageReceived
- Backend может отправить новый MonitoringStart с другим sessionId

### Fallback-сессия тоже отключается

Если fallback-сессия отключается вскоре после промоушена:
- Тот же алгоритм выполняется повторно
- Ищется следующий fallback из оставшихся entries

### Все сессии отключились

- Listener удаляется
- Entries остаются (с requesting_session_id которые offline)
- При reconnect любой из сессий — backend должен повторно отправить MonitoringStart

### Race condition: disconnect во время MonitoringStart

- MonitoringStart добавляет entry, затем пытается set_listener
- Если сессия отключилась между этими шагами → entry уже в registry
- Disconnect callback подхватит и выполнит fallback

## Связанные документы

* [Groups Overview](overview.md)
* [Join/Leave Flow](join-leave-flow.md)
* [Monitoring Registry](group-registry.md)
* [Session Lifecycle](../sessions/session-lifecycle.md)
