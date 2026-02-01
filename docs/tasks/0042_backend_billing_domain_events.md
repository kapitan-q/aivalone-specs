# Задача 0042: Billing Domain Events

## Описание

Создать все доменные события для Billing Context.

## Фаза

**Phase 3: Domain Events**

## Спецификация

📄 [events/overview.md](../backend/billing/events/overview.md)

## Зависимости

- ✅ `DomainEventInterface` (Shared Context) — задача 0007
- ⏳ `TariffId` Value Object — задача 0037
- ⏳ `UserSubscriptionId` Value Object — задача 0038

## Расположение файлов

```
src/Context/Billing/Domain/Event/
├── TariffUpdated.php
├── UserSubscriptionUpdated.php
├── SubscriptionRenewed.php
├── SubscriptionExpired.php
└── SubscriptionExpiringSoon.php
```

---

## События

### 1. TariffUpdated

**Когда генерируется**: При изменении параметров тарифа или его опций

📄 [tariff-updated-event.md](../backend/billing/events/tariff-updated-event.md)

```php
class TariffUpdated implements DomainEventInterface
{
    public function __construct(
        private TariffId $tariffId,
        private Tariff $code,
        private string $changeType, // 'option_added', 'option_updated', 'option_removed', 'tariff_updated'
        private array $payload,
        private \DateTimeImmutable $occurredAt
    );
}
```

---

### 2. UserSubscriptionUpdated

**Когда генерируется**: При добавлении или удалении подписки

📄 [user-subscription-updated-event.md](../backend/billing/events/user-subscription-updated-event.md)

```php
class UserSubscriptionUpdated implements DomainEventInterface
{
    public function __construct(
        private UserSubscriptionId $subscriptionId,
        private UserId $userId,
        private TariffId $tariffId,
        private string $action, // 'ADDED', 'REMOVED', 'CANCELLED'
        private ?SubscriptionPeriod $period,
        private ?\DateTimeImmutable $validUntil,
        private \DateTimeImmutable $occurredAt
    );
}
```

---

### 3. SubscriptionRenewed

**Когда генерируется**: При продлении подписки

📄 [subscription-renewed-event.md](../backend/billing/events/subscription-renewed-event.md)

```php
class SubscriptionRenewed implements DomainEventInterface
{
    public function __construct(
        private UserSubscriptionId $subscriptionId,
        private UserSubscriptionId $previousSubscriptionId,
        private UserId $userId,
        private TariffId $tariffId,
        private SubscriptionPeriod $period,
        private \DateTimeImmutable $validUntil,
        private \DateTimeImmutable $occurredAt
    );
}
```

---

### 4. SubscriptionExpired

**Когда генерируется**: Когда подписка истекла (обнаружено CLI командой)

📄 [subscription-expired-event.md](../backend/billing/events/subscription-expired-event.md)

```php
class SubscriptionExpired implements DomainEventInterface
{
    public function __construct(
        private UserSubscriptionId $subscriptionId,
        private UserId $userId,
        private TariffId $tariffId,
        private \DateTimeImmutable $expiredAt,
        private \DateTimeImmutable $occurredAt
    );
}
```

---

### 5. SubscriptionExpiringSoon

**Когда генерируется**: За 7, 3 или 1 день до истечения подписки

📄 [subscription-expiring-soon-event.md](../backend/billing/events/subscription-expiring-soon-event.md)

```php
class SubscriptionExpiringSoon implements DomainEventInterface
{
    public function __construct(
        private UserSubscriptionId $subscriptionId,
        private UserId $userId,
        private TariffId $tariffId,
        private int $daysRemaining, // 7, 3, или 1
        private \DateTimeImmutable $expiresAt,
        private \DateTimeImmutable $occurredAt
    );
}
```

---

## Общие требования

Каждое событие должно реализовать `DomainEventInterface`:

```php
interface DomainEventInterface
{
    public function getEventId(): string;
    public function getAggregateId(): string;
    public function getAggregateType(): string;
    public function getEventName(): string;
    public function getOccurredAt(): \DateTimeImmutable;
    public function getPayload(): array;
}
```

---

## Критерии готовности

- [x] Созданы все 5 классов событий
- [x] Каждый класс реализует `DomainEventInterface`
- [x] Реализованы все методы интерфейса
- [x] События immutable
- [ ] Написаны unit-тесты
- [x] Код соответствует PSR-12

## Статус: ✅ ВЫПОЛНЕНО

## Связанные задачи

- 0034: Tariff Aggregate Root
- 0036: UserSubscription Aggregate Root
- 0046: SubscriptionService
- 0047: CLI Check Subscription Expiration
