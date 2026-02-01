# Обзор исключений Billing Context

## Иерархия исключений

```
Exception (PHP)
├── DomainException (Shared Context)
│   ├── TariffNotFoundException
│   ├── DuplicateSubscriptionException
│   ├── SubscriptionNotFoundException
│   └── TariffOptionNotFoundException
└── ValidationException (Shared Context)
    ├── InvalidSubscriptionException
    ├── InvalidTariffException (валидация Tariff из Shared)
    └── InvalidOptionTypeException
```

## Все исключения Billing Context

| Исключение | Тип | Описание | Файл |
| ---------- | --- | -------- | ---- |
| **TariffNotFoundException** | DomainException | Тариф не найден по ID или коду | [tariff-not-found-exception.md](tariff-not-found-exception.md) |
| **InvalidSubscriptionException** | ValidationException | Данные подписки некорректны или нарушают инварианты | [invalid-subscription-exception.md](invalid-subscription-exception.md) |
| **DuplicateSubscriptionException** | DomainException | У пользователя уже активна подписка на этот тариф | [duplicate-subscription-exception.md](duplicate-subscription-exception.md) |
| **SubscriptionNotFoundException** | DomainException | Подписка не найдена | [subscription-not-found-exception.md](subscription-not-found-exception.md) |
| **InvalidTariffException** | ValidationException | Код тарифа невалидный (Tariff enum из Shared) | [invalid-tariff-code-exception.md](invalid-tariff-code-exception.md) |
| **InvalidOptionTypeException** | ValidationException | Тип опции невалидный | [invalid-option-type-exception.md](invalid-option-type-exception.md) |
| **TariffOptionNotFoundException** | DomainException | Опция не найдена в тарифе | [tariff-option-not-found-exception.md](tariff-option-not-found-exception.md) |

## Структура исключений

```
docs/backend/billing/exceptions/
├── overview.md                                    # Этот файл (оглавление)
├── tariff-not-found-exception.md                  # Спецификация исключения
├── invalid-subscription-exception.md              # Спецификация исключения
├── duplicate-subscription-exception.md            # Спецификация исключения
├── subscription-not-found-exception.md            # Спецификация исключения
├── invalid-tariff-code-exception.md               # Спецификация исключения
├── invalid-option-type-exception.md               # Спецификация исключения
└── tariff-option-not-found-exception.md           # Спецификация исключения
```

## Использование исключений в контексте

### DomainException (нарушения бизнес-правил)

```php
throw TariffNotFoundException::byCode('PRO');
throw DuplicateSubscriptionException::forUserAndTariff($userId, 'PRO');
throw SubscriptionNotFoundException::forUserAndTariff($userId, 'FREE');
throw TariffOptionNotFoundException::byCode('MAX_GROUPS', 'FREE');
```

### ValidationException (невалидные входные данные)

```php
throw InvalidSubscriptionException::validUntilInPast($date);
throw InvalidTariffException::withCode('UNKNOWN', ['FREE', 'BASE', 'PRO', 'ENTERPRISE']);
throw InvalidOptionTypeException::withType('INVALID', ['MAX_CONSTRAINT', 'BOOL', 'TEXT']);
```

## Связанные документы

* [Shared Context Exceptions](../../shared/exceptions/overview.md)
* [Billing Context Overview](../overview.md)
