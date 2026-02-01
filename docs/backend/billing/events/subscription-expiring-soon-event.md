# SubscriptionExpiringSoon Event

## Описание

Событие `SubscriptionExpiringSoon` генерируется за N дней до истечения подписки (7, 3, 1 день).

## Когда выбрасывается

- За 7 дней до истечения
- За 3 дня до истечения
- За 1 день до истечения

## Структура события

```php
{
    eventId: string (UUID),
    aggregateId: string (subscriptionId),
    aggregateType: 'UserSubscription',
    eventName: 'SubscriptionExpiringSoon',
    occurredAt: DateTimeImmutable,
    payload: {
        subscriptionId: string,
        userId: string,
        tariffId: string,
        tariffCode: string,
        daysUntilExpiration: int,
        validUntil: string (ISO 8601),
        expiringAt: string (ISO 8601)
    }
}
```

## Основные методы

### static create(UserSubscription $subscription, int $daysUntilExpiration): self

Создает событие предупреждения об истечении подписки.

**Параметры**:
- `$subscription` — объект UserSubscription
- `$daysUntilExpiration` — количество дней до истечения (7, 3 или 1)

**Выбрасывает**: DomainException если daysUntilExpiration не 1, 3 или 7

---

### Методы доступа

```php
public function getSubscriptionId(): string
public function getUserId(): string
public function getTariffId(): string
public function getTariffCode(): string
public function getDaysUntilExpiration(): int
public function getValidUntil(): DateTimeImmutable
public function getExpiringAt(): DateTimeImmutable
```

## Обработчики

- Отправляет уведомление пользователю (через Bot Context)
- Логирует информацию о скором истечении
- Может отправить Email уведомление

## Статус реализации

- [ ] Класс создан с базовой структурой
- [ ] Наследование от DomainEventInterface установлено
- [ ] Метод create реализован с валидацией daysUntilExpiration
- [ ] Все методы доступа реализованы
- [ ] Payload правильно структурирован
- [ ] Unit тесты написаны (6+ тестов)
- [ ] Integration тесты пройдены
- [ ] Код соответствует стилю проекта
- [ ] Документация актуальна

## Связанные сущности

- [UserSubscription Aggregate Root](../models/user-subscription-aggregate-root.md)
- [Events Overview](overview.md)
- [Billing Context Overview](../overview.md)
