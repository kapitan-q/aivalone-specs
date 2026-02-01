# SubscriptionService

## Описание

Application Service `SubscriptionService` выполняет всю бизнес-логику управления подписками и тарифами.

**Ключевая ответственность**: Сервис отвечает за **полное сохранение сущностей, публикацию событий и управление транзакциями**. Это означает, что вызов любого метода сервиса гарантирует:
- ✅ Все изменения сохранены в базе данных
- ✅ Все события зарегистрированы в агрегатах
- ✅ Все события опубликованы через EventBus
- ✅ Действие выполнено окончательно и атомарно

**Архитектурный паттерн** (как в Account/UserService):
- **Application Layer (SubscriptionService)**: Валидирует данные, проверяет бизнес-инварианты, вызывает методы доменной модели, сохраняет и публикует события
- **Domain Layer (UserSubscription)**: Содержит бизнес-логику, регистрирует события через `recordEvent()`
- **Command Handlers**: Только валидируют входные данные и конвертируют типы, делегируют все остальное сервису

## Зависимости

```php
public function __construct(
    private readonly TariffRepositoryInterface $tariffRepository,
    private readonly UserSubscriptionRepositoryInterface $subscriptionRepository,
    private readonly EventBusInterface $eventBus,
) {}
```

## Основные методы

### addSubscription(UserId $userId, Tariff $tariff, SubscriptionPeriod $period): UserSubscription

Добавляет новую подписку для пользователя.

**Параметры**:
- `$userId` — идентификатор пользователя (уже конвертирован из string)
- `$tariff` — enum тарифа из Shared Context (уже конвертирован)
- `$period` — период подписки (MONTH, YEAR)

**Ответственность Application Layer (SubscriptionService)**:
- Получает Tariff по коду из репозитория
  - Выбрасывает `TariffNotFoundException` если не найден
- Проверяет уникальность подписки (нет активной подписки на этот тариф)
  - Выбрасывает `DuplicateSubscriptionException` если существует
- Вызывает `UserSubscription::create(userId, tariffId, period)`
- Сохраняет: `UserSubscriptionRepository::save($subscription)`
- **Публикует события**: `EventBus::publish(...$subscription->pullEvents())`

**Ответственность Domain Layer (UserSubscription)**:
- Создание объекта с инициализацией всех полей
- Вычисление validUntil на основе period
- Регистрация события `UserSubscriptionUpdated`

**Возвращает**: UserSubscription объект (уже сохраненный и событие опубликовано)

**Выбрасывает**: TariffNotFoundException, DuplicateSubscriptionException

---

### createFreeSubscription(UserId $userId): UserSubscription

Создаёт бессрочную FREE подписку для пользователя.

**Параметры**:
- `$userId` — идентификатор пользователя

**Логика**:
1. Получает FREE тариф из репозитория
   - Выбрасывает `TariffNotFoundException` если не найден (критическая ошибка конфигурации)
2. Проверяет, нет ли уже FREE подписки у пользователя
   - Если есть — возвращает существующую (идемпотентность)
3. Создаёт `UserSubscription::createFree(userId, freeTariffId)`
4. Сохраняет и публикует события

**Возвращает**: UserSubscription с period = null, validUntil = null

**Использование**: Вызывается из UserRegisteredEventHandler

---

### renewSubscription(UserId $userId, Tariff $tariff, SubscriptionPeriod $period): UserSubscription

Продлевает подписку пользователя (создаёт новую подписку с ссылкой на предыдущую).

**Параметры**:
- `$userId` — идентификатор пользователя
- `$tariff` — enum тарифа из Shared Context
- `$period` — период продления (MONTH, YEAR)

**Логика**:
1. Получает Tariff по коду из репозитория
   - Выбрасывает `TariffNotFoundException` если не найден
2. Находит активную подписку пользователя на этот тариф
   - Выбрасывает `SubscriptionNotFoundException` если не найдена
3. Создаёт НОВУЮ подписку с previousSubscriptionId = текущая подписка
4. Сохраняет и публикует события SubscriptionRenewed, UserSubscriptionUpdated

**Возвращает**: Новая UserSubscription (старая остаётся без изменений)

**Выбрасывает**: TariffNotFoundException, SubscriptionNotFoundException

