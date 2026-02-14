# Groups — Обзор модуля

## Назначение

Модуль `groups` управляет мониторингом Telegram-групп и каналов: начало/прекращение мониторинга, прослушивание сообщений, дедупликация listener'ов, автоматический fallback при обрыве сессии.

## Зона ответственности

### Отвечает за:

* Обработку команд `JoinGroup` (MonitoringStart), `LeaveGroup` (MonitoringStop)
* Дедупликацию: один Telethon listener на один chat_id
* Выбор сервисного аккаунта для публичных групп (из пула)
* Проверку доступа к приватным группам (через кэш + GetParticipantRequest)
* Регистрацию event handlers для новых сообщений
* Публикацию `MessageReceived` (одно событие на сообщение, без group_id)
* Ведение реестра мониторинга (`MonitoringRegistry`)
* Автоматический fallback при обрыве listener-сессии
* Маппинг ошибок Telethon → события `GroupJoinFailed`

### НЕ отвечает за:

* Авторизацию пользователей (модуль `auth`)
* Управление TelegramClient (модуль `sessions`)
* Фильтрацию сообщений (backend — Filtering Context)
* Решение о подписке/отписке (backend инициирует)
* Определение подписчиков по chatId (backend маршрутизирует)

## Компоненты

| Компонент | Файл | Описание |
|-----------|------|----------|
| `GroupHandler` | `src/groups/handler.py` | Обработчики MonitoringStart/Stop |
| `MessageListener` | `src/groups/listener.py` | Telethon event handler для новых сообщений |
| `MonitoringRegistry` | `src/groups/monitoring_registry.py` | Реестр: chat_id → ActiveListener + MonitoringEntry[] |
| `FallbackHandler` | `src/groups/fallback.py` | Автоматический fallback при обрыве сессии |

## Диаграмма взаимодействия

```
Backend       GroupHandler    MonitoringRegistry   SAPool/ClientPool   Telegram
  │               │                  │                   │               │
  │ JoinGroup     │                  │                   │               │
  │──────────────→│                  │                   │               │
  │               │ get_listener()   │                   │               │
  │               │─────────────────→│                   │               │
  │               │ ◄ exists/null    │                   │               │
  │               │                  │                   │               │
  │               │ [if null: select client]              │               │
  │               │──────────────────┼──────────────────→│               │
  │               │                  │                   │  join/verify  │
  │               │                  │                   │──────────────→│
  │               │                  │                   │               │
  │               │ start_monitoring │                   │               │
  │               │─────────────────→│                   │               │
  │               │ set_listener     │                   │               │
  │               │─────────────────→│                   │               │
  │ GroupJoined   │                  │                   │               │
  │◄──────────────│                  │                   │               │
  │               │                  │                   │               │
  │               │    ┌─────────────┤                   │               │
  │               │    │  MessageListener                │  NewMessage   │
  │               │    │             │                   │◄──────────────│
  │               │    │  is_monitored?                  │               │
  │               │    │─────────────→                   │               │
  │ MessageReceived    │             │                   │               │
  │◄──────────────┼────┘             │                   │               │
```

## Внутреннее состояние

| Данные | Хранение | Описание |
|--------|----------|----------|
| Listeners | In-memory (`MonitoringRegistry`) | `Dict[chat_id, ActiveListener]` |
| Entries | In-memory (внутри ActiveListener) | `List[MonitoringEntry]` |
| Индексы | In-memory | `group_id → chat_id`, `session_id → group_ids` |

## Инварианты

1. **Один listener на chat_id** — дедупликация: несколько entries используют один Telethon handler
2. **MonitoringEntry глобально уникален по group_id** — одна группа не мониторится дважды
3. **Пустой ActiveListener удаляется** — если entries == 0, listener снимается (приватные) или удаляется из registry (публичные)
4. **При disconnect сессии** — автоматический fallback на другую сессию с доступом
5. **MessageReceived** — одно событие на сообщение, backend сам маршрутизирует
6. **SA не покидают группы при MonitoringStop** — продолжают получать сообщения, но не пересылают

## Связанные документы

* [Join/Leave Flow](join-leave-flow.md)
* [Group Handler](group-handler.md)
* [Message Listener](message-listener.md)
* [Monitoring Registry](group-registry.md)
* [Fallback](fallback.md)
* [Error Handling](error-handling.md)
* [Service Account Pool](../sessions/service-account.md)
* [User Groups](../auth/user-groups.md)
