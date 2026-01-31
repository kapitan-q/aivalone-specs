# Задача 03: Реализация InvalidUuidException

## Описание

Реализовать специализированное исключение `InvalidUuidException` для защиты домена от некорректных идентификаторов.

## Требования

- [x] Класс должен наследоваться от `ValidationException`
- [x] Должен быть расположен в `src/Context/Shared/Domain/Exception/InvalidUuidException.php`
- [x] Выбрасывается при попытке создать UUID с невалидным форматом
- [x] Может использоваться не только для UUID, но и для ID основанных на UUID

## Реализация

```php
namespace App\Context\Shared\Domain\Exception;

class InvalidUuidException extends ValidationException
{
    public static function invalidFormat(string $value): self
    {
        return new self(sprintf('UUID "%s" имеет невалидный формат', $value));
    }
}
```

## Использование

```php
throw InvalidUuidException::invalidFormat($value);
```

## Критерии готовности

- [x] Класс реализован с методом `invalidFormat()`
- [ ] Покрыт unit-тестами
- [ ] Документация актуальна

## Зависимости

- [Задача 02: ValidationException](./backend_shared_02_validation_exception.md)

## Статус

done