---

### removeSubscription(UserId $userId, Tariff $tariff): void

Удаляет (отменяет) подписку пользователя.

**Параметры**:
- `$userId` — идентификатор пользователя
- `$tariff` — enum тарифа из Shared Context

**Логика**:
- Получает Tariff по коду из репозитория
  - Выбрасывает `TariffNotFoundException` если не найден
- Находит активную подписку по userId и tariffId
  - Выбрасывает `SubscriptionNotFoundException` если не найдена
- Вызывает `$subscription->cancel()` (бизнес-логика в доменной модели)
- Сохраняет: `UserSubscriptionRepository::save($subscription)`
- **Публикует события**: `EventBus::publish(...$subscription->pullEvents())`

**Ответственность Domain Layer (UserSubscription)**:
- Обновление статуса: `status = CANCELLED`
- Регистрация события `UserSubscriptionUpdated`

**Возвращает**: void (но подписка уже обновлена и событие опубликовано)

**Выбрасывает**: TariffNotFoundException, SubscriptionNotFoundException

---

### getActiveSubscriptions(UserId $userId): array

Получает все активные подписки пользователя.

**Параметры**:
- `$userId` — идентификатор пользователя

**Возвращает**: `array<UserSubscription>` — только подписки со статусом ACTIVE

---

### getAllSubscriptions(UserId $userId): array

Получает все подписки пользователя (включая неактивные).

**Параметры**:
- `$userId` — идентификатор пользователя

**Возвращает**: `array<UserSubscription>` — все подписки

---

### expireSubscription(UserSubscription $subscription): void

Помечает подписку как истекшую.

**Параметры**:
- `$subscription` — объект подписки

**Логика**:
1. Вызывает `$subscription->expire()`
2. Сохраняет и публикует события

**Использование**: Вызывается из CLI команды CheckSubscriptionExpiration

---

## Использование в Command Handlers

```php
// В Command Handler для добавления подписки
public function handle(AddUserSubscriptionCommand $command): UserSubscription
{
    $userId = UserId::fromString($command->userId);
    $tariff = Tariff::fromCode($command->tariffCode);
    $period = SubscriptionPeriod::fromCode($command->period);

    return $this->subscriptionService->addSubscription($userId, $tariff, $period);
}

// В Command Handler для продления
public function handle(RenewSubscriptionCommand $command): UserSubscription
{
    $userId = UserId::fromString($command->userId);
    $tariff = Tariff::fromCode($command->tariffCode);
    $period = SubscriptionPeriod::fromCode($command->period);

    return $this->subscriptionService->renewSubscription($userId, $tariff, $period);
}
```

## Статус реализации

- [ ] Класс создан в Application Layer
- [ ] Метод addSubscription реализован с поддержкой period
- [ ] Метод createFreeSubscription реализован
- [ ] Метод renewSubscription реализован
- [ ] Метод removeSubscription реализован
- [ ] Методы getActiveSubscriptions, getAllSubscriptions реализованы
- [ ] Метод expireSubscription реализован
- [ ] Бизнес-логика корректна
- [ ] Исключения выбрасываются правильно
- [ ] EventBus инъектирован в конструктор
- [ ] Все события публикуются после сохранения
- [ ] Unit тесты написаны (15+ тестов)
- [ ] Integration тесты пройдены
- [ ] Код соответствует стилю проекта
- [ ] Документация актуальна

## Связанные сущности

- [AddUserSubscription Process](../processes/add-user-subscription-process.md)
- [RenewSubscription Process](../processes/renew-subscription-process.md)
- [RemoveSubscription Process](../processes/remove-subscription-process.md)
- [Processes Overview](../processes/overview.md)
- [UserSubscription Aggregate Root](../models/user-subscription-aggregate-root.md)
- [SubscriptionPeriod Enum](../enums/subscription-period.md)
- [UserRegisteredEventHandler](../handlers/user-registered-event-handler.md)
- [TariffRepository](../infrastructure/tariff-repository.md)
- [UserSubscriptionRepository](../infrastructure/user-subscription-repository.md)
- [Account Context: UserService](../../account/services/user-service.md) (аналогичный паттерн)
- [Services Overview](overview.md)
- [Billing Context Overview](../overview.md)
