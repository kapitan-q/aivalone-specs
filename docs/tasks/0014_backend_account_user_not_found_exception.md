# Задача 0014: Реализация UserNotFoundException

## Описание

Реализовать специализированное исключение `UserNotFoundException` для обработки ошибки при попытке обращения к несуществующему пользователю.

## Требования

- [x] Класс должен наследоваться от `DomainException` (Shared Context)
- [x] Должен быть расположен в `src/Context/Account/Domain/Exception/UserNotFoundException.php`
- [x] Выбрасывается при попытке получить несуществующего пользователя
- [x] Должен содержать static методы для создания с различными контекстами

## Реализация

```php
namespace App\Context\Account\Domain\Exception;

class UserNotFoundException extends DomainException
{
    public static function byId(UserId $userId): self
    {
        return new self(sprintf('Пользователь "%s" не найден', $userId->toString()));
    }
}
```

## Использование

```php
throw UserNotFoundException::byId($userId);
```

## Критерии готовности

- [x] Класс реализован с методом `byId()`
- [x] Покрыт unit-тестами
- [x] Документация актуальна

## Зависимости

- Shared Context: DomainException
- Account Context: UserId

## Статус

done
