# UserSubscriptionRepository

## Описание

Репозиторий для работы с агрегатом `UserSubscription`. Предоставляет методы для поиска, фильтрации и сохранения подписок пользователей.

## Расположение

```
# Интерфейс (Domain Layer)
src/Context/Billing/Domain/Repository/UserSubscriptionRepositoryInterface.php

# Реализация (Infrastructure Layer)
src/Context/Billing/Infrastructure/Persistence/DoctrineUserSubscriptionRepository.php
```

## Интерфейс

```php
namespace App\Context\Billing\Domain\Repository;

use App\Context\Billing\Domain\Model\UserSubscription;
use App\Context\Billing\Domain\Model\UserSubscriptionId;
use App\Context\Billing\Domain\Model\TariffId;
use App\Context\Shared\Domain\Model\UserId;

interface UserSubscriptionRepositoryInterface
{
    /**
     * Найти подписку по ID
     */
    public function findById(UserSubscriptionId $id): ?UserSubscription;

    /**
     * Найти все активные подписки пользователя
     */
    public function findActiveByUserId(UserId $userId): array;

    /**
     * Найти все подписки пользователя (включая неактивные)
     */
    public function findAllByUserId(UserId $userId): array;

    /**
     * Найти активную подписку пользователя на конкретный тариф
     */
    public function findActiveByUserAndTariff(UserId $userId, TariffId $tariffId): ?UserSubscription;

    /**
     * Найти все активные подписки с validUntil <= указанной даты
     */
    public function findExpiredSubscriptions(\DateTimeImmutable $date): array;

    /**
     * Найти активные подписки, истекающие в указанный период
     */
    public function findExpiringSoon(\DateTimeImmutable $from, \DateTimeImmutable $to): array;

    /**
     * Сохранить подписку
     */
    public function save(UserSubscription $subscription): void;

    /**
     * Удалить подписку
     */
    public function remove(UserSubscription $subscription): void;
}
```

## Методы

### findById(UserSubscriptionId $id): ?UserSubscription

Находит подписку по уникальному идентификатору.

**Параметры**:
- `$id` — UserSubscriptionId (UUID)

**Возвращает**: `UserSubscription` или `null` если не найдена

---

### findActiveByUserId(UserId $userId): array

Находит все активные подписки пользователя.

**Параметры**:
- `$userId` — UserId (из Account Context)

**Возвращает**: `array<UserSubscription>` — только подписки со статусом ACTIVE

**Использование**: REST API `/api/users/{id}/subscriptions`, GetUserLimits Query

---

### findAllByUserId(UserId $userId): array

Находит все подписки пользователя (включая EXPIRED и CANCELLED).

**Параметры**:
- `$userId` — UserId

**Возвращает**: `array<UserSubscription>` — все подписки пользователя

**Использование**: История подписок, аналитика

---

### findActiveByUserAndTariff(UserId $userId, TariffId $tariffId): ?UserSubscription

Находит активную подписку пользователя на конкретный тариф.

**Параметры**:
- `$userId` — UserId
- `$tariffId` — TariffId

**Возвращает**: `UserSubscription` или `null`

**Использование**:
- Проверка на дублирование при создании подписки
- Поиск подписки при продлении

---

### findExpiredSubscriptions(DateTimeImmutable $date): array

Находит активные подписки с validUntil <= указанной даты.

**Параметры**:
- `$date` — дата для сравнения (обычно now())

**Возвращает**: `array<UserSubscription>` — подписки, которые истекли

**Использование**: CLI команда CheckSubscriptionExpiration

**Важно**: Исключает подписки с validUntil = null (бессрочные)

---

### findExpiringSoon(DateTimeImmutable $from, DateTimeImmutable $to): array

Находит активные подписки, истекающие в указанный период.

**Параметры**:
- `$from` — начало периода
- `$to` — конец периода

**Возвращает**: `array<UserSubscription>` — подписки в указанном диапазоне

**Использование**: CLI команда для уведомлений о скором истечении

**Пример**:
```php
// Подписки, истекающие через 7 дней
$from = new DateTimeImmutable('+6 days 23:59:59');
$to = new DateTimeImmutable('+7 days 23:59:59');
$expiringSoon = $repository->findExpiringSoon($from, $to);
```

