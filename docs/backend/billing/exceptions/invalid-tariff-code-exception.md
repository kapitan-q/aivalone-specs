# InvalidTariffException

## Описание

Исключение выбрасывается когда код тарифа некорректен или не существует. 
Используется при валидации значений enum Tariff из Shared Context.

## Когда выбрасывается

- При попытке конвертировать строку в Tariff enum, если строка не соответствует доступным кодам
- При попытке создать подписку с невалидным тарифом

## Сообщение об ошибке

```
"Invalid tariff code 'INVALID_CODE'. Available codes: FREE, BASE, PRO, ENTERPRISE"
```

## Наследование

- Наследует от: `TariffValidationException` (Shared Context)

## Методы

### static withCode(string $code, array $availableCodes): self

Создает исключение с информацией о невалидном коде и доступных вариантах.

```php
throw InvalidTariffException::withCode('INVALID', ['FREE', 'BASE', 'PRO', 'ENTERPRISE']);
```

## Статус реализации

- [ ] Класс создан с базовой структурой
- [ ] Наследование от TariffValidationException установлено
- [ ] Метод withCode реализован
- [ ] Сообщение об ошибке информативно
- [ ] Unit тесты написаны (3+ теста)
- [ ] Integration тесты пройдены
- [ ] Код соответствует стилю проекта
- [ ] Документация актуальна

## Связанные сущности

- [Tariff Enum](../../shared/models/tariff.md)
- [TariffValidationException](../../shared/exceptions/tariff-validation-exception.md)
- [Exceptions Overview](overview.md)
- [Billing Context Overview](../overview.md)
