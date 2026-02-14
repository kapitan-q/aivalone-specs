# Обзор сервиса Gateway Telegram

## Назначение

**Gateway Telegram** — Python-сервис для взаимодействия с Telegram API через протокол MTProto (библиотека Telethon). Сервис выступает шлюзом (gateway) между backend-системой AIValone и Telegram, объединяя функции **auth-service** и **listener-service** в единый компонент для мессенджера Telegram.

Gateway Telegram не содержит бизнес-логики — он является тонким адаптером между backend (Monitoring Context) и Telegram API. Все решения о том, что мониторить, как фильтровать и кого уведомлять, принимает backend.

## Роль в архитектуре

```
┌─────────────────────────────────────────────────────────────┐
│                         Backend                             │
│  ┌───────────────┐  ┌───────────────┐  ┌───────────────┐    │
│  │   Monitoring  │  │   Filtering   │  │      Bot      │    │
│  │    Context    │  │    Context    │  │    Context    │    │
│  └───────┬───────┘  └───────────────┘  └───────────────┘    │
│          │                                                  │
│          │ Commands / Events                                │
└──────────┼──────────────────────────────────────────────────┘
           │
     ┌─────┴─────────────────────┐
     │       Message Queue       │
     │       (RabbitMQ)          │
     └─────┬─────────────────────┘
           │
     ┌─────▼─────────────────────┐
     │   Gateway Telegram        │
     │   (Python + Telethon)     │
     │                           │
     │  • auth-service роль      │
     │  • listener-service роль  │
     └─────┬─────────────────────┘
           │
     ┌─────▼─────────────────────┐
     │     Telegram MTProto API  │
     └───────────────────────────┘
```

## Ключевые принципы

### Gateway Telegram отвечает за:

* Авторизацию пользователей в Telegram (MTProto через Telethon)
* Хранение и управление `.session` файлами Telethon
* Генерацию и управление `externalSessionId`
* Присоединение к Telegram-группам и выход из них
* Получение сообщений из групп и пересылку в backend
* Управление сервисным аккаунтом для публичных групп
* Обработку ошибок Telegram API и корректную трансляцию в события

### Gateway Telegram НЕ отвечает за:

* Бизнес-логику мониторинга (Backend — Monitoring Context)
* Фильтрацию сообщений (Backend — Filtering Context)
* Хранение состояния сессий в БД (Backend хранит свою модель `MonitoringSession`)
* Уведомление пользователей (Backend — Bot Context)
* Управление лимитами тарифов (Backend — Billing Context)
* Принятие решений о подписке/отписке от групп (Backend инициирует)

## Почему один сервис, а не auth + listener

Telethon использует единый `TelegramClient` на каждую сессию. Авторизация и прослушивание — операции на одном и том же клиенте. Разделение на два сервиса потребовало бы:

* Общего хранилища `.session` файлов или IPC между процессами
* Дополнительной координации жизненного цикла клиентов
* Усложнения без практической пользы

## Технологический стек

| Компонент     | Технология              | Версия | Назначение                      |
|---------------|-------------------------|--------|---------------------------------|
| Язык          | Python                  | 3.12+  | Основной язык                   |
| Telegram API  | Telethon                | 1.x    | MTProto клиент                  |
| Message Queue | aio-pika                | 9.x    | AMQP клиент для RabbitMQ        |
| Async Runtime | asyncio                 | stdlib | Асинхронное выполнение          |
| Конфигурация  | pydantic-settings       | 2.x    | Типизированные настройки из env |
| Логирование   | structlog               | 24.x   | Структурированные логи (JSON)   |
| Тестирование  | pytest + pytest-asyncio | —      | Unit и integration тесты        |

## Коммуникация с Backend

### Транспорт

Взаимодействие осуществляется через **RabbitMQ** — асинхронный обмен сообщениями. Backend отправляет команды, Gateway отвечает событиями. Все сообщения содержат `correlationId` для связи запроса с ответом.

### RabbitMQ Exchanges

| Exchange              | Тип    | Направление       | Описание                                  |
|-----------------------|--------|-------------------|-------------------------------------------|
| `monitoring.commands` | direct | Backend → Gateway | Команды авторизации и управления группами |
| `listener.events`     | topic  | Gateway → Backend | События от gateway                        |

