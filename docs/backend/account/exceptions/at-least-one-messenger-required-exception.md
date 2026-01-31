# AtLeastOneMessengerRequiredException Specification

## Назначение

`AtLeastOneMessengerRequiredException` — исключение, выбрасываемое при попытке нарушить бизнес-инвариант о том, что пользователь должен иметь минимум один мессенджер.

Используется в методе `User::removeMessenger()` при попытке удалить последний (единственный) мессенджер пользователя.

## Наследование

Наследник базового `DomainException` из Shared Context.

## Расположение

`src/Context/Account/Domain/Exception/AtLeastOneMessengerRequiredException.php`

## Реализация

```php
namespace App\Context\Account\Domain\Exception;

use App\Context\Shared\Domain\Exception\DomainException;

class AtLeastOneMessengerRequiredException extends DomainException
{
    public static function cannotRemoveLastOne(): self
    {
        return new self(
            'Невозможно удалить последний мессенджер пользователя. ' .
            'Пользователь должен иметь минимум один мессенджер.'
        );
    }
}
```

## Область применения

* Процесс удаления мессенджера у пользователя (метод `User::removeMessenger()`)
* Проверка инварианта: пользователь должен иметь минимум один мессенджер

## Когда выбрасывается

* При попытке удалить последний (единственный) мессенджер пользователя

## Инварианты

* **Минимум один мессенджер**: пользователь всегда должен иметь хотя бы один мессенджер
* Нельзя оставить пользователя без мессенджеров

## Использование

```php
// В методе User::removeMessenger()
public function removeMessenger(Messenger $messenger, string $messengerId): void
{
    // Поиск мессенджера в коллекции
    $key = null;
    foreach ($this->messengers as $idx => $userMessenger) {
        if ($userMessenger->messenger === $messenger && 
            $userMessenger->messengerId === $messengerId) {
            $key = $idx;
            break;
        }
    }

    // Мессенджер не найден
    if ($key === null) {
        throw MessengerNotFoundException::byCode(
            $messenger->code(),
            $messengerId
        );
    }

    // Проверка инварианта: минимум один мессенджер
    if (count($this->messengers) === 1) {
        throw AtLeastOneMessengerRequiredException::cannotRemoveLastOne();
    }

    // Удаление мессенджера
    unset($this->messengers[$key]);
    $this->messengers = array_values($this->messengers); // Переиндексирование
    $this->updatedAt = new DateTimeImmutable();
    
    // Публикация события
    $this->recordEvent(
        new UserMessengersUpdated($this->userId, $this->messengers)
    );
}
```

## Различие между исключениями

| Исключение | Статус | Причина | Пример |
|-----------|--------|--------|---------|
| `MessengerNotFoundException` | 404 | Мессенджер не найден в коллекции | Попытка удалить несуществующий Telegram ID |
| `AtLeastOneMessengerRequiredException` | 409 | Нарушение инварианта | Попытка удалить единственный мессенджер |

## HTTP коды ошибок (для REST API)

- **409 Conflict**: Попытка нарушить инвариант (удалить последний мессенджер)

## Связанные документы

* [MessengerNotFoundException](./messenger-not-found-exception.md)
* [User Domain Model](../model/user.md)
* [Account Context Overview](../overview.md)
* [Shared Context DomainException](../../../shared/exceptions/domain-exception.md)

## Статус

* [x] Спецификация написана
* [x] Реализация исключения AtLeastOneMessengerRequiredException
