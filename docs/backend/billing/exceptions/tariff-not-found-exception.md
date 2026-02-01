# TariffNotFoundException

## Описание

Исключение выбрасывается когда тариф не найден по ID или коду.

## Когда выбрасывается

- При попытке найти Tariff по ID в репозитории
- При попытке найти Tariff по коду в репозитории
- При попытке добавить подписку на несуществующий тариф

## Сообщение об ошибке

```
"Tariff with code 'PRO' not found"
// или
"Tariff with ID '987f6543-e89b-12d3-a456-426614174111' not found"
```

## Наследование

- Наследует от: `DomainException` (Shared Context)

## Методы

### static byCode(string $code): self

Создает исключение для не найденного тарифа по коду.

```php
throw TariffNotFoundException::byCode('PRO');
```

---

### static byId(string $id): self

Создает исключение для не найденного тарифа по ID.

```php
throw TariffNotFoundException::byId('987f6543-e89b-12d3-a456-426614174111');
```

## Статус реализации

- [ ] Класс создан с базовой структурой
- [ ] Наследование от DomainException установлено
- [ ] Методы byCode и byId реализованы
- [ ] Сообщения об ошибке информативны
- [ ] Unit тесты написаны (4+ теста)
- [ ] Integration тесты пройдены
- [ ] Код соответствует стилю проекта
- [ ] Документация актуальна

## Связанные сущности

- [Tariff Aggregate Root](../models/tariff-aggregate-root.md)
- [Exceptions Overview](overview.md)
- [Billing Context Overview](../overview.md)
