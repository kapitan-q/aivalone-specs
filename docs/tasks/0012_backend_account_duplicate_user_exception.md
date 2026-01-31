# Задача 0012: Реализация DuplicateUserException

## Описание

Реализовать специализированное исключение `DuplicateUserException` для обработки ошибки при попытке создать пользователя с существующей комбинацией мессенджера и идентификатора.

## Требования

- [x] Класс должен наследоваться от `DomainException` (Shared Context)
- [x] Должен быть расположен в `src/Context/Account/Domain/Exception/DuplicateUserException.php`
- [x] Выбрасывается при попытке создать пользователя с дублирующимся `messenger` + `messengerId`
- [x] Должен содержать static методы для создания с различными контекстами

## Реализация

```php
namespace App\Context\Account\Domain\Exception;

class DuplicateUserException extends DomainException
{
    public static function byMessenger(Messenger $messenger, string $messengerId): self
    {
        return new self(sprintf(
            'Пользователь с мессенджером "%s" и ID "%s" уже существует',
            $messenger->code(),
            $messengerId
        ));
    }
}
```

## Использование

```php
throw DuplicateUserException::byMessenger($messenger, $messengerId);
```

## Критерии готовности

- [x] Класс реализован с методом `byMessenger()`
- [x] Покрыт unit-тестами
- [x] Документация актуальна

## Зависимости

- Shared Context: DomainException
- Shared Context: Messenger

## Статус

done
