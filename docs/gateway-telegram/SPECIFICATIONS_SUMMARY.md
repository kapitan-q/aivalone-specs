# Gateway Telegram — Сводка спецификаций

## Описание

Полный список спецификаций сервиса Gateway Telegram с указанием статуса.

Gateway Telegram — Python-сервис (Telethon + MTProto), выполняющий роль шлюза между backend и Telegram API:
- Авторизация пользовательских сессий
- Мониторинг групп и получение сообщений (дедупликация, fallback)
- Пул сервисных аккаунтов для публичных групп

## Структура документации

```
docs/gateway-telegram/
├── overview.md
├── SPECIFICATIONS_SUMMARY.md
│
├── models/
│   ├── overview.md
│   ├── commands.md
│   ├── events.md
│   ├── internal.md                      # ActiveListener, MonitoringEntry, ServiceAccountInfo, UserGroupCache
│   └── errors.md
│
├── infrastructure/
│   └── overview.md
│
├── messaging/
│   ├── overview.md
│   ├── commands/
│   │   ├── request-auth-code.md
│   │   ├── submit-auth-code.md
│   │   ├── submit-2fa.md
│   │   ├── start-session.md
│   │   ├── stop-session.md
│   │   ├── join-group.md
│   │   └── leave-group.md
│   └── events/
│       ├── auth-code-requested.md
│       ├── auth-2fa-required.md
│       ├── session-authorized.md
│       ├── session-failed.md
│       ├── session-expired.md
│       ├── group-joined.md
│       ├── group-join-failed.md
│       └── message-received.md
│
├── auth/
│   ├── overview.md
│   ├── auth-flow.md
│   ├── auth-handler.md
│   ├── auth-state.md
│   ├── session-manager.md
│   ├── user-groups.md                   # Сканирование групп при авторизации
│   └── error-handling.md
│
├── sessions/
│   ├── overview.md
│   ├── client-pool.md
│   ├── session-lifecycle.md
│   └── service-account.md              # Пул сервисных аккаунтов
│
├── groups/
│   ├── overview.md
│   ├── join-leave-flow.md              # MonitoringStart/Stop с дедупликацией
│   ├── group-handler.md
│   ├── message-listener.md             # 1 событие на сообщение
│   ├── group-registry.md               # MonitoringRegistry (chat_id → ActiveListener)
│   ├── fallback.md                     # Автоматический fallback при обрыве
│   └── error-handling.md
│
└── security/
    └── overview.md
```

## Статус спецификаций

### Обзор и инфраструктура

| Документ | Статус | Описание |
|----------|--------|----------|
| [overview.md](overview.md) | ✅ | Обзор сервиса, дедупликация, пул SA |
| [infrastructure/overview.md](infrastructure/overview.md) | ✅ | Docker, env, зависимости |

### Модели данных

| Документ | Статус | Описание |
|----------|--------|----------|
| [models/overview.md](models/overview.md) | ✅ | Обзор, маппинг backend ↔ gateway |
| [models/commands.md](models/commands.md) | ✅ | Pydantic-схемы 7 команд |
| [models/events.md](models/events.md) | ✅ | Pydantic-схемы 8 событий (MessageReceived без group_id) |
| [models/internal.md](models/internal.md) | ✅ | ActiveListener, MonitoringEntry, ServiceAccountInfo, UserGroupCache |
| [models/errors.md](models/errors.md) | ✅ | Маппинг Telethon exceptions → reason codes |

### Messaging — Команды

| Документ | Статус | Описание |
|----------|--------|----------|
| [messaging/commands/request-auth-code.md](messaging/commands/request-auth-code.md) | ✅ | InitAuth handler |
| [messaging/commands/submit-auth-code.md](messaging/commands/submit-auth-code.md) | ✅ | SubmitCode handler |
| [messaging/commands/submit-2fa.md](messaging/commands/submit-2fa.md) | ✅ | Submit2FA handler |
| [messaging/commands/start-session.md](messaging/commands/start-session.md) | ✅ | StartSession handler |
| [messaging/commands/stop-session.md](messaging/commands/stop-session.md) | ✅ | StopSession handler |
| [messaging/commands/join-group.md](messaging/commands/join-group.md) | ✅ | MonitoringStart (дедупликация) |
| [messaging/commands/leave-group.md](messaging/commands/leave-group.md) | ✅ | MonitoringStop |

### Messaging — События

