# SubscriptionNotFoundException

## Описание

Исключение выбрасывается когда подписка не найдена.

## Когда выбрасывается

- При попытке удалить несуществующую подписку
- При попытке получить информацию о несуществующей подписке
- При RemoveSubscription команде, если подписка не найдена

## Сообщение об ошибке

```
"Subscription not found for user '123e4567-e89b-12d3-a456-426614174000' and tariff 'PRO'"
```

## Наследование

- Наследует от: `DomainException` (Shared Context)

## Методы

### static forUserAndTariff(UserId $userId, Tariff $tariff): self

Создает исключение с информацией о пользователе и тарифе.

```php
throw SubscriptionNotFoundException::forUserAndTariff(
    UserId::fromString('123e4567-e89b-12d3-a456-426614174000'),
    'PRO'
);
```

---

### static byId(SubscriptionId $subscriptionId): self

Создает исключение с информацией о подписке по ID.

```php
throw SubscriptionNotFoundException::byId($subscriptionId);
```

## Статус реализации

- [ ] Класс создан с базовой структурой
- [ ] Наследование от DomainException установлено
- [ ] Методы forUserAndTariff и byId реализованы
- [ ] Сообщения об ошибке информативны
- [ ] Unit тесты написаны (4+ теста)
- [ ] Integration тесты пройдены
- [ ] Код соответствует стилю проекта
- [ ] Документация актуальна

## Связанные сущности

- [UserSubscription Aggregate Root](../models/user-subscription-aggregate-root.md)
- [SubscriptionService](../services/overview.md)
- [Exceptions Overview](overview.md)
- [Billing Context Overview](../overview.md)
