# EndpointAlreadyRevokedException Specification

## Назначение

Исключение `EndpointAlreadyRevokedException` выбрасывается при попытке отозвать уже отозванный endpoint.

## Наследование

Расширяет `DomainException` из Shared Context.

## Атрибуты

| Поле         | Тип      | Описание                     |
| ------------ | -------- | ---------------------------- |
| `endpointId` | `string` | UUID endpoint                |

## Реализация

```php
final class EndpointAlreadyRevokedException extends DomainException
{
    public function __construct(
        public readonly string $endpointId,
    ) {
        parent::__construct(
            sprintf('NotificationEndpoint "%s" is already revoked', $endpointId)
        );
    }
}
```

## Когда выбрасывается

* В `NotificationEndpoint::revoke()` когда status уже REVOKED

## Связанные документы

* [Exceptions Overview](overview.md)
* [NotificationEndpoint](../models/notification-endpoint.md)

## Статус реализации

* [ ] Класс создан
* [ ] Unit тесты написаны
