# Обзор моделей Billing Context

## Основные доменные модели

Все модели Billing Context разделены на категории:

### Aggregates (AggregateRoots)

| Модель | Описание | Файл |
| ------ | -------- | ---- |
| **Tariff** | Управляет информацией о тарифе и его опциями | [tariff-aggregate-root.md](tariff-aggregate-root.md) |
| **UserSubscription** | Управляет подпиской пользователя на тариф | [user-subscription-aggregate-root.md](user-subscription-aggregate-root.md) |

### Value Objects

| Объект | Описание | Файл |
| ------ | -------- | ---- |
| **TariffOption** | Опция (ограничение) тарифа, часть Tariff | [tariff-option-value-object.md](tariff-option-value-object.md) |
| **TariffId** | Идентификатор тарифа | [tariff-id-value-object.md](tariff-id-value-object.md) |
| **UserSubscriptionId** | Идентификатор пользовательской подписки | [user-subscription-id-value-object.md](user-subscription-id-value-object.md) |

### Enums

| Enum | Описание | Файл |
| ---- | -------- | ---- |
| **Tariff** | Коды доступных тарифов (FREE, BASE, PRO, ENTERPRISE) | [../../shared/models/tariff.md](../../shared/models/tariff.md) |
| **TariffOptionType** | Типы опций тарифов (MAX_CONSTRAINT, BOOL, TEXT) | [../../shared/models/tariff-option-type.md](../../shared/models/tariff-option-type.md) |
| **SubscriptionStatus** | Статусы подписок (ACTIVE, EXPIRED, CANCELLED) | [../enums/subscription-status.md](../enums/subscription-status.md) |
| **SubscriptionPeriod** | Периоды подписок (MONTH, YEAR) | [../enums/subscription-period.md](../enums/subscription-period.md) |

## Структура моделей

```
docs/backend/billing/
├── models/
│   ├── overview.md                              # Этот файл (оглавление)
│   ├── tariff-aggregate-root.md                 # Спецификация Tariff
│   ├── tariff-option-value-object.md            # Спецификация TariffOption
│   ├── tariff-id-value-object.md                # Спецификация TariffId
│   ├── user-subscription-id-value-object.md     # Спецификация UserSubscriptionId
│   └── user-subscription-aggregate-root.md      # Спецификация UserSubscription
├── enums/
│   ├── subscription-status.md                   # Спецификация SubscriptionStatus
│   └── subscription-period.md                   # Спецификация SubscriptionPeriod
```

## Примеры использования

### Работа с Tariff

```php
$tariff = Tariff::create(Tariff::FREE, 'Free Plan', 0, 0.00);
$tariff->addOption('Max Groups', 'MAX_GROUPS', TariffOptionType::MAX_CONSTRAINT, 5);
$tariff->updateOption('MAX_GROUPS', 7);
$events = $tariff->pullEvents();
```

### Работа с UserSubscription

```php
// Создание подписки с периодом
$subscription = UserSubscription::create(
    UserId::fromString('123e4567-e89b-12d3-a456-426614174000'),
    TariffId::fromString('987f6543-e89b-12d3-a456-426614174111'),
    SubscriptionPeriod::MONTH  // период = месяц, validUntil вычисляется автоматически
);

// Создание бессрочной FREE подписки
$freeSubscription = UserSubscription::createFree(
    UserId::fromString('123e4567-e89b-12d3-a456-426614174000'),
    TariffId::fromString('550e8400-e29b-41d4-a716-446655440000')  // FREE tariff ID
);
// freeSubscription.validUntil = null
// freeSubscription.period = null

if ($subscription->isActive()) {
    $subscription->expire();
}

$events = $subscription->pullEvents();
```

### Продление подписки

```php
// Продление создаёт НОВУЮ подписку с ссылкой на предыдущую
$newSubscription = UserSubscription::create(
    $userId,
    $tariffId,
    SubscriptionPeriod::YEAR,
    $currentSubscription->getId()  // previousSubscriptionId
);

$newSubscription->isRenewal();  // true
$newSubscription->getPreviousSubscriptionId();  // ID старой подписки
```

## Инварианты модели

- Tariff всегда имеет TariffId и enum Tariff из Shared Context
- TariffOption коды уникальны в рамках одного Tariff
- TariffOption не может существовать отдельно от Tariff
- UserSubscription всегда имеет SubscriptionId, userId, tariffId
- Если period указан, validUntil вычисляется автоматически (now + period)
- Если period = null, validUntil = null (бессрочная подписка)
- У пользователя не может быть двух активных подписок на **один и тот же** тариф
- Пользователь может иметь активные подписки на **разные** тарифы (FREE + PRO)

## Связанные документы

* [Billing Context Overview](../overview.md)
* [События](../events/overview.md)
* [Исключения](../exceptions/overview.md)
* [Репозитории](../infrastructure/tariff-repository.md)
* [Account Context: UserId](../../account/models/overview.md)
