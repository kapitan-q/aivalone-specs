# EndpointNotFoundException Specification

## Назначение

Исключение `EndpointNotFoundException` выбрасывается когда NotificationEndpoint не найден.

## Наследование

Расширяет `DomainException` из Shared Context.

## Атрибуты

| Поле         | Тип      | Описание                              |
| ------------ | -------- | ------------------------------------- |
| `identifier` | `string` | Идентификатор (userId или endpointId) |
| `type`       | `string` | Тип поиска (by_id, by_user)           |

## Реализация

```php
final class EndpointNotFoundException extends DomainException
{
    public function __construct(
        public readonly string $identifier,
        public readonly string $type,
    ) {
        parent::__construct(
            sprintf('NotificationEndpoint not found (%s: %s)', $type, $identifier)
        );
    }

    public static function withId(string $endpointId): self
    {
        return new self($endpointId, 'by_id');
    }

    public static function forUser(string $userId, string $messenger): self
    {
        return new self(
            sprintf('%s/%s', $userId, $messenger),
            'by_user_messenger'
        );
    }
}
```

## Когда выбрасывается

* В `NotificationEndpointRepository::findById()` когда endpoint не найден
* При попытке отправить сообщение пользователю без endpoint

## Связанные документы

* [Exceptions Overview](overview.md)
* [NotificationEndpoint](../models/notification-endpoint.md)

## Статус реализации

* [ ] Класс создан
* [ ] Unit тесты написаны
