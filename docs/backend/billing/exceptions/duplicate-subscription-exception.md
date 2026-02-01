# DuplicateSubscriptionException

## Описание

Исключение выбрасывается когда у пользователя уже есть активная подписка на данный тариф.

## Когда выбрасывается

- При попытке добавить подписку на тариф, который уже активен для пользователя
- При попытке добавить вторую активную подписку на один тариф

## Сообщение об ошибке

```
"User '123e4567-e89b-12d3-a456-426614174000' already has active subscription for tariff 'PRO'"
```

## Наследование

- Наследует от: `DomainException` (Shared Context)

## Методы

### static forUserAndTariff(UserId $userId, Tariff $tariff): self

Создает исключение с информацией о пользователе и тарифе.

```php
throw DuplicateSubscriptionException::forUserAndTariff(
    UserId::fromString('123e4567-e89b-12d3-a456-426614174000'),
    'PRO'
);
```

## Статус реализации

- [ ] Класс создан с базовой структурой
- [ ] Наследование от DomainException установлено
- [ ] Метод forUserAndTariff реализован
- [ ] Сообщение об ошибке информативно
- [ ] Unit тесты написаны (3+ теста)
- [ ] Integration тесты пройдены
- [ ] Код соответствует стилю проекта
- [ ] Документация актуальна

## Связанные сущности

- [UserSubscription Aggregate Root](../models/user-subscription-aggregate-root.md)
- [SubscriptionService](../services/overview.md)
- [Exceptions Overview](overview.md)
- [Billing Context Overview](../overview.md)
