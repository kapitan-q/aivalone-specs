# Задача 0036: UserSubscription Aggregate Root (Domain Model)

## Описание

Создать доменную модель `UserSubscription` — AggregateRoot для управления подписками пользователей.

## Фаза

**Phase 1: Foundation**

## Спецификация

📄 [user-subscription-aggregate-root.md](../backend/billing/models/user-subscription-aggregate-root.md)

## Зависимости

- ✅ `AggregateRoot` (Shared Context) — задача 0008
- ✅ `UserId` (Account Context) — задача 0017
- ⏳ `UserSubscriptionId` Value Object — задача 0038
- ⏳ `TariffId` Value Object — задача 0037
- ⏳ `SubscriptionStatus` Enum — задача 0040
- ⏳ `SubscriptionPeriod` Enum — задача 0040

## Расположение файла

```
src/Context/Billing/Domain/Model/UserSubscription.php
```

## Требования к реализации

### Атрибуты

| Атрибут | Тип | Описание |
| ------- | --- | -------- |
| id | UserSubscriptionId | Уникальный идентификатор подписки |
| userId | UserId | Идентификатор пользователя |
| tariffId | TariffId | Идентификатор тарифа |
| status | SubscriptionStatus | Статус подписки |
| period | SubscriptionPeriod \| null | Период подписки (MONTH, YEAR) или null для бессрочной |
| validUntil | DateTimeImmutable \| null | Дата окончания или null для бессрочной |
| previousSubscriptionId | UserSubscriptionId \| null | ID предыдущей подписки (для продлений) |
| createdAt | DateTimeImmutable | Дата создания |
| updatedAt | DateTimeImmutable | Дата обновления |

### Методы

| Метод | Описание |
| ----- | -------- |
| `create(userId, tariffId, period, ?previousId): self` | Фабричный метод создания с периодом |
| `createFree(userId, tariffId): self` | Фабричный метод для FREE (бессрочная) |
| `getId(): UserSubscriptionId` | Получить идентификатор |
| `getUserId(): UserId` | Получить ID пользователя |
| `getTariffId(): TariffId` | Получить ID тарифа |
| `getStatus(): SubscriptionStatus` | Получить статус |
| `getPeriod(): ?SubscriptionPeriod` | Получить период |
| `getValidUntil(): ?DateTimeImmutable` | Получить дату окончания |
| `getPreviousSubscriptionId(): ?UserSubscriptionId` | Получить ID предыдущей подписки |
| `isActive(): bool` | Проверить активна ли подписка |
| `isExpired(): bool` | Проверить истекла ли подписка |
| `isPermanent(): bool` | Проверить бессрочная ли (validUntil = null) |
| `isRenewal(): bool` | Проверить является ли продлением |
| `expire(): void` | Пометить как истекшую |
| `cancel(): void` | Отменить подписку |

### Инварианты

- При создании с period автоматически вычисляется validUntil (now + period)
- Для FREE подписок: period = null, validUntil = null
- Статус по умолчанию ACTIVE
- При изменениях генерируется событие `UserSubscriptionUpdated`

## Пример использования

```php
// Создание подписки с периодом
$subscription = UserSubscription::create(
    $userId,
    $tariffId,
    SubscriptionPeriod::MONTH
);
// validUntil = now + 1 month

// Создание FREE подписки (бессрочная)
$freeSub = UserSubscription::createFree($userId, $freeTariffId);
// validUntil = null, period = null, isPermanent() = true

// Продление
$renewal = UserSubscription::create(
    $userId,
    $tariffId,
    SubscriptionPeriod::YEAR,
    $subscription->getId() // previousSubscriptionId
);
$renewal->isRenewal(); // true

// Проверка и истечение
if ($subscription->isActive() && $subscription->getValidUntil() < new \DateTimeImmutable()) {
    $subscription->expire();
}

$events = $subscription->pullEvents();
```

## Критерии готовности

- [x] Создан класс `UserSubscription` наследующий `AggregateRoot`
- [x] Реализованы фабричные методы `create()` и `createFree()`
- [x] Автоматический расчёт `validUntil` на основе `period`
- [x] Реализована поддержка `previousSubscriptionId` для продлений
- [x] Генерируется событие `UserSubscriptionUpdated` при изменениях
- [x] Написаны unit-тесты
- [x] Код соответствует PSR-12

## Статус: ✅ ВЫПОЛНЕНО

## Связанные задачи

- 0037: TariffId Value Object
- 0038: UserSubscriptionId Value Object
- 0040: SubscriptionStatus & SubscriptionPeriod Enums
- 0042: Domain Events
