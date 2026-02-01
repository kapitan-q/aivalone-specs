# InvalidOptionTypeException

## Описание

Исключение выбрасывается когда тип опции тарифа невалидный. Используется при валидации значений enum TariffOptionType из Shared Context.

## Когда выбрасывается

- При попытке конвертировать строку в TariffOptionType enum, если строка не соответствует доступным типам
- При попытке создать опцию тарифа с невалидным типом

## Сообщение об ошибке

```
"Invalid option type 'INVALID_TYPE'. Available types: MAX_CONSTRAINT, BOOL, TEXT"
```

## Наследование

- Наследует от: `ValidationException` (Shared Context)

## Методы

### static withType(string $type, array $availableTypes): self

Создает исключение с информацией о невалидном типе и доступных вариантах.

```php
throw InvalidOptionTypeException::withType('INVALID', ['MAX_CONSTRAINT', 'BOOL', 'TEXT']);
```

## Статус реализации

- [ ] Класс создан с базовой структурой
- [ ] Наследование от ValidationException установлено
- [ ] Метод withType реализован
- [ ] Сообщение об ошибке информативно
- [ ] Unit тесты написаны (3+ теста)
- [ ] Integration тесты пройдены
- [ ] Код соответствует стилю проекта
- [ ] Документация актуальна

## Связанные сущности

- [TariffOptionType Enum](../../shared/models/tariff-option-type.md)
- [Exceptions Overview](overview.md)
- [Billing Context Overview](../overview.md)
