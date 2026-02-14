# План: Архитектура Gateway v2 — пул SA, дедупликация, fallback

## Ключевые изменения относительно текущих спецификаций

### 1. Концептуальный сдвиг
- `JoinGroup` → **MonitoringStart** (семантика: "начни мониторинг группы", а не "вступи в группу")
- `LeaveGroup` → **MonitoringStop** (семантика: "прекрати мониторинг")
- Wire names от backend остаются `JoinGroup`/`LeaveGroup`, но внутри gateway логика кардинально другая

### 2. Пул сервисных аккаунтов
- Вместо 1 SA → N сервисных аккаунтов
- Выбор SA: уже в группе → используем; нет → с наименьшим числом групп
- SA всегда активны, всегда получают сообщения от TG (даже если никто не мониторит)

### 3. Дедупликация приватных групп
- Одна группа = один Telethon listener (через одну "активную сессию")
- Другие сессии, имеющие доступ к этой группе — fallback
- При обрыве активной сессии → повторный MonitoringStart через fallback

### 4. MessageReceived: 1 событие на сообщение
- Убираем `group_id` из MessageReceived
- Отправляем `external_group_id` (chatId) — backend сам определяет подписчиков
- Проверяем: мониторится ли группа? Если нет → пропускаем сообщение

### 5. Сканирование групп при авторизации
- После успешной авторизации: `get_dialogs()` → кэш групп пользователя
- Кэш обновляется при каждом MonitoringStart
- Используется для: проверки доступа + выбора fallback сессий

---

## Новые модели данных

### ServiceAccountInfo
```
ServiceAccountInfo:
    session_name: str           # имя .session файла
    external_session_id: str    # идентификатор в пуле
    client: TelegramClient
    joined_groups: set[int]     # chat_ids групп, в которых состоит SA
```

### ActiveListener
```
ActiveListener:
    chat_id: int                     # Telegram chat ID
    external_group_id: str           # @username или chat_id строкой
    listener_session_id: str         # external_session_id сессии-слушателя
    listener_handle: EventHandler    # Telethon handler
    is_public: bool
    monitoring_entries: list[MonitoringEntry]  # кто мониторит эту группу
```

### MonitoringEntry
```
MonitoringEntry:
    group_id: str               # backend UUID (из команды JoinGroup)
    requesting_session_id: str  # sessionId из команды (кто запросил мониторинг)
    is_public: bool
    added_at: datetime
```

### UserGroupCache
```
UserGroupCache:
    session_id: str             # external_session_id
    chat_ids: set[int]          # список групп пользователя (из get_dialogs)
    last_updated: datetime
```

### Обновлённый ActiveClient
```
ActiveClient:
    external_session_id: str
    session_id: str | None      # backend session UUID
    client: TelegramClient
    is_service_account: bool
    user_groups: UserGroupCache | None  # кэш групп (для user sessions)
    connected_at: datetime
```

---

## Фазы реализации

### Фаза 1: Модели данных (models/internal.md)
Обновить внутренние модели:
- Добавить: `ServiceAccountInfo`, `ActiveListener`, `MonitoringEntry`, `UserGroupCache`
- Обновить: `ActiveClient` (добавить `user_groups`)
- Удалить: `GroupSubscription` (заменяется на `ActiveListener` + `MonitoringEntry`)

### Фаза 2: Пул сервисных аккаунтов (sessions/service-account.md → полная переработка)
**Было**: 1 SA, env `SERVICE_ACCOUNT_PHONE` + `SERVICE_ACCOUNT_SESSION`
**Стало**: N SA, конфигурация через JSON/env
- `ServiceAccountPool` класс
- API: `get_for_group(chat_id)`, `get_least_loaded()`, `get_all()`
- Инициализация: все SA подключаются при старте, FATAL если хоть один не авторизован
- Env: `SERVICE_ACCOUNTS=[{"phone": "+7...", "session": "sa_1"}, ...]`

### Фаза 3: Реестр мониторинга (groups/group-registry.md → полная переработка)
**Было**: `group_id → GroupSubscription` (1:1)
**Стало**: `chat_id → ActiveListener` с множественными `MonitoringEntry`

