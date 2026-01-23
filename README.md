# Спецификации проекта Aivalone

Добро пожаловать в документацию проекта Aivalone. Здесь собраны все спецификации реализованного функционала.

## Структура документации

### Core (Основные документы)

- [Обзор проекта](docs/core/overview.md) - Общее описание проекта и архитектуры
- [Архитектура DDD](docs/core/architecture.md) - Принципы Domain-Driven Design в проекте

### Backend

#### Account Context

- [Обзор Account Context](docs/backend/account/overview.md) - Управление пользователями и правами

#### Billing Context

- [Обзор Billing Context](docs/backend/billing/overview.md) - Управление тарифами и подписками
- [Тарифы](docs/backend/billing/tariffs.md) - Тарифные планы и опции
- [Подписки](docs/backend/billing/subscriptions.md) - Управление подписками пользователей
- [Система прав](docs/backend/billing/permissions.md) - Агрегация и проверка прав

#### Bot Context

- [Обзор Bot Context](docs/backend/bot/overview.md) - Telegram-бот и диалоги
- [Диалоги](docs/backend/bot/conversations.md) - Многошаговые диалоги с сохранением состояния
- [Вебхуки](docs/backend/bot/webhooks.md) - Управление вебхуками мессенджеров
- [Роутинг](docs/backend/bot/router.md) - Маршрутизация сообщений

#### Monitoring Context

- [Обзор Monitoring Context](docs/backend/monitoring/overview.md) - Мониторинг и фильтрация Telegram-контента
- [Фильтры](docs/backend/monitoring/filters.md) - Фильтры пользователей и стратегии сопоставления
- [Сессии](docs/backend/monitoring/sessions.md) - Управление сессиями Telegram
- [Карта пересечений](docs/backend/monitoring/intersection-map.md) - Shared Ingress и failover

#### Shared Context

- [Обзор Shared Context](docs/backend/shared/overview.md) - Общие компоненты
- [Система прав](docs/backend/shared/permissions.md) - PermissionsTrait и FeatureRegistry

## Быстрая навигация

### По функциональности

**Управление пользователями:**
- [Account Context](docs/backend/account/overview.md)

**Тарифы и подписки:**
- [Billing Context - Обзор](docs/backend/billing/overview.md)
- [Тарифы](docs/backend/billing/tariffs.md)
- [Подписки](docs/backend/billing/subscriptions.md)

**Система прав:**
- [Billing Context - Система прав](docs/backend/billing/permissions.md)
- [Shared Context - Система прав](docs/backend/shared/permissions.md)

**Telegram-бот:**
- [Bot Context - Обзор](docs/backend/bot/overview.md)
- [Диалоги](docs/backend/bot/conversations.md)
- [Вебхуки](docs/backend/bot/webhooks.md)
- [Роутинг](docs/backend/bot/router.md)

**Мониторинг:**
- [Monitoring Context - Обзор](docs/backend/monitoring/overview.md)
- [Фильтры](docs/backend/monitoring/filters.md)
- [Сессии](docs/backend/monitoring/sessions.md)
- [Карта пересечений](docs/backend/monitoring/intersection-map.md)

**Архитектура:**
- [Обзор проекта](docs/core/overview.md)
- [Архитектура DDD](docs/core/architecture.md)

## Связи между контекстами

### Account ↔ Billing

- **Billing → Account:** События об изменении прав (`UserPermissionsUpdatedEvent`)
- **Account ← Billing:** Обновление кэша прав в поле `permissions`

### Bot ↔ Account

- **Bot → Account:** Проверка прав пользователя через `AccountPortInterface`
- **Bot → Account:** Регистрация пользователя при первом обращении

### Bot ↔ Monitoring

- **Monitoring → Bot:** Команды на отправку уведомлений (`SendNotificationCommand`)
- **Bot → Monitoring:** Создание фильтров через диалоги

### Monitoring ↔ Billing

- **Monitoring → Billing:** Проверка прав доступа к функциям (AI, приватные группы)
- **Billing → Monitoring:** Ограничения на количество групп и фильтров

## Технологический стек

- **Backend:** PHP 8.3, Symfony 8.0
- **База данных:** PostgreSQL 16
- **Кэш/Очереди:** Redis
- **Очереди для Python-bridge:** RabbitMQ (AMQP)
- **Почтовый сервис:** MailHog (для разработки)
- **Веб-сервер:** Nginx

## Планы разработки

Все выполненные задачи задокументированы в папке [plans](docs/plans/).

## Обновление документации

При добавлении нового функционала:
1. Обновите соответствующий файл спецификации
2. Добавьте ссылки на новые документы в этот README
3. Обновите связи между контекстами