### Входящие команды (Backend → Gateway)

#### Авторизация

| Команда           | Описание                                          |
|-------------------|---------------------------------------------------|
| `RequestAuthCode` | Запросить отправку кода авторизации (phoneNumber) |
| `SubmitAuthCode`  | Отправить код подтверждения                       |
| `Submit2FA`       | Отправить 2FA пароль                              |
| `StopSession`     | Завершить сессию (logout)                         |

#### Управление группами

| Команда        | Описание                               |
|----------------|----------------------------------------|
| `StartSession` | Запустить сессию (активировать клиент) |
| `JoinGroup`    | Подписаться на группу                  |
| `LeaveGroup`   | Отписаться от группы                   |

### Исходящие события (Gateway → Backend)

#### Авторизация

| Событие | Описание |
|---------|----------|
| `AuthCodeRequested` | Код отправлен пользователю в Telegram |
| `Auth2FARequired` | Требуется 2FA пароль (hint) |
| `SessionAuthorized` | Авторизация успешна (externalSessionId, displayName) |
| `SessionFailed` | Ошибка авторизации (reason, canRetry) |

#### Группы и сообщения

| Событие | Описание |
|---------|----------|
| `GroupJoined` | Успешно присоединились к группе (groupTitle, memberCount) |
| `GroupJoinFailed` | Ошибка присоединения к группе (reason) |
| `MessageReceived` | Получено новое сообщение из группы |

#### Сессии

| Событие | Описание |
|---------|----------|
| `SessionExpired` | Сессия истекла (reason: session_revoked, etc.) |

## Публичные vs Приватные группы

| Тип группы | sessionId в команде | Кто прослушивает | Авторизация |
|------------|---------------------|------------------|-------------|
| **Публичная** | `null` | SA из пула (выбирается автоматически) | Не требуется от пользователя |
| **Приватная** | `externalSessionId` | Одна из пользовательских сессий с доступом | Требуется авторизованная сессия |

* **Публичные группы** — gateway выбирает SA из пула (уже в группе → используем; нет → с наименьшим числом групп)
* **Приватные группы** — если группа уже мониторится через другую сессию, используем существующий listener; иначе — создаём новый через указанную сессию. При обрыве — автоматический fallback на другую сессию.

### Дедупликация мониторинга

Один Telethon listener на один chat_id. Если 10 пользователей мониторят одну группу — gateway использует одно подключение и отправляет **одно** `MessageReceived` на сообщение. Backend сам маршрутизирует по `chatId`/`externalGroupId`.

## Идентификаторы

| Идентификатор | Источник | Формат | Описание |
|--------------|---------|--------|----------|
| `sessionId` | Backend | UUID | Внутренний ID сессии в `MonitoringSession` |
| `externalSessionId` | Gateway | string | ID сессии в gateway (идентификатор `.session` файла) |
| `groupId` | Backend | UUID | Внутренний ID группы в `MonitoredGroup` |
| `externalGroupId` | Backend | string | Username (`@channel`) или chat ID в Telegram |
| `correlationId` | Backend | string | ID для связи команды с ответным событием |

## Безопасность

1. **`.session` файлы** хранятся ТОЛЬКО внутри gateway (Docker volume), недоступны извне
2. **Номера телефонов** не логируются и не сохраняются (используются только во время авторизации)
3. **Коды подтверждения и 2FA пароли** не логируются
4. **`externalSessionId`** не выходит за пределы связки gateway ↔ backend
5. **`externalGroupId`** не включается в доменные события backend (только в MQ-сообщениях)

## Scope реализации

### MVP

* Авторизация пользовательских сессий (полный flow: phone → code → 2FA → success/fail)
* `StartSession` / `StopSession` — управление жизненным циклом клиентов
* `JoinGroup` / `LeaveGroup` для публичных и приватных групп
* Получение и пересылка `MessageReceived`
* Сервисный аккаунт для публичных групп
* Docker-образ и интеграция в docker-compose
* Structured logging (JSON)

### Post-MVP

* Reconnect и graceful degradation при потере связи с Telegram
* Мониторинг здоровья сессий (heartbeat / periodic check)
* Пулинг сервисных аккаунтов для масштабирования
* Health endpoint (HTTP) для мониторинга состояния gateway
* Metrics (Prometheus)
* Graceful shutdown с корректным завершением всех клиентов

