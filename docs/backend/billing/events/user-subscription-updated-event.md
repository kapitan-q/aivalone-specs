# UserSubscriptionUpdated Event

## Описание

Событие `UserSubscriptionUpdated` генерируется при добавлении или удалении подписки пользователя.

## Когда выбрасывается

- При добавлении новой подписки (AddUserSubscription команда)
- При удалении подписки (RemoveSubscription команда)

## Структура события

```php
{
    eventId: string (UUID),
    aggregateId: string (subscriptionId),
    aggregateType: 'UserSubscription',
    eventName: 'UserSubscriptionUpdated',
    occurredAt: DateTimeImmutable,
    payload: {
        subscriptionId: string,
        userId: string,
        tariffId: string,
        tariffCode: string,
        createdAt: string (ISO 8601),
        validUntil: string|null (ISO 8601),
        status: string,
        action: 'ADDED'|'REMOVED'
    }
}
```

## Основные методы

### static forAddition(UserSubscription $subscription): self

Создает событие при добавлении подписки.

**Параметры**:
- `$subscription` — созданный объект UserSubscription

---

### static forRemoval(UserSubscription $subscription): self

Создает событие при удалении подписки.

**Параметры**:
- `$subscription` — объект UserSubscription перед удалением

---

### Методы доступа

```php
public function getSubscriptionId(): string
public function getUserId(): string
public function getTariffId(): string
public function getTariffCode(): string
public function getCreatedAt(): DateTimeImmutable
public function getValidUntil(): DateTimeImmutable|null
public function getStatus(): string
public function getAction(): string  // 'ADDED' или 'REMOVED'
public function isAdded(): bool
public function isRemoved(): bool
```

## Обработчики

События слушаются для синхронизации с Account Context и отправки уведомлений.

## Статус реализации

- [ ] Класс создан с базовой структурой
- [ ] Наследование от DomainEventInterface установлено
- [ ] Методы forAddition и forRemoval реализованы
- [ ] Все методы доступа реализованы
- [ ] Методы isAdded и isRemoved реализованы
- [ ] Payload правильно структурирован
- [ ] Unit тесты написаны (6+ тестов)
- [ ] Integration тесты пройдены
- [ ] Код соответствует стилю проекта
- [ ] Документация актуальна

## Связанные сущности

- [UserSubscription Aggregate Root](../models/user-subscription-aggregate-root.md)
- [Events Overview](overview.md)
- [Billing Context Overview](../overview.md)