Новый `MonitoringRegistry`:
- `_by_chat_id: dict[int, ActiveListener]` — основной индекс
- `_by_group_id: dict[str, int]` — group_id → chat_id (для быстрого lookup)
- `_by_session_id: dict[str, set[int]]` — session_id → chat_ids (для cleanup)

API:
- `start_monitoring(chat_id, group_id, requesting_session_id, is_public) → bool` — True если создан новый listener
- `stop_monitoring(group_id) → StopResult` — результат: listener удалён или только entry
- `get_listener(chat_id) → ActiveListener | None`
- `get_by_group_id(group_id) → ActiveListener | None`
- `is_monitored(chat_id) → bool` — мониторится ли группа кем-либо
- `get_monitoring_entries(chat_id) → list[MonitoringEntry]`
- `cleanup_session(session_id)` — удалить все entries сессии + перезапуск если нужно
- `set_listener_handle(chat_id, handle, listener_session_id)` — установить/обновить listener

### Фаза 4: Join/Leave Flow (groups/join-leave-flow.md → полная переработка)

**MonitoringStart (приватная группа):**
```
1. ПРОВЕРИТЬ СЕССИЮ
   active_client = client_pool.get(sessionId)
   └── Не найдена → ошибка

2. ПРОВЕРИТЬ ПОДКЛЮЧЕНИЕ К TG
   await active_client.client.connect() / is_user_authorized()
   └── Не авторизован → ошибка

3. ПРОВЕРИТЬ ДОСТУП К ГРУППЕ
   user_groups = await refresh_user_groups(active_client)
   └── chat_id not in user_groups → ошибка NOT_A_MEMBER

4. ПРОВЕРИТЬ АКТИВНЫЙ LISTENER
   listener = monitoring_registry.get_listener(chat_id)
   ├── ЕСТЬ → группа уже слушается
   │   monitoring_registry.start_monitoring(chat_id, group_id, sessionId, is_public=False)
   │   → GroupJoined (не создаём новый listener!)
   │
   └── НЕТ → нужен новый listener
       handler = client.add_event_handler(on_message, NewMessage(chats=[chat_id]))
       monitoring_registry.start_monitoring(chat_id, group_id, sessionId, is_public=False)
       monitoring_registry.set_listener_handle(chat_id, handler, sessionId)
       → GroupJoined
```

**MonitoringStart (публичная группа):**
```
1. ПРОВЕРИТЬ ПРИВЯЗКУ К SA
   listener = monitoring_registry.get_listener(chat_id)
   ├── ЕСТЬ → SA уже в группе
   │   monitoring_registry.start_monitoring(chat_id, group_id, null, is_public=True)
   │   → GroupJoined
   │
   └── НЕТ → выбрать SA
       sa = service_account_pool.get_for_group(chat_id)
       ├── SA уже в группе TG → используем
       └── нет → sa = get_least_loaded(); JoinChannelRequest
       handler = sa.client.add_event_handler(on_message, NewMessage(chats=[chat_id]))
       monitoring_registry.start_monitoring(chat_id, group_id, null, is_public=True)
       monitoring_registry.set_listener_handle(chat_id, handler, sa.external_session_id)
       → GroupJoined
```

**MonitoringStop:**
```
1. НАЙТИ MONITORING ENTRY
   listener = monitoring_registry.get_by_group_id(group_id)
   └── Не найден → warning, idempotent

2. УДАЛИТЬ ENTRY
   result = monitoring_registry.stop_monitoring(group_id)
   ├── result.entries_remaining > 0 → ничего больше не делаем
   └── result.entries_remaining == 0 → группу больше никто не мониторит
       ├── ПРИВАТНАЯ → снять Telethon listener (отписка от событий TG)
       └── ПУБЛИЧНАЯ → ничего не делаем (SA всегда получает сообщения)
```

### Фаза 5: Fallback при обрыве сессии (groups/fallback.md — НОВЫЙ ФАЙЛ)

```
При обрыве сессии (disconnect callback):
1. Получить все chat_ids где эта сессия — listener
2. Для каждого chat_id:
   a. Получить все MonitoringEntry этого chat_id
   b. Найти другую сессию из entries, которая:
      - Всё ещё в client_pool
      - Имеет доступ к этой группе (user_groups cache)
   c. Если найдена → создать новый listener через эту сессию
   d. Если не найдена → отправить событие ListenerLost (backend решит что делать)
3. Удалить обрывшуюся сессию из всех entries
```

