# EndpointStatus Enum Specification

## Назначение

`EndpointStatus` — Enum, определяющий возможные статусы `NotificationEndpoint`.

## Значения

| Значение    | Описание                                           |
| ----------- | -------------------------------------------------- |
| `ACTIVE`    | Endpoint активен, доступен для доставки сообщений  |
| `REVOKED`   | Endpoint отозван, сообщения не доставляются        |

## Реализация

```php
enum EndpointStatus: string
{
    case ACTIVE = 'active';
    case REVOKED = 'revoked';
}
```

## Использование

```php
$endpoint = NotificationEndpoint::create($userId, $messenger, $chatId);
// $endpoint->getStatus() === EndpointStatus::ACTIVE

$endpoint->revoke();
// $endpoint->getStatus() === EndpointStatus::REVOKED
```

## Переходы статусов

```mermaid
graph LR
    A[ACTIVE] -->|revoke()| B[REVOKED]
```

Переход REVOKED → ACTIVE не предусмотрен. При необходимости создаётся новый endpoint.

## Связанные документы

* [NotificationEndpoint](notification-endpoint.md)

## Статус реализации

* [ ] Enum EndpointStatus создан
* [ ] Unit тесты написаны
