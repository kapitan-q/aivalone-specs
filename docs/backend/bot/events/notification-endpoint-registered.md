# NotificationEndpointRegistered Event Specification

## Назначение

Событие `NotificationEndpointRegistered` генерируется при регистрации нового NotificationEndpoint для пользователя.

## Атрибуты

| Поле              | Тип                  | Описание                                            |
| ----------------- | -------------------- | --------------------------------------------------- |
| `endpointId`      | `string`             | UUID endpoint                                       |
| `userId`          | `string`             | UUID пользователя                                   |
| `messenger`       | `string`             | Код мессенджера                                     |
| `occurredAt`      | `DateTimeImmutable`  | Время возникновения события                         |

## Реализация

```php
final readonly class NotificationEndpointRegistered implements DomainEventInterface
{
    public function __construct(
        public string $endpointId,
        public string $userId,
        public string $messenger,
        public DateTimeImmutable $occurredAt,
    ) {}
}
```

## Когда генерируется

* При создании NotificationEndpoint через `NotificationEndpoint::create()`
* Обычно происходит при первом /start пользователя

## Безопасность

**ВАЖНО**: Событие НЕ содержит:
- externalTargetId (chat_id)
- Любых внешних идентификаторов мессенджера

Это намеренно — внешние идентификаторы инкапсулированы в Bot Context.

## Потребители

| Контекст | Обработчик | Действие |
| -------- | ---------- | -------- |
| Account  | NotificationEndpointRegisteredHandler | Сохранение информации о доступных каналах доставки |

## Связанные документы

* [NotificationEndpoint](../models/notification-endpoint.md)
* [Events Overview](overview.md)

## Статус реализации

* [ ] Класс события создан
* [ ] Генерация в NotificationEndpoint::create()
* [ ] Интеграция с EventBus
* [ ] Unit тесты написаны
