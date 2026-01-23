# Bot Context - Обзор

## Назначение

Контекст **Bot** отвечает за прием сообщений от мессенджеров, их маршрутизацию и управление состоянием диалогов с пользователями.

## Основные функции

1. **Прием и обработка сообщений**
   - Прием обновлений через Webhook
   - Парсинг и валидация запросов
   - Роутинг сообщений в соответствующие обработчики

2. **Управление диалогами (Conversations)**
   - Многошаговые диалоги с сохранением состояния
   - Автоматический переход между шагами
   - Восстановление состояния после перезагрузки

3. **Интеграция с проверкой прав**
   - Проверка прав пользователя перед запуском диалогов
   - Блокировка доступа при превышении лимитов

4. **Platform Agnostic архитектура**
   - Абстракция от конкретного мессенджера через `BotInterface`
   - Поддержка нескольких мессенджеров (Telegram, WhatsApp и т.д.)

## Доменные модели

### ConversationInterface

Интерфейс для всех диалогов.

**Методы:**
- `getCode(): string` - Код диалога (обязательный)
- `getCommand(): string` - Команда для запуска (по умолчанию '/{`getCode()`}')
- `getDescription(): string` - Описание для справки
- `requiresPermission(): bool` - Требуется ли проверка прав
- `getRequiredPermission(): ?string` - Код права
- `getAvailableSteps(): array` - Список доступных шагов
- `start(BotRequest $request): BotResponse` - Начало диалога
- `handle(BotRequest $request): BotResponse` - Обработка сообщения
- `isFinished(): bool` - Завершен ли диалог

### StateEntity

Состояние диалога пользователя.

**Поля:**
- `userId` - ID пользователя
- `conversationCode` - Код диалога
- `currentStep` - Текущий шаг (имя метода, например `stepStart`)
- `contextData` - JSON с временными данными диалога

### BotInterface

Интерфейс для работы с мессенджерами.

**Методы:**
- `sendMessage(...)` - Отправка сообщения
- `parseRequest(array $update): BotRequest` - Парсинг запроса
- `handleUpdate(...)` - Обработка обновления
- `answerCallbackQuery(...)` - Ответ на callback query
- `validateRequest(...): bool` - Валидация запроса

## Слои контекста

### Domain Layer

- `Domain/Messenger/BotInterface` - Интерфейс для работы с мессенджерами
- `Domain/Model/ConversationInterface` - Интерфейс диалогов
- `Domain/Model/AbstractConversation` - Базовый класс для диалогов
- `Domain/Model/StateEntity` - Состояние диалога
- `Domain/Repository/StateRepositoryInterface` - Интерфейс репозитория состояний
- `Domain/Webhook/WebhookManagerInterface` - Интерфейс для работы с веб-хуками

### Application Layer

- `Application/Service/BotRouter` - Роутинг сообщений
- `Application/Service/ConversationManager` - Управление диалогами
- `Application/Service/BotFactory` - Фабрика для получения адаптеров ботов
- `Application/Service/ConversationFactory` - Фабрика для получения диалогов
- `Application/Service/WebhookManagerFactory` - Фабрика для получения менеджера веб-хуков
- `Application/Service/WebhookService` - Управление вебхуками
- `Application/Conversation/*` - Реализации диалогов

### Infrastructure Layer

- `Infrastructure/Messenger/TelegramAdapter` - Реализация BotInterface для Telegram
- `Infrastructure/Messenger/TelegramWebhookAdapter` - Реализация WebhookManagerInterface для Telegram
- `Infrastructure/Persistence/StateRepository` - Реализация репозитория состояний
- `Infrastructure/Entity/StateOrmEntity` - ORM сущность для состояний

### Presentation Layer

- `Presentation/Controller/WebhookController` - Контроллер для приема вебхуков

## Интеграция с другими контекстами

### Account Context

**Порты:**
- `AccountPortInterface::getUserByTelegramId()` - Получение пользователя по Telegram ID

**Использование:**
- Проверка прав пользователя перед запуском диалогов
- Получение информации о пользователе

### Monitoring Context

**Команды:**
- `SendNotificationCommand` - Отправка уведомления пользователю

**Использование:**
- Отправка уведомлений о найденных совпадениях по фильтрам

## База данных

Таблица `bot_conversation_states`:
- `user_id` (PK) - ID пользователя из Account контекста
- `conversation_code` - Код диалога
- `current_step` - Текущий шаг (имя метода)
- `context_data` (JSON) - Временные данные диалога
- `created_at`, `updated_at` - Временные метки

## Связанные документы

- [Диалоги (Conversations)](conversations.md)
- [Вебхуки](webhooks.md)
- [Роутинг](router.md)
- [Account Context](../account/overview.md)
- [Monitoring Context](../monitoring/overview.md)
