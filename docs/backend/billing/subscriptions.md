# Подписки

## Обзор

Система поддерживает модель **наслаиваемых подписок**. Пользователь может иметь несколько одновременно активных подписок (например, вечный `Free` + месячный `Base`).

## Доменная модель

### Subscription

**Поля:**
- `id` - Уникальный идентификатор
- `userId` - ID пользователя (UserId)
- `tariffPlanId` - ID тарифного плана
- `expiresAt` - Дата истечения подписки (может быть `null` для вечных подписок)
- `isActive` - Активна ли подписка
- `createdAt` - Дата создания
- `updatedAt` - Дата обновления

## Принципы работы

### Создание подписки

1. При покупке новой подписки создается **новая запись** (не заменяется существующая)
2. Подписка автоматически активируется (`isActive = true`)
3. Вызывается `PermissionAggregator` для пересчета прав
4. Публикуется событие `SubscriptionActivatedEvent`

### Активные подписки

Подписка считается активной, если:
- `isActive === true`
- `expiresAt === null` (вечная подписка) ИЛИ `expiresAt > now()`

### Деактивация подписки

1. Подписка помечается как неактивная (`isActive = false`)
2. Вызывается `PermissionAggregator` для пересчета прав
3. Публикуется событие `UserPermissionsUpdatedEvent`

## События

### SubscriptionActivatedEvent

Выбрасывается при покупке новой подписки.

**Payload:**
- `subscriptionId` - ID подписки
- `userId` - ID пользователя
- `tariffPlanId` - ID тарифного плана

### SubscriptionExpiredEvent

Выбрасывается при истечении срока подписки.

**Payload:**
- `subscriptionId` - ID подписки
- `userId` - ID пользователя
- `tariffPlanId` - ID тарифного плана

### SubscriptionExpiringSoonEvent

Выбрасывается за определенное количество дней до истечения подписки.

**Настройки:**
- По умолчанию: за 3 дня до истечения
- Можно настроить несколько периодов (например, 7-3-1 день)

**Особенность:** Событие отправляется **один раз** за указанный период, а не при каждом вызове метода проверки.

**Payload:**
- `subscriptionId` - ID подписки
- `userId` - ID пользователя
- `tariffPlanId` - ID тарифного плана
- `daysUntilExpiration` - Количество дней до истечения

### UserPermissionsUpdatedEvent

Выбрасывается при изменении подписки (истечение срока, добавление новой, деактивация, удаление).

**Payload:**
- `userId` - ID пользователя
- `permissions` - JSON с итоговым набором прав

## Консольная команда

### app:billing:check-expiration

Проверяет истечение подписок и отправляет уведомления.

**Логика работы:**

1. Находит все активные подписки, где `expiresAt <= now() + 7 days`
2. Для каждой подписки вызывает метод `checkExpirationTime()`:
   - Если срок истек (`expiresAt <= now()`):
     - Деактивирует подписку (`isActive = false`)
     - Выбрасывает `SubscriptionExpiredEvent`
   - Если срок истекает скоро (например, за 3 дня):
     - Проверяет, было ли уже отправлено уведомление за этот период
     - Если нет - выбрасывает `SubscriptionExpiringSoonEvent`
3. Вызывает `PermissionAggregator` для пересчета прав всех затронутых пользователей
4. Публикует `UserPermissionsUpdatedEvent` для каждого пользователя

**Запуск:**
- Рекомендуется запускать по расписанию (раз в час)
- Настройка через cron или Symfony Scheduler

**Пример:**
```bash
php bin/console app:billing:check-expiration
```

## Free тариф по умолчанию

При регистрации нового пользователя автоматически создается вечная подписка на Free тариф.

**Логика:**
- Создается в `UserService::registerUser()`
- `expiresAt = null` (вечная подписка)
- `isActive = true`

## Примеры использования

### Создание подписки

```php
$subscription = new Subscription(
    userId: $userId,
    tariffPlanId: $tariffPlanId,
    expiresAt: new DateTime('+1 month')
);

$subscriptionService->activate($subscription);
// Автоматически вызывается PermissionAggregator
// Публикуется SubscriptionActivatedEvent
```

### Проверка активных подписок

```php
$activeSubscriptions = $subscriptionRepository->findActiveByUserId($userId);
// Возвращает все подписки, где isActive = true и expiresAt > now()
```

### Деактивация подписки

```php
$subscription->deactivate();
// Автоматически вызывается PermissionAggregator
// Публикуется UserPermissionsUpdatedEvent
```

## База данных

Таблица `billing_subscriptions`:
- `id` (PK) - Уникальный идентификатор
- `user_id` (FK) - ID пользователя
- `tariff_plan_id` (FK) - ID тарифного плана
- `expires_at` - Дата истечения (nullable)
- `is_active` - Активна ли подписка
- `created_at` - Дата создания
- `updated_at` - Дата обновления

**Индексы:**
- `idx_user_id` - Для быстрого поиска подписок пользователя
- `idx_expires_at` - Для быстрого поиска истекающих подписок

## Связанные документы

- [Обзор Billing Context](overview.md)
- [Тарифы](tariffs.md)
- [Система прав](permissions.md)
- [Account Context](../account/overview.md)
