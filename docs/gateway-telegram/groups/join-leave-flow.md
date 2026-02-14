# MonitoringStart / MonitoringStop Flow

## Описание

Процесс начала и прекращения мониторинга Telegram-групп. Wire names от backend: `JoinGroup` / `LeaveGroup`, но семантика внутри gateway — **начать/прекратить мониторинг**, а не "вступить/выйти из группы".

Ключевой принцип: **один Telethon listener на один chat_id**. Если группу мониторят несколько пользователей — используем одно подключение.

---

## MonitoringStart — Приватная группа

### Предусловия

- Backend отправляет `JoinGroup` с `sessionId` ≠ null
- Указанная сессия авторизована и находится в ClientPool

### Sequence Diagram

```
Backend              GroupHandler         MonitoringRegistry     ClientPool     Telegram
  │                      │                      │                  │              │
  │ JoinGroup            │                      │                  │              │
  │ sessionId=extId      │                      │                  │              │
  │─────────────────────→│                      │                  │              │
  │                      │                      │                  │              │
  │                      │  get(sessionId)       │                  │              │
  │                      │─────────────────────────────────────────→│              │
  │                      │  ◄ active_client      │                  │              │
  │                      │                      │                  │              │
  │                      │  ensure_user_has_group(client, chat_id)  │              │
  │                      │─────────────────────────────────────────────────────────→│
  │                      │  ◄ True/False         │                  │              │
  │                      │                      │                  │              │
  │                      │  get_listener(chat_id) │                 │              │
  │                      │─────────────────────→│                  │              │
  │                      │  ◄ listener / None    │                  │              │
  │                      │                      │                  │              │
  │                      │  [Если listener == None]                 │              │
  │                      │      add_event_handler(NewMessage)       │              │
  │                      │──────────────────────────────────────────────────────────→│
  │                      │                      │                  │              │
  │                      │  start_monitoring()  │                  │              │
  │                      │─────────────────────→│                  │              │
  │                      │                      │                  │              │
  │   GroupJoined        │                      │                  │              │
  │◄─────────────────────│                      │                  │              │
```

### Шаги

```
1. ПРОВЕРИТЬ СЕССИЮ
   active_client = client_pool.get(sessionId)
   └── Не найдена → GroupJoinFailed (reason=SESSION_NOT_FOUND)

2. ПРОВЕРИТЬ ПОДКЛЮЧЕНИЕ
   if not await active_client.client.is_user_authorized():
       → GroupJoinFailed (reason=SESSION_NOT_AUTHORIZED)

3. ПРОВЕРИТЬ ДОСТУП К ГРУППЕ
   has_access = await ensure_user_has_group(active_client, chat_id)
   └── False → GroupJoinFailed (reason=NOT_A_MEMBER,
               message="Пользователь не является участником приватной группы")

4. ПРОВЕРИТЬ АКТИВНЫЙ LISTENER
   listener = monitoring_registry.get_listener(chat_id)

   ├── LISTENER СУЩЕСТВУЕТ (группа уже прослушивается):
   │   # Другая сессия уже слушает эту группу — просто добавляем запись
   │   monitoring_registry.start_monitoring(
   │       chat_id, external_group_id, group_id,
   │       requesting_session_id=sessionId, is_public=False
   │   )
   │   # НЕ создаём новый Telethon handler!
   │   → GroupJoined (используем info из существующего listener)
   │
   └── LISTENER НЕ СУЩЕСТВУЕТ (нужен новый):
       # Подписываемся на получение сообщений через TG
       needs_new = monitoring_registry.start_monitoring(
           chat_id, external_group_id, group_id,
           requesting_session_id=sessionId, is_public=False
       )  # returns True

       entity = await active_client.client.get_entity(chat_id)
       handler = active_client.client.add_event_handler(
           message_listener.create_handler(chat_id),
           events.NewMessage(chats=[chat_id])
       )
       monitoring_registry.set_listener(chat_id, handler, sessionId)

       # Получить информацию о группе
       full = await active_client.client(GetFullChannelRequest(entity))
       → GroupJoined (group_title, member_count)
```

---

## MonitoringStart — Публичная группа

### Предусловия

- Backend отправляет `JoinGroup` с `sessionId` = null
- Пул сервисных аккаунтов инициализирован

### Sequence Diagram

```
Backend          GroupHandler       MonitoringRegistry    SAPool      Telegram
  │                  │                    │                 │            │
  │ JoinGroup        │                    │                 │            │
  │ sessionId=null   │                    │                 │            │
  │─────────────────→│                    │                 │            │
  │                  │                    │                 │            │
  │                  │  get_listener(chat_id)               │            │
  │                  │───────────────────→│                 │            │
  │                  │  ◄ listener/None   │                 │            │
  │                  │                    │                 │            │
  │                  │  [Если listener == None]              │            │
  │                  │  select_sa_for_group(chat_id)        │            │
  │                  │─────────────────────────────────────→│            │
  │                  │  ◄ sa                │                │            │
  │                  │                    │                 │            │
  │                  │  [SA не в группе]   │                │            │
  │                  │  JoinChannelRequest  │               │            │
  │                  │──────────────────────────────────────────────────→│
  │                  │                    │                 │            │
  │                  │  add_event_handler  │                │            │
  │                  │──────────────────────────────────────────────────→│
  │                  │                    │                 │            │
  │                  │  start_monitoring() │                │            │
  │                  │───────────────────→│                 │            │
  │                  │                    │                 │            │
  │  GroupJoined     │                    │                 │            │
  │◄─────────────────│                    │                 │            │
```

