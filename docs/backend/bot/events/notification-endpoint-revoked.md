# NotificationEndpointRevoked Event Specification

## Назначение

Событие `NotificationEndpointRevoked` генерируется при отзыве NotificationEndpoint.

## Атрибуты

| Поле              | Тип                  | Описание                                            |
| ----------------- | -------------------- | --------------------------------------------------- |
| `endpointId`      | `string`             | UUID endpoint                                       |
| `userId`          | `string`             | UUID пользователя                                   |
| `messenger`       | `string`             | Код мессенджера                                     |
| `reason`          | `string`             | Причина отзыва                                      |
| `occurredAt`      | `DateTimeImmutable`  | Время возникновения события                         |

## Причины отзыва

| Reason          | Описание                                        |
| --------------- | ----------------------------------------------- |
| `user_blocked`  | Пользователь заблокировал бота                  |
| `chat_deleted`  | Чат удалён                                      |
| `manual`        | Ручной отзыв (админ или пользователь)           |
| `delivery_failed` | Множественные ошибки доставки                 |

## Реализация

```php
final readonly class NotificationEndpointRevoked implements DomainEventInterface
{
    public function __construct(
        public string $endpointId,
        public string $userId,
        public string $messenger,
        public string $reason,
        public DateTimeImmutable $occurredAt,
    ) {}
}
```

## Когда генерируется

* При вызове `NotificationEndpoint::revoke()`
* При обнаружении недоступности endpoint (blocked, deleted)

## Потребители

| Контекст | Обработчик | Действие |
| -------- | ---------- | -------- |
| Account  | NotificationEndpointRevokedHandler | Обновление статуса канала доставки |

## Связанные документы

* [NotificationEndpoint](../models/notification-endpoint.md)
* [Events Overview](overview.md)

## Статус реализации

* [ ] Класс события создан
* [ ] Генерация в NotificationEndpoint::revoke()
* [ ] Интеграция с EventBus
* [ ] Unit тесты написаны
