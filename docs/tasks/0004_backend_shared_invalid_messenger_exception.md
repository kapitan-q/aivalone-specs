# Задача 04: Реализация InvalidMessengerException

## Описание

Реализовать специализированное исключение `InvalidMessengerException` для валидации поддерживаемых мессенджеров.

## Требования

- [x] Класс должен наследоваться от `ValidationException`
- [x] Должен быть расположен в `src/Context/Shared/Domain/Exception/InvalidMessengerException.php`
- [x] Выбрасывается при попытке создать Messenger с неподдерживаемым кодом
- [x] Должен содержать static методы для создания с различными контекстами

## Реализация

```php
namespace App\Context\Shared\Domain\Exception;

class InvalidMessengerException extends ValidationException
{
    public static function unsupportedCode(string $code): self
    {
        return new self(sprintf('Мессенджер "%s" не поддерживается', $code));
    }
}
```

## Использование

```php
throw InvalidMessengerException::unsupportedCode($code);
```

## Критерии готовности

- [x] Класс реализован с методом `unsupportedCode()`
- [ ] Покрыт unit-тестами
- [ ] Документация актуальна

## Зависимости

- [Задача 02: ValidationException](./backend_shared_02_validation_exception.md)

## Статус

done
