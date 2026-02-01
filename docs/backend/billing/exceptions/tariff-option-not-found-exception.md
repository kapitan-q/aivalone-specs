# TariffOptionNotFoundException

## Описание

Исключение выбрасывается когда опция тарифа не найдена.

## Когда выбрасывается

- При попытке обновить опцию, которая не существует в тарифе
- При попытке удалить опцию, которая не существует в тарифе
- При попытке получить опцию по коду, которая не существует

## Сообщение об ошибке

```
"Option 'MAX_GROUPS' not found in tariff 'FREE'"
```

## Наследование

- Наследует от: `DomainException` (Shared Context)

## Методы

### static byCode(string $optionCode, string $tariffCode): self

Создает исключение с информацией о коде опции и тарифе.

```php
throw TariffOptionNotFoundException::byCode('MAX_GROUPS', 'FREE');
```

## Статус реализации

- [ ] Класс создан с базовой структурой
- [ ] Наследование от DomainException установлено
- [ ] Метод byCode реализован
- [ ] Сообщение об ошибке информативно
- [ ] Unit тесты написаны (3+ теста)
- [ ] Integration тесты пройдены
- [ ] Код соответствует стилю проекта
- [ ] Документация актуальна

## Связанные сущности

- [Tariff Aggregate Root](../models/tariff-aggregate-root.md)
- [TariffOption Value Object](../models/tariff-option-value-object.md)
- [Exceptions Overview](overview.md)
- [Billing Context Overview](../overview.md)
