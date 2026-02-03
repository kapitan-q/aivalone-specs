# Обзор контекста Bot

## Назначение

**Bot Context** отвечает за интеграцию с мессенджерами и управление диалогами с пользователями. Он обеспечивает:

* интеграцию с мессенджерами (Telegram, WhatsApp, Discord и т.п.)
* доставку сообщений пользователю через `NotificationEndpoint`
* первичную инициализацию пользователя (/start)
* управление диалогами через Conversation FSM
* маршрутизацию входящих сообщений
* хранение и разрешение NotificationEndpoint

Контекст построен по принципам **DDD** и является автономной частью `Core Backend`, что позволит в будущем при необходимости выделить его в отдельный сервис.

## Ключевые принципы

Bot Context **не**:
* регистрирует пользователей (делегирует Account Context)
* не содержит бизнес-логики аккаунта
* не принимает решений о доступах вне своих границ
* не передаёт внешние идентификаторы (chat_id) другим контекстам

## Основные функции

1. **Messenger Adapter Layer**

   * Интеграция с различными мессенджерами (Telegram, WhatsApp, Discord)
   * Webhook setup и status check
   * Преобразование входящих сообщений в BotRequest
   * Отправка BotResponse в мессенджер

2. **Conversation FSM (Finite State Machine)**

   * Управление диалогами пользователя
   * Сохранение состояния диалога (ConversationState)
   * Обработка команд и шагов диалога
   * Поддержка inline-кнопок и callback

3. **Notification Endpoint**

   * Унифицированная модель доставки сообщений
   * Инкапсуляция внешних идентификаторов (chat_id)
   * Поддержка нескольких endpoints на пользователя (future)

4. **Message Router**

   * Маршрутизация входящих сообщений
   * Определение текущего диалога и шага
   * Обработка команд (/start, /help и т.д.)

5. **Интеграция с другими контекстами**

   * **Account Context** — асинхронная связь через Event Bus
   * **Billing Context** — запрос лимитов пользователя для Conversations

## Доменные модели (high-level)

| Модель                   | Описание                                                            |
| ------------------------ | ------------------------------------------------------------------- |
| **NotificationEndpoint** | Унифицированная точка доставки сообщений (инкапсулирует chat_id)    |
| **Conversation**         | Абстрактный диалог с FSM логикой                                   |
| **ConversationState**    | Состояние активного диалога пользователя                           |
| **BotRequest**           | Унифицированный формат входящего сообщения                         |
| **BotResponse**          | Унифицированный формат исходящего ответа                           |

## Основные сценарии использования

* Обработка команды /start от нового пользователя
* Маршрутизация входящих сообщений к соответствующим диалогам
* Отправка сообщений пользователю по команде от других контекстов
* Управление состоянием диалога (переход между шагами, отмена)
* Регистрация и отзыв NotificationEndpoint
* Генерация WebApp ссылок и inline-кнопок

## Взаимодействие с другими контекстами

| Контекст | Тип взаимодействия          | Примеры                                                      |
| -------- | --------------------------- | ------------------------------------------------------------ |
| Account  | Async (Event Bus)           | `BotUserConnected`, `NotificationEndpointRegistered`         |
| Account  | Sync (Command)              | `getOrCreateUser(messenger, externalUserId)`                 |
| Billing  | Sync (Query)                | Запрос `GetUserLimits` для проверки прав в диалогах          |
| Shared   | Зависимость                 | Messenger enum, базовые исключения, события                  |

## Архитектурные ограничения

* Bot Context никогда не запрашивает Account Context синхронно (кроме getOrCreateUser)
* externalTargetId (chat_id) никогда не выходит за пределы Bot Context
* Один активный диалог на пользователя и мессенджер
* Завершённый диалог удаляет state
* Все внешние идентификаторы инкапсулированы в NotificationEndpoint
* В MVP: 1 endpoint = 1 messenger = 1 user

## Структура контекста (файловая)

```
src/Context/Bot/
├── Domain/
│   ├── Model/              # NotificationEndpoint, ConversationState, BotRequest, BotResponse
│   ├── Conversation/       # Абстрактный Conversation и конкретные диалоги
│   ├── Event/              # BotUserConnected, NotificationEndpointRegistered, etc.
│   ├── Repository/         # Интерфейсы репозиториев
│   └── Exception/          # Исключения (ConversationNotFound, InvalidCommand, etc.)
├── Application/
│   ├── Service/            # Router, ConversationRegistry
│   ├── Command/            # SendMessageToUser, RegisterEndpoint
│   ├── Query/              # GetConversationMenu
│   └── Event/              # Event Handlers
├── Infrastructure/
│   ├── Persistence/        # Doctrine Entities, Repositories
│   ├── Messenger/          # Telegram, WhatsApp, Discord Adapters
│   └── Cli/                # CLI Commands (webhook setup)
└── Presentation/
    └── Http/
        └── Controller/     # Webhook Controllers
```

## Структура документации Bot Context

- [Модели](./models/overview.md) — доменные сущности и value objects
- [Conversations](./conversations/overview.md) — диалоги и FSM логика
- [События](./events/overview.md) — события, генерируемые при изменениях
- [Исключения](./exceptions/overview.md) — ошибки и нарушения инвариантов
- [Сервисы](./services/overview.md) — Router и вспомогательные сервисы
- [Процессы](./processes/overview.md) — диаграммы взаимодействия
- [Инфраструктура](./infrastructure/overview.md) — репозитории и адаптеры
- [API](./api/endpoints.md) — Webhook endpoints

## MVP Scope

В MVP реализуется:
* NotificationEndpoint
* Telegram Adapter (Nutgram)
* /start Conversation
* WebApp button / link
* SendMessageToUser command
* Async integration с Account Context

Отложено:
* multi-endpoint
* fallback delivery
* retry policies
* advanced menu
* WhatsApp / Discord adapters

## Связанные документы

* [Account Context Overview](../backend/account/overview.md)
* [Billing Context Overview](../backend/billing/overview.md)
* [Shared Context Overview](../backend/shared/overview.md)
* [Bot Architecture](../backend/bot/bot-context-architecture.md)

## Статус реализации

* [ ] Domain Layer (NotificationEndpoint, ConversationState, BotRequest, BotResponse)
* [ ] Conversation FSM (AbstractConversation, StartConversation)
* [ ] Application Layer (Router, ConversationRegistry, Commands, Queries)
* [ ] Infrastructure Layer (Repositories, Telegram Adapter)
* [ ] Webhook Controller для Telegram
* [ ] Интеграция с EventBus для публикации событий
* [ ] Unit-тесты
* [ ] Integration-тесты
