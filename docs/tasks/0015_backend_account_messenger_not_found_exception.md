# Задача 0015: Реализация MessengerNotFoundException

## Описание

Реализовать специализированное исключение `MessengerNotFoundException` для обработки ошибки при попытке удалить мессенджер, который не привязан к пользователю.

## Требования

- [x] Класс должен наследоваться от `DomainException` (Shared Context)
- [x] Должен быть расположен в `src/Context/Account/Domain/Exception/MessengerNotFoundException.php`
- [x] Выбрасывается при попытке удалить несуществующий мессенджер
- [x] Должен содержать static методы для создания с различными контекстами

## Реализация

```php
namespace App\Context\Account\Domain\Exception;

class MessengerNotFoundException extends DomainException
{
    public static function notFound(Messenger $messenger, string $messengerId): self
    {
        return new self(sprintf(
            'Мессенджер "%s" с ID "%s" не привязан к пользователю',
            $messenger->code(),
            $messengerId
        ));
    }
}
```

## Использование

```php
throw MessengerNotFoundException::notFound($messenger, $messengerId);
```

## Критерии готовности

- [x] Класс реализован с методом `notFound()`
- [x] Покрыт unit-тестами
- [x] Документация актуальна

## Зависимости

- Shared Context: DomainException
- Shared Context: Messenger

## Статус

done
