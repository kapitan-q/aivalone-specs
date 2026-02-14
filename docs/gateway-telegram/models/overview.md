# Модели данных Gateway Telegram — Обзор

## Назначение

Все модели данных сервиса Gateway Telegram разделены на три категории:

1. **Команды** — входящие сообщения от backend через RabbitMQ
2. **События** — исходящие сообщения к backend через RabbitMQ
3. **Внутренние модели** — состояние, не выходящее за пределы gateway

Все модели реализуются как **Pydantic-модели** (v2) для автоматической валидации и сериализации.

## Структура

```
src/messaging/schemas.py          # Pydantic-модели команд и событий
src/auth/auth_state.py            # AuthState
src/groups/group_registry.py      # GroupSubscription
src/telegram/client_pool.py       # ActiveClient
```

## Команды (Backend → Gateway)

Команды поступают из RabbitMQ exchange `monitoring.commands`. Каждая команда содержит `correlationId` для связи с ответным событием.

| Команда | Модуль | Описание |
|---------|--------|----------|
| `RequestAuthCode` | auth | Запросить отправку кода авторизации |
| `SubmitAuthCode` | auth | Отправить код подтверждения |
| `Submit2FA` | auth | Отправить 2FA пароль |
| `StartSession` | sessions | Запустить авторизованную сессию |
| `StopSession` | sessions | Остановить сессию (logout) |
| `JoinGroup` | groups | Подписаться на группу |
| `LeaveGroup` | groups | Отписаться от группы |

**Подробнее**: [commands.md](commands.md)

## События (Gateway → Backend)

События публикуются в RabbitMQ exchange `listener.events` с routing key по шаблону `telegram.<category>.<action>`.

| Событие | Модуль | Описание |
|---------|--------|----------|
| `AuthCodeRequested` | auth | Код отправлен в Telegram |
| `Auth2FARequired` | auth | Требуется 2FA пароль |
| `SessionAuthorized` | auth | Сессия авторизована |
| `SessionFailed` | auth | Ошибка авторизации |
| `SessionExpired` | sessions | Сессия истекла / отозвана |
| `GroupJoined` | groups | Присоединились к группе |
| `GroupJoinFailed` | groups | Ошибка присоединения |
| `MessageReceived` | groups | Получено сообщение |

**Подробнее**: [events.md](events.md)

## Внутренние модели

| Модель | Модуль | Описание |
|--------|--------|----------|
| `AuthState` | auth | Состояние незавершённой авторизации (phone_code_hash, TTL) |
| `GroupSubscription` | groups | Активная подписка на группу (client, listener handle) |
| `ActiveClient` | sessions | Обёртка над TelegramClient в пуле |
| `MessageEnvelope` | messaging | Базовый формат MQ-сообщения |

**Подробнее**: [internal.md](internal.md)

## Коды ошибок

Маппинг исключений Telethon в коды ошибок gateway для трансляции в события `SessionFailed` и `GroupJoinFailed`.

**Подробнее**: [errors.md](errors.md)

## Соответствие имён Backend ↔ Gateway

Backend использует свои имена команд в MQ-сообщениях. Gateway маппит их на внутренние обработчики:

| Backend wire name | Gateway handler | Описание |
|-------------------|----------------|----------|
| `InitAuth` | `RequestAuthCode` | Начало авторизации |
| `SubmitCode` | `SubmitAuthCode` | Отправка кода |
| `Submit2FA` | `Submit2FA` | Отправка 2FA |
| `StartSession` | `StartSession` | Запуск сессии |
| `StopSession` | `StopSession` | Остановка сессии |
| `JoinGroup` | `JoinGroup` | Подписка на группу |
| `LeaveGroup` | `LeaveGroup` | Отписка от группы |

Аналогично для событий:

| Gateway event name | Backend expects | Описание |
|-------------------|----------------|----------|
| `AuthCodeRequested` | `AuthCodeSent` | Код отправлен |
| `Auth2FARequired` | `Auth2FARequired` | Требуется 2FA |
| `SessionAuthorized` | `AuthSuccess` | Успешная авторизация |
| `SessionFailed` | `AuthFailed` | Ошибка авторизации |
| `SessionExpired` | `SessionExpired` | Сессия истекла |
| `GroupJoined` | `GroupJoined` | Присоединились |
| `GroupJoinFailed` | `GroupJoinFailed` | Ошибка присоединения |
| `MessageReceived` | `MessageReceived` | Новое сообщение |

## Связанные документы

* [Gateway Telegram Overview](../overview.md)
* [Messaging Overview](../messaging/overview.md)
* [Backend Monitoring Infrastructure](../../backend/monitoring/infrastructure/overview.md)