## Структура документации

```
docs/gateway-telegram/
├── overview.md                          # Этот файл
├── SPECIFICATIONS_SUMMARY.md            # Сводка спецификаций со статусами
│
├── models/                              # Модели данных
│   ├── overview.md                      # Обзор всех моделей, маппинг имён backend ↔ gateway
│   ├── commands.md                      # Pydantic-схемы входящих команд
│   ├── events.md                        # Pydantic-схемы исходящих событий
│   ├── internal.md                      # Внутренние модели (ActiveListener, MonitoringEntry, etc.)
│   └── errors.md                        # Коды ошибок, маппинг Telethon exceptions
│
├── infrastructure/                      # Инфраструктура и конфигурация
│   └── overview.md                      # Структура проекта, Docker, env, зависимости
│
├── messaging/                           # RabbitMQ контракты
│   ├── overview.md                      # Топология exchanges/queues, формат сообщений
│   ├── commands/                        # Входящие команды (от backend)
│   │   ├── request-auth-code.md
│   │   ├── submit-auth-code.md
│   │   ├── submit-2fa.md
│   │   ├── start-session.md
│   │   ├── stop-session.md
│   │   ├── join-group.md
│   │   └── leave-group.md
│   └── events/                          # Исходящие события (к backend)
│       ├── auth-code-requested.md
│       ├── auth-2fa-required.md
│       ├── session-authorized.md
│       ├── session-failed.md
│       ├── session-expired.md
│       ├── group-joined.md
│       ├── group-join-failed.md
│       └── message-received.md
│
├── auth/                                # Модуль авторизации
│   ├── overview.md                      # Обзор модуля, компоненты, инварианты
│   ├── auth-flow.md                     # Полный flow с sequence diagram и pseudocode
│   ├── auth-handler.md                  # AuthHandler — pseudocode всех методов
│   ├── auth-state.md                    # AuthState — модель, lifecycle, TTL
│   ├── session-manager.md               # SessionManager — .session файлы, externalSessionId
│   ├── user-groups.md                   # Сканирование групп пользователя при авторизации
│   └── error-handling.md                # Маппинг auth ошибок → SessionFailed
│
├── sessions/                            # Модуль сессий
│   ├── overview.md                      # Обзор модуля, компоненты, инварианты
│   ├── client-pool.md                   # ClientPool API, factory, disconnect callback
│   ├── session-lifecycle.md             # StartSession/StopSession flow, graceful shutdown
│   └── service-account.md              # Пул сервисных аккаунтов: выбор, инициализация, reconnect
│
├── groups/                              # Модуль групп
│   ├── overview.md                      # Обзор модуля, компоненты, инварианты
│   ├── join-leave-flow.md               # MonitoringStart/Stop flow с дедупликацией
│   ├── group-handler.md                 # GroupHandler pseudocode
│   ├── message-listener.md              # Telethon event handler (1 событие на сообщение)
│   ├── group-registry.md                # MonitoringRegistry — chat_id → ActiveListener
│   ├── fallback.md                      # Автоматический fallback при обрыве сессии
│   └── error-handling.md                # Маппинг group ошибок → GroupJoinFailed
│
└── security/                            # Безопасность
    └── overview.md                      # Threat model, .session хранение, чувствительные данные
```

## Связанные документы

* [Monitoring Context Overview](../backend/monitoring/overview.md) — backend-контекст, оркестрирующий мониторинг
* [Monitoring Infrastructure](../backend/monitoring/infrastructure/overview.md) — контракты MQ со стороны backend
* [Authorize Session Process](../backend/monitoring/processes/authorize-session.md) — flow авторизации со стороны backend
* [Subscribe to Group Process](../backend/monitoring/processes/subscribe-to-group.md) — flow подписки со стороны backend

## Статус реализации

* [x] Спецификация overview (этот файл)
* [x] Спецификация инфраструктуры
* [x] Спецификации моделей данных (5 файлов)
* [x] Спецификации messaging контрактов (16 файлов)
* [x] Спецификации модуля auth (7 файлов)
* [x] Спецификации модуля sessions (4 файла)
* [x] Спецификации модуля groups (7 файлов)
* [x] Спецификация безопасности
* [ ] Создание репозитория сервиса
* [ ] Реализация MVP
