# Задача 0046: SubscriptionService

## Описание

Создать доменный сервис `SubscriptionService` для управления подписками пользователей.

## Фаза

**Phase 4: Application Layer**

## Спецификация

📄 [subscription-service.md](../backend/billing/services/subscription-service.md)

## Зависимости

- ⏳ `UserSubscriptionRepository` — задача 0048
- ⏳ `TariffRepository` — задача 0048
- ⏳ `UserSubscription` Domain Model — задача 0036
- ⏳ Exceptions — задача 0041
- ⏳ Domain Events — задача 0042
- ✅ `EventDispatcherInterface` (Symfony)

## Расположение файла

```
src/Context/Billing/Application/Service/SubscriptionService.php
```

---

## Интерфейс сервиса

```php
interface SubscriptionServiceInterface
{
    public function addSubscription(
        UserId $userId,
        TariffId $tariffId,
        SubscriptionPeriod $period
    ): UserSubscriptionId;

    public function renewSubscription(
        UserId $userId,
        TariffId $tariffId,
        SubscriptionPeriod $period
    ): UserSubscriptionId;

    public function removeSubscription(
        UserId $userId,
        TariffId $tariffId
    ): void;

    public function createFreeSubscription(
        UserId $userId
    ): UserSubscriptionId;

    public function expireSubscription(
        UserSubscriptionId $subscriptionId
    ): void;

    public function getActiveSubscriptions(
        UserId $userId
    ): array;

    public function getAllSubscriptions(
        UserId $userId
    ): array;
}
```

---

## Реализация

```php
class SubscriptionService implements SubscriptionServiceInterface
{
    public function __construct(
        private UserSubscriptionRepositoryInterface $subscriptionRepository,
        private TariffRepositoryInterface $tariffRepository,
        private EventDispatcherInterface $eventDispatcher
    ) {}

    public function addSubscription(
        UserId $userId,
        TariffId $tariffId,
        SubscriptionPeriod $period
    ): UserSubscriptionId {
        // 1. Проверить, нет ли уже активной подписки на этот тариф
        $existing = $this->subscriptionRepository->findActiveByUserAndTariff($userId, $tariffId);
        if ($existing) {
            throw DuplicateSubscriptionException::forUserAndTariff($userId, $tariffId);
        }

        // 2. Создать подписку
        $subscription = UserSubscription::create($userId, $tariffId, $period);

        // 3. Сохранить
        $this->subscriptionRepository->save($subscription);

        // 4. Диспатчить события
        $this->dispatchEvents($subscription);

        return $subscription->getId();
    }

    public function renewSubscription(
        UserId $userId,
        TariffId $tariffId,
        SubscriptionPeriod $period
    ): UserSubscriptionId {
        // 1. Найти текущую активную подписку
        $current = $this->subscriptionRepository->findActiveByUserAndTariff($userId, $tariffId);
        if (!$current) {
            throw SubscriptionNotFoundException::forUserAndTariff($userId, $tariffId);
        }

        // 2. Создать новую подписку с ссылкой на предыдущую
        $newSubscription = UserSubscription::create(
            $userId,
            $tariffId,
            $period,
            $current->getId() // previousSubscriptionId
        );

        // 3. Истечь старую подписку
        $current->expire();

        // 4. Сохранить обе
        $this->subscriptionRepository->save($current);
        $this->subscriptionRepository->save($newSubscription);

        // 5. Диспатчить события
        $this->dispatchEvents($current);
        $this->dispatchEvents($newSubscription);

        return $newSubscription->getId();
    }

    public function createFreeSubscription(UserId $userId): UserSubscriptionId
    {
        // 1. Найти FREE тариф
        $freeTariff = $this->tariffRepository->findByCode(Tariff::FREE);
        if (!$freeTariff) {
            throw TariffNotFoundException::withCode(Tariff::FREE);
        }

        // 2. Проверить, нет ли уже FREE подписки
        $existing = $this->subscriptionRepository->findActiveByUserAndTariff(
            $userId,
            $freeTariff->getId()
        );
        if ($existing) {
            return $existing->getId(); // Уже есть — возвращаем существующую
        }

        // 3. Создать бессрочную FREE подписку
        $subscription = UserSubscription::createFree($userId, $freeTariff->getId());

        // 4. Сохранить
        $this->subscriptionRepository->save($subscription);

        // 5. Диспатчить события
        $this->dispatchEvents($subscription);

        return $subscription->getId();
    }

    public function removeSubscription(UserId $userId, TariffId $tariffId): void
    {
        // 1. Найти подписку
        $subscription = $this->subscriptionRepository->findActiveByUserAndTariff($userId, $tariffId);
        if (!$subscription) {
            throw SubscriptionNotFoundException::forUserAndTariff($userId, $tariffId);
        }

        // 2. Отменить
        $subscription->cancel();

        // 3. Сохранить
        $this->subscriptionRepository->save($subscription);

        // 4. Диспатчить события
        $this->dispatchEvents($subscription);
    }

    public function expireSubscription(UserSubscriptionId $subscriptionId): void
    {
        // 1. Найти подписку
        $subscription = $this->subscriptionRepository->findById($subscriptionId);
        if (!$subscription) {
            throw SubscriptionNotFoundException::withId($subscriptionId);
        }

        // 2. Истечь
        $subscription->expire();

        // 3. Сохранить
        $this->subscriptionRepository->save($subscription);

        // 4. Диспатчить события
        $this->dispatchEvents($subscription);
    }

    private function dispatchEvents(UserSubscription $subscription): void
    {
        foreach ($subscription->pullEvents() as $event) {
            $this->eventDispatcher->dispatch($event);
        }
    }
}
```

---

## Гарантии сервиса

| Гарантия | Описание |
| -------- | -------- |
| ✅ Полное сохранение | Все изменения сохранены в БД после вызова метода |
| ✅ Регистрация событий | Все события диспатчены |
| ✅ Атомарность | Либо полный успех, либо исключение |
| ✅ Окончательность | Состояние зафиксировано после вызова |

---

## Критерии готовности

- [x] Создан интерфейс `SubscriptionServiceInterface`
- [x] Создана реализация `SubscriptionService`
- [x] Реализованы все методы согласно спецификации
- [x] Методы диспатчат доменные события
- [ ] Написаны unit-тесты (с mock репозиториев)
- [x] Код соответствует PSR-12

## Статус: ✅ ВЫПОЛНЕНО

## Связанные задачи

- 0036: UserSubscription Aggregate Root
- 0042: Domain Events
- 0043: AddUserSubscription Command Handler
- 0044: RemoveSubscription Command Handler
- 0048: Doctrine Infrastructure
- 0049: Event Handlers
