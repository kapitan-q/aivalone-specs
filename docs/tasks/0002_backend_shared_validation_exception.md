# Задача 02: Реализация ValidationException

## Описание

Реализовать базовое исключение `ValidationException` для сигнализации об ошибках валидации на границах домена.

## Требования

- [x] Класс должен наследоваться от `DomainException`
- [x] Должен быть расположен в `src/Context/Shared/Domain/Exception/ValidationException.php`
- [x] Используется для сигнализации о нарушении инвариантов Value Object или Entity

## Реализация

```php
namespace App\Context\Shared\Domain\Exception;

class ValidationException extends DomainException
{
    // специализированное исключение для ошибок валидации
}
```

## Использование

```php
throw new ValidationException("Некорректный формат");
```

## Критерии готовности

- [x] Класс реализован
- [ ] Покрыт unit-тестами
- [ ] Документация актуальна

## Зависимости

- [Задача 01: DomainException](./backend_shared_01_domain_exception.md)

## Статус

done
