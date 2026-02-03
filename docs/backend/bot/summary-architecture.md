# Bot — Summary Architecture

Назначение
- Интеграция с мессенджерами и управление диалогами (Conversation FSM), маршрутизация сообщений и доставка через NotificationEndpoint.

Границы ответственности
- Прием и нормализация входящих сообщений (BotRequest), управление состояниями диалогов, отправка ответов в мессенджеры.
- Инкапсуляция внешних идентификаторов (chat_id) внутри NotificationEndpoint — они не покидают контекст.
- Не регистрирует пользователей (делегирует Account) и не хранит бизнес-правила тарифов (делегирует Billing).

Ключевые компоненты
- Domain: NotificationEndpoint, Conversation, ConversationState, BotRequest, BotResponse.
- Application: Router, ConversationRegistry, команды (SendMessageToUser, RegisterEndpoint), event handlers.
- Infrastructure: messenger adapters (Telegram, WhatsApp, Discord), persistence для conversation state, webhook controllers.
- Presentation: webhook HTTP контроллеры и CLI для управления адаптерами.

Взаимодействие с другими контекстами
- Account: получение/создание пользователя (getOrCreateUser) и async события (BotUserConnected).
- Billing: синхронные запросы GetUserLimits для принятия решений в диалогах.

Ключевые сценарии
- /start от пользователя → инициация Conversation, возможно вызов Account для создания пользователя.
- Обработка входящего сообщения → Router определяет диалог и передаёт событие/команду соответствующему Conversation.
- Отправка сообщений из других контекстов через унифицированный SendMessageToUser.

Ограничения и правила
- Внешние идентификаторы не передаются наружу; один активный диалог на пользователя+мессенджер; завершённые диалоги очищают state.

Статус
- MVP: Telegram adapter, NotificationEndpoint и базовый /start conversation в приоритете; дополнительные адаптеры и тесты запланированы.