| Документ | Статус | Описание |
|----------|--------|----------|
| [messaging/events/auth-code-requested.md](messaging/events/auth-code-requested.md) | ✅ | Код отправлен |
| [messaging/events/auth-2fa-required.md](messaging/events/auth-2fa-required.md) | ✅ | Нужен 2FA |
| [messaging/events/session-authorized.md](messaging/events/session-authorized.md) | ✅ | Авторизация успешна |
| [messaging/events/session-failed.md](messaging/events/session-failed.md) | ✅ | Ошибка авторизации |
| [messaging/events/session-expired.md](messaging/events/session-expired.md) | ✅ | Сессия истекла |
| [messaging/events/group-joined.md](messaging/events/group-joined.md) | ✅ | Мониторинг начат |
| [messaging/events/group-join-failed.md](messaging/events/group-join-failed.md) | ✅ | Ошибка мониторинга |
| [messaging/events/message-received.md](messaging/events/message-received.md) | ✅ | Сообщение (1 на сообщение, с chatId) |

### Модуль Auth

| Документ | Статус | Описание |
|----------|--------|----------|
| [auth/overview.md](auth/overview.md) | ✅ | Обзор, компоненты, инварианты |
| [auth/auth-flow.md](auth/auth-flow.md) | ✅ | Полный flow, sequence diagram |
| [auth/auth-handler.md](auth/auth-handler.md) | ✅ | AuthHandler pseudocode |
| [auth/auth-state.md](auth/auth-state.md) | ✅ | AuthState lifecycle, TTL |
| [auth/session-manager.md](auth/session-manager.md) | ✅ | .session файлы, externalSessionId |
| [auth/user-groups.md](auth/user-groups.md) | ✅ | Сканирование групп при авторизации |
| [auth/error-handling.md](auth/error-handling.md) | ✅ | Auth ошибки → SessionFailed |

### Модуль Sessions

| Документ | Статус | Описание |
|----------|--------|----------|
| [sessions/overview.md](sessions/overview.md) | ✅ | Обзор, компоненты |
| [sessions/client-pool.md](sessions/client-pool.md) | ✅ | ClientPool API |
| [sessions/session-lifecycle.md](sessions/session-lifecycle.md) | ✅ | Start/Stop flow |
| [sessions/service-account.md](sessions/service-account.md) | ✅ | Пул SA: выбор, инициализация, reconnect |

### Модуль Groups

| Документ | Статус | Описание |
|----------|--------|----------|
| [groups/overview.md](groups/overview.md) | ✅ | Обзор, дедупликация, компоненты |
| [groups/join-leave-flow.md](groups/join-leave-flow.md) | ✅ | MonitoringStart/Stop с дедупликацией |
| [groups/group-handler.md](groups/group-handler.md) | ✅ | GroupHandler pseudocode |
| [groups/message-listener.md](groups/message-listener.md) | ✅ | 1 событие на сообщение, проверка мониторинга |
| [groups/group-registry.md](groups/group-registry.md) | ✅ | MonitoringRegistry (chat_id → ActiveListener) |
| [groups/fallback.md](groups/fallback.md) | ✅ | Автоматический fallback при обрыве |
| [groups/error-handling.md](groups/error-handling.md) | ✅ | Group ошибки + NOT_A_MEMBER, SESSION_NOT_AUTHORIZED |

### Безопасность

| Документ | Статус | Описание |
|----------|--------|----------|
| [security/overview.md](security/overview.md) | ✅ | Threat model, .session хранение |

## Итого

| Категория | Написано | Всего |
|-----------|---------|-------|
| Обзор и инфраструктура | 2 | 2 |
| Модели | 5 | 5 |
| Messaging | 16 | 16 |
| Auth | 7 | 7 |
| Sessions | 4 | 4 |
| Groups | 7 | 7 |
| Security | 1 | 1 |
| **Всего** | **42** | **42** |

## Ключевые архитектурные решения (v2)

| Решение | Описание |
|---------|---------|
| **Пул SA** | Несколько сервисных аккаунтов, выбор least-loaded, гарантия no-duplicate join |
| **Дедупликация** | Один Telethon listener на chat_id, множественные MonitoringEntry |
| **MessageReceived** | Одно событие на сообщение, без group_id, backend маршрутизирует по chatId |
| **Fallback** | При обрыве listener-сессии → автоматический промоушен fallback-сессии |
| **User group scan** | get_dialogs() при авторизации + обновление при MonitoringStart |

## Связанные документы

* [Gateway Telegram Overview](overview.md)
* [Monitoring Context Specifications](../backend/monitoring/SPECIFICATIONS_SUMMARY.md)