**При отзыве авторизации (StopSession с logout):**
```
1. Удалить .session файл
2. Удалить все MonitoringEntry этой сессии
3. Если сессия была listener для каких-то групп → выполнить fallback (шаг 5)
4. Удалить client из pool
```

### Фаза 6: Message Listener (groups/message-listener.md — обновление)

```
on_new_message(event):
    chat_id = event.chat_id

    # Проверяем: мониторится ли группа?
    if not monitoring_registry.is_monitored(chat_id):
        return  # Пропускаем — никто не мониторит

    # Отправляем ОДНО событие
    publish(MessageReceived(
        external_group_id=str(chat_id),  # или @username
        sender_name=...,
        message_text=...,
        message_id=...,
        timestamp=...,
    ))
    # Без group_id! Backend сам определит подписчиков
```

### Фаза 7: Сканирование групп при авторизации (auth/user-groups.md — НОВЫЙ ФАЙЛ)

```
После _complete_auth():
    dialogs = await client.get_dialogs()
    chat_ids = {d.id for d in dialogs if d.is_group or d.is_channel}
    active_client.user_groups = UserGroupCache(
        session_id=external_session_id,
        chat_ids=chat_ids,
        last_updated=now(),
    )
```

Обновление кэша:
- При каждом MonitoringStart для приватной группы: `refresh_user_groups()`
- Best-effort: если get_dialogs() не удался — используем старый кэш + GetParticipantRequest

### Фаза 8: Обновление контрактов событий

**MessageReceived** (models/events.md + messaging/events/message-received.md):
- Убрать: `group_id`
- Оставить: `external_group_id` (chatId), `sender_name`, `message_text`, `message_id`, `timestamp`
- Добавить: `chat_id` (числовой Telegram ID)

**Новое событие: ListenerLost** (опционально):
- Когда listener оборвался и нет fallback
- `chat_id`, `external_group_id`, `reason`

### Фаза 9: Обновление Group Handler (groups/group-handler.md)

Полная переработка pseudocode с новой логикой:
- `handle_join_group()` → MonitoringStart логика
- `handle_leave_group()` → MonitoringStop логика
- Новый: `handle_session_disconnect()` → fallback логика

### Фаза 10: Обновление обзорных файлов

- `groups/overview.md` — новая архитектура, компоненты
- `sessions/overview.md` — пул SA
- `sessions/client-pool.md` — обновить API (убрать get_service_account)
- `overview.md` — пул SA, MessageReceived без group_id
- `infrastructure/overview.md` — новые env vars (SERVICE_ACCOUNTS JSON)
- `groups/error-handling.md` — новые ошибки
- `SPECIFICATIONS_SUMMARY.md` — обновить индекс

---

## Новые файлы
1. `groups/fallback.md` — механизм fallback при обрыве сессии
2. `auth/user-groups.md` — сканирование групп пользователя

## Файлы с полной переработкой
3. `models/internal.md` — новые модели
4. `sessions/service-account.md` — пул SA
5. `groups/group-registry.md` → переименовать в monitoring-registry.md
6. `groups/join-leave-flow.md` — полностью новый flow
7. `groups/group-handler.md` — полностью новый handler
8. `groups/message-listener.md` — проверка мониторинга + один event

## Файлы с частичным обновлением
9. `models/events.md` — MessageReceived без group_id
10. `messaging/events/message-received.md` — обновить контракт
11. `auth/auth-handler.md` — добавить group scanning
12. `groups/overview.md`
13. `groups/error-handling.md`
14. `sessions/overview.md`
15. `sessions/client-pool.md`
16. `overview.md`
17. `infrastructure/overview.md`
18. `SPECIFICATIONS_SUMMARY.md`

---

## Порядок выполнения

**Batch 1** (базовые модели): Фаза 1
**Batch 2** (новые компоненты): Фазы 2, 3, 5, 7 (параллельно — независимые файлы)
**Batch 3** (основные flow): Фазы 4, 6, 9
**Batch 4** (контракты): Фаза 8
**Batch 5** (обзорные): Фаза 10
