# SubscriptionRenewed Event

## Описание

Событие `SubscriptionRenewed` генерируется при продлении подписки. При продлении создаётся новая подписка, которая ссылается на предыдущую.

## Когда выбрасывается

- При успешном продлении подписки (RenewSubscription команда)
- Только когда создана новая подписка с previousSubscriptionId

## Структура события

```php
{
    eventId: string (UUID),
    aggregateId: string (новый subscriptionId),
    aggregateType: 'UserSubscription',
    eventName: 'SubscriptionRenewed',
    occurredAt: DateTimeImmutable,
    payload: {
        subscriptionId: string,           // ID новой подписки
        previousSubscriptionId: string,   // ID предыдущей подписки
        userId: string,
        tariffId: string,
        tariffCode: string,
        period: string,                   // 'MONTH' | 'YEAR'
        createdAt: string (ISO 8601),
        validUntil: string (ISO 8601)
    }
}
```

## Основные методы

### static create(UserSubscription $newSubscription, UserSubscription $previousSubscription): self

Создает событие продления подписки.

**Параметры**:
- `$newSubscription` — новая подписка (созданная при продлении)
- `$previousSubscription` — предыдущая подписка

---

### Методы доступа

```php
public function getSubscriptionId(): string
public function getPreviousSubscriptionId(): string
public function getUserId(): string
public function getTariffId(): string
public function getTariffCode(): string
public function getPeriod(): string
public function getCreatedAt(): DateTimeImmutable
public function getValidUntil(): DateTimeImmutable
```

## Обработчики (в других контекстах)

- **Notifications Context**: Отправляет уведомление о продлении подписки
- **Account Context**: Может обновить информацию о тарифах пользователя

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
- [RenewSubscription Process](../processes/renew-subscription-process.md)
- [SubscriptionPeriod Enum](../enums/subscription-period.md)
- [Events Overview](overview.md)
- [Billing Context Overview](../overview.md)