---

### save(UserSubscription $subscription): void

Сохраняет подписку (insert или update).

**Параметры**:
- `$subscription` — объект UserSubscription

---

### remove(UserSubscription $subscription): void

Удаляет подписку из базы данных.

**Параметры**:
- `$subscription` — объект UserSubscription

## Doctrine реализация

```php
namespace App\Context\Billing\Infrastructure\Persistence;

use App\Context\Billing\Domain\Enum\SubscriptionStatus;
use App\Context\Billing\Domain\Model\UserSubscription;
use App\Context\Billing\Domain\Model\UserSubscriptionId;
use App\Context\Billing\Domain\Model\TariffId;
use App\Context\Billing\Domain\Repository\UserSubscriptionRepositoryInterface;
use App\Context\Shared\Domain\Model\UserId;
use Doctrine\ORM\EntityManagerInterface;

final class DoctrineUserSubscriptionRepository implements UserSubscriptionRepositoryInterface
{
    public function __construct(
        private readonly EntityManagerInterface $em,
    ) {}

    public function findById(UserSubscriptionId $id): ?UserSubscription
    {
        return $this->em->find(UserSubscription::class, $id->toString());
    }

    public function findActiveByUserId(UserId $userId): array
    {
        return $this->em->getRepository(UserSubscription::class)
            ->findBy([
                'userId' => $userId->toString(),
                'status' => SubscriptionStatus::ACTIVE->value,
            ]);
    }

    public function findAllByUserId(UserId $userId): array
    {
        return $this->em->getRepository(UserSubscription::class)
            ->findBy(['userId' => $userId->toString()], ['createdAt' => 'DESC']);
    }

    public function findActiveByUserAndTariff(UserId $userId, TariffId $tariffId): ?UserSubscription
    {
        return $this->em->getRepository(UserSubscription::class)
            ->findOneBy([
                'userId' => $userId->toString(),
                'tariffId' => $tariffId->toString(),
                'status' => SubscriptionStatus::ACTIVE->value,
            ]);
    }

    public function findExpiredSubscriptions(\DateTimeImmutable $date): array
    {
        return $this->em->createQueryBuilder()
            ->select('s')
            ->from(UserSubscription::class, 's')
            ->where('s.status = :status')
            ->andWhere('s.validUntil IS NOT NULL')
            ->andWhere('s.validUntil <= :date')
            ->setParameter('status', SubscriptionStatus::ACTIVE->value)
            ->setParameter('date', $date)
            ->getQuery()
            ->getResult();
    }

    public function findExpiringSoon(\DateTimeImmutable $from, \DateTimeImmutable $to): array
    {
        return $this->em->createQueryBuilder()
            ->select('s')
            ->from(UserSubscription::class, 's')
            ->where('s.status = :status')
            ->andWhere('s.validUntil IS NOT NULL')
            ->andWhere('s.validUntil BETWEEN :from AND :to')
            ->setParameter('status', SubscriptionStatus::ACTIVE->value)
            ->setParameter('from', $from)
            ->setParameter('to', $to)
            ->getQuery()
            ->getResult();
    }

    public function save(UserSubscription $subscription): void
    {
        $this->em->persist($subscription);
        $this->em->flush();
    }

    public function remove(UserSubscription $subscription): void
    {
        $this->em->remove($subscription);
        $this->em->flush();
    }
}
```

## Статус реализации

- [ ] Интерфейс создан в Domain Layer
- [ ] Doctrine реализация создана
- [ ] Все методы реализованы
- [ ] Маппинг Doctrine настроен
- [ ] Индексы БД созданы (userId, tariffId, status, validUntil)
- [ ] Unit тесты написаны (10+ тестов)
- [ ] Integration тесты пройдены
- [ ] Сервис зарегистрирован в DI контейнере

## Связанные сущности

- [UserSubscription Aggregate Root](../models/user-subscription-aggregate-root.md)
- [UserSubscriptionId Value Object](../models/user-subscription-id-value-object.md)
- [TariffRepository](./tariff-repository.md)
- [CheckSubscriptionExpiration Process](../processes/check-subscription-expiration-process.md)
- [Billing Context Overview](../overview.md)
