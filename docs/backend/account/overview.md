# Account Context - Обзор

## Назначение

Контекст **Account** отвечает за управление пользователями и их правами доступа.

## Основные функции

1. **Управление пользователями**
   - Регистрация пользователей по `telegram_id`
   - Хранение информации о пользователе
   - Проверка прав доступа

2. **Интеграция с системой прав**
   - Хранение снимка (snapshot) активных прав пользователя в поле `permissions` (JSON)
   - Использование `PermissionsTrait` для проверки прав через метод `can()`
   - Идемпотентное обновление прав при получении событий от Billing Context

## Доменные модели

### User

Доменная модель пользователя.

**Поля:**
- `id` - Уникальный идентификатор (UserId)
- `telegramId` - ID пользователя в Telegram
- `permissions` - JSON с активными правами (кэш от Billing Context)

**Методы:**
- `can(string $code, $value, FeatureRegistry $registry): bool` - Проверка прав (из `PermissionsTrait`)

## Слои контекста

### Domain Layer

- `Domain/Model/User` - Доменная модель пользователя
- `Domain/Repository/UserRepositoryInterface` - Интерфейс репозитория

### Application Layer

- `Application/Service/UserService` - Сервис для работы с пользователями
- `Application/Port/AccountPortInterface` - Интерфейс для взаимодействия с другими контекстами

### Infrastructure Layer

- `Infrastructure/Entity/UserOrmEntity` - ORM сущность для Doctrine
- `Infrastructure/Persistence/UserRepository` - Реализация репозитория

### Presentation Layer

- `Presentation/Controller/UserController` - REST API контроллеры (если требуется)

## Интеграция с другими контекстами

### Billing Context

**События:**
- `UserPermissionsUpdatedEvent` - Обновление прав пользователя

**Обработчик:**
- `UserPermissionsUpdatedHandler` - Обновляет кэш прав в поле `permissions`

### Bot Context

**Порты:**
- `AccountPortInterface::getUserByTelegramId()` - Получение пользователя по Telegram ID

## База данных

Таблица `users`:
- `id` (PK) - Уникальный идентификатор
- `telegram_id` - ID пользователя в Telegram (уникальный)
- `permissions` (JSON) - Кэш активных прав
- `created_at`, `updated_at` - Временные метки

## Связанные документы

- [Архитектура DDD](../../core/architecture.md)
- [Billing Context - Система прав](../billing/permissions.md)
- [Shared Context - PermissionsTrait](../shared/permissions.md)