### Шаги

```
1. ПРОВЕРИТЬ АКТИВНЫЙ LISTENER
   listener = monitoring_registry.get_listener(chat_id)

   ├── LISTENER СУЩЕСТВУЕТ (SA уже слушает эту группу):
   │   monitoring_registry.start_monitoring(
   │       chat_id, external_group_id, group_id,
   │       requesting_session_id=None, is_public=True
   │   )
   │   → GroupJoined
   │
   └── LISTENER НЕ СУЩЕСТВУЕТ:

2. ВЫБРАТЬ СЕРВИСНЫЙ АККАУНТ
   sa = service_account_pool.select_sa_for_group(chat_id)
   # Если SA уже в этой группе TG → используем
   # Если нет → выбираем с наименьшим числом групп

3. ВСТУПИТЬ В ГРУППУ (если SA ещё не участник)
   if chat_id not in sa.joined_groups:
       entity = await sa.client.get_entity(external_group_id)
       try:
           await sa.client(JoinChannelRequest(entity))
       except UserAlreadyParticipantError:
           pass  # OK
       service_account_pool.add_group(sa.external_session_id, chat_id)

4. ПОДПИСАТЬСЯ НА СООБЩЕНИЯ
   monitoring_registry.start_monitoring(
       chat_id, external_group_id, group_id,
       requesting_session_id=None, is_public=True
   )
   handler = sa.client.add_event_handler(
       message_listener.create_handler(chat_id),
       events.NewMessage(chats=[chat_id])
   )
   monitoring_registry.set_listener(chat_id, handler, sa.external_session_id)

5. ПОЛУЧИТЬ ИНФОРМАЦИЮ
   full = await sa.client(GetFullChannelRequest(entity))
   → GroupJoined (group_title, member_count)
```

---

## MonitoringStop

### Wire name

`LeaveGroup` от backend.

### Шаги

```
1. НАЙТИ MONITORING ENTRY
   listener = monitoring_registry.get_by_group_id(group_id)
   └── Не найден → logger.warning("not_monitored"), завершаем (idempotent)

2. УДАЛИТЬ ENTRY
   result = monitoring_registry.stop_monitoring(group_id)

3. ПРОВЕРИТЬ LISTENER
   ├── result.entries_remaining > 0:
   │   # Другие пользователи мониторят — ничего не делаем
   │   → Готово
   │
   └── result.entries_remaining == 0:
       ├── ПУБЛИЧНАЯ (result.is_public):
       │   # SA продолжает получать сообщения от TG, но мы не отправляем их в backend
       │   # Listener удалён из registry, но SA остаётся в группе
       │   → Готово
       │
       └── ПРИВАТНАЯ:
           # Нужно снять Telethon event handler
           active_client = client_pool.get(listener.listener_session_id)
           if active_client:
               handle = monitoring_registry.remove_listener(chat_id)
               active_client.client.remove_event_handler(handle)
           → Готово

4. НЕТ ОТВЕТНОГО СОБЫТИЯ
   # MonitoringStop — fire-and-forget
```

---

## Публичная vs Приватная — сравнение

| Аспект | Публичная | Приватная |
|--------|-----------|----------|
| `sessionId` в команде | `null` | `externalSessionId` |
| Кто слушает | Сервисный аккаунт из пула | Пользовательская сессия |
| `externalGroupId` | `@username` | Chat ID (числовой) |
| Telegram join | `JoinChannelRequest` (реальный join SA) | **Запрещён** — пользователь уже участник |
| Дедупликация | Один SA на группу | Один listener на chat_id |
| Fallback | Не нужен (SA reconnect) | Автоматический через другую сессию |
| При MonitoringStop | SA остаётся в группе | Снимается Telethon listener |
| Если не участник | SA вступает автоматически | `GroupJoinFailed (NOT_A_MEMBER)` |

---

## Cleanup при disconnect клиента

Когда пользовательская сессия отключается (StopSession, SessionExpired), срабатывает fallback:

```python
async def on_client_disconnect(session_id: str):
    # Подробности в fallback.md
    await fallback_handler.handle_session_disconnect(session_id)
```

> Подробный алгоритм fallback: [fallback.md](fallback.md)

---

## Обработка ошибок

### MonitoringStart — приватная

| Ситуация | Ошибка | Действие |
|----------|--------|---------|
| Сессия не найдена | `SESSION_NOT_FOUND` | GroupJoinFailed |
| Сессия не авторизована | `SESSION_NOT_AUTHORIZED` | GroupJoinFailed |
| Нет доступа к группе | `NOT_A_MEMBER` | GroupJoinFailed |
| Группа не существует | `CHAT_INVALID` | GroupJoinFailed |
| FloodWait | `FLOOD_WAIT` | GroupJoinFailed |

### MonitoringStart — публичная

| Ситуация | Ошибка | Действие |
|----------|--------|---------|
| Группа не существует | `CHAT_INVALID` | GroupJoinFailed |
| SA забанен | `USER_BANNED` | GroupJoinFailed |
| FloodWait | `FLOOD_WAIT` | GroupJoinFailed |

---

## Связанные документы

* [Groups Overview](overview.md)
* [Group Handler](group-handler.md)
* [Monitoring Registry](group-registry.md)
* [Fallback](fallback.md)
* [Service Account Pool](../sessions/service-account.md)
* [User Groups](../auth/user-groups.md)
* [Message Listener](message-listener.md)
* [Команда JoinGroup](../messaging/commands/join-group.md)
* [Команда LeaveGroup](../messaging/commands/leave-group.md)
