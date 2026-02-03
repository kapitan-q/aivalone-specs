# MessageDeliveryException Specification

## Назначение

Исключение `MessageDeliveryException` выбрасывается при ошибке доставки сообщения через мессенджер.

## Наследование

Расширяет `DomainException` из Shared Context.

## Атрибуты

| Поле         | Тип      | Описание                              |
| ------------ | -------- | ------------------------------------- |
| `messenger`  | `string` | Код мессенджера                       |
| `errorCode`  | `string` | Код ошибки от API мессенджера         |
| `errorMessage` | `string` | Сообщение об ошибке                 |

## Реализация

```php
final class MessageDeliveryException extends DomainException
{
    public function __construct(
        public readonly string $messenger,
        public readonly string $errorCode,
        public readonly string $errorMessage,
    ) {
        parent::__construct(
            sprintf('[%s] Delivery failed: %s (%s)', $messenger, $errorMessage, $errorCode)
        );
    }

    public static function botBlocked(string $messenger): self
    {
        return new self($messenger, 'BOT_BLOCKED', 'Bot was blocked by user');
    }

    public static function chatNotFound(string $messenger): self
    {
        return new self($messenger, 'CHAT_NOT_FOUND', 'Chat not found');
    }

    public static function rateLimited(string $messenger): self
    {
        return new self($messenger, 'RATE_LIMITED', 'Rate limit exceeded');
    }

    public static function apiError(string $messenger, string $code, string $message): self
    {
        return new self($messenger, $code, $message);
    }

    public function isRecoverable(): bool
    {
        return match($this->errorCode) {
            'RATE_LIMITED' => true,
            'TIMEOUT' => true,
            default => false,
        };
    }

    public function shouldRevokeEndpoint(): bool
    {
        return match($this->errorCode) {
            'BOT_BLOCKED', 'CHAT_NOT_FOUND', 'USER_DEACTIVATED' => true,
            default => false,
        };
    }
}
```

## Когда выбрасывается

* В Messenger Adapters при ошибке API мессенджера
* При timeout или сетевых ошибках

## Обработка

* Если `isRecoverable()` — повторная попытка через retry policy
* Если `shouldRevokeEndpoint()` — отзыв endpoint и генерация события

## Связанные документы

* [Exceptions Overview](overview.md)
* [MessengerAdapter](../infrastructure/messenger-adapter-interface.md)

## Статус реализации

* [ ] Класс создан
* [ ] Фабричные методы реализованы
* [ ] Интеграция с адаптерами
* [ ] Unit тесты написаны
