# InvalidSubscriptionException

## Описание

Исключение выбрасывается когда данные подписки некорректны или нарушают инварианты.

## Когда выбрасывается

- validUntil находится в прошлом при создании подписки
- userId некорректный или пусто
- tariffId некорректный или пусто

## Сообщение об ошибке

```
"Subscription validUntil cannot be in the past"
// или другие варианты в зависимости от причины
```

## Наследование

- Наследует от: `ValidationException` (Shared Context)

## Методы

### static validUntilInPast(DateTimeImmutable $validUntil): self

Создает исключение когда дата окончания подписки в прошлом.

```php
throw InvalidSubscriptionException::validUntilInPast($invalidDate);
```

---

### static invalidUserId(): self

Создает исключение когда userId некорректный.

```php
throw InvalidSubscriptionException::invalidUserId();
```

---

### static invalidTariffId(): self

Создает исключение когда tariffId некорректный.

```php
throw InvalidSubscriptionException::invalidTariffId();
```

## Статус реализации

- [ ] Класс создан с базовой структурой
- [ ] Наследование от ValidationException установлено
- [ ] Методы validUntilInPast, invalidUserId, invalidTariffId реализованы
- [ ] Сообщения об ошибке информативны
- [ ] Unit тесты написаны (4+ теста)
- [ ] Integration тесты пройдены
- [ ] Код соответствует стилю проекта
- [ ] Документация актуальна

## Связанные сущности

- [UserSubscription Aggregate Root](../models/user-subscription-aggregate-root.md)
- [Exceptions Overview](overview.md)
- [Billing Context Overview](../overview.md)
