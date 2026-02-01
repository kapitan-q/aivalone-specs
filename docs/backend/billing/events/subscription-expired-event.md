# SubscriptionExpired Event

## Описание

Событие `SubscriptionExpired` генерируется когда подписка истекла (validUntil < текущей даты).

## Когда выбрасывается

- Событие формируется в момент когда срок подписки истек
- Только для подписок со статусом ACTIVE

## Структура события

```php
{
    eventId: string (UUID),
    aggregateId: string (subscriptionId),
    aggregateType: 'UserSubscription',
    eventName: 'SubscriptionExpired',
    occurredAt: DateTimeImmutable,
    payload: {
        subscriptionId: string,
        userId: string,
        tariffId: string,
        tariffCode: string,
        expiredAt: string (ISO 8601),
        validUntil: string (ISO 8601)
    }
}
```

## Основные методы

### static create(UserSubscription $subscription, DateTimeImmutable $expiredAt): self

Создает событие истечения подписки.

**Параметры**:
- `$subscription` — объект UserSubscription
- `$expiredAt` — дата истечения

---

### Методы доступа

```php
public function getSubscriptionId(): string
public function getUserId(): string
public function getTariffId(): string
public function getTariffCode(): string
public function getExpiredAt(): DateTimeImmutable
public function getValidUntil(): DateTimeImmutable
```

## Обработчики

- Обновляет статус подписки на EXPIRED в репозитории
- Отправляет уведомление пользователю (через Bot Context)
- Синхронизирует информацию с Account Context

## Статус реализации

- [ ] Класс создан с базовой структурой
- [ ] Наследование от DomainEventInterface установлено
- [ ] Метод create реализован
- [ ] Все методы доступа реализованы
- [ ] Payload правильно структурирован
- [ ] Unit тесты написаны (5+ тестов)
- [ ] Integration тесты пройдены
- [ ] Код соответствует стилю проекта
- [ ] Документация актуальна

## Связанные сущности

- [UserSubscription Aggregate Root](../models/user-subscription-aggregate-root.md)
- [Events Overview](overview.md)
- [Billing Context Overview](../overview.md)
