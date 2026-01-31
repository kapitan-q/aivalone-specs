# Задача 0013: Реализация DuplicateMessengerException

## Описание

Реализовать специализированное исключение `DuplicateMessengerException` для обработки ошибки при попытке добавить мессенджер, который уже привязан к другому пользователю.

## Требования

- [x] Класс должен наследоваться от `DomainException` (Shared Context)
- [x] Должен быть расположен в `src/Context/Account/Domain/Exception/DuplicateMessengerException.php`
- [x] Выбрасывается при попытке добавить мессенджер, который уже привязан к другому пользователю
- [x] Должен содержать static методы для создания с различными контекстами

## Реализация

```php
namespace App\Context\Account\Domain\Exception;

class DuplicateMessengerException extends DomainException
{
    public static function alreadyUsed(Messenger $messenger, string $messengerId): self
    {
        return new self(sprintf(
            'Мессенджер "%s" с ID "%s" уже привязан к другому пользователю',
            $messenger->code(),
            $messengerId
        ));
    }
}
```

## Использование

```php
throw DuplicateMessengerException::alreadyUsed($messenger, $messengerId);
```

## Критерии готовности

- [x] Класс реализован с методом `alreadyUsed()`
- [x] Покрыт unit-тестами
- [x] Документация актуальна

## Зависимости

- Shared Context: DomainException
- Shared Context: Messenger

## Статус

done
