# BotUserDisconnected Event Specification

## Назначение

Событие `BotUserDisconnected` генерируется когда пользователь отключается от бота (заблокировал бота, удалил чат и т.д.).

## Атрибуты

| Поле              | Тип                  | Описание                                            |
| ----------------- | -------------------- | --------------------------------------------------- |
| `userId`          | `string`             | UUID пользователя                                   |
| `messenger`       | `string`             | Код мессенджера                                     |
| `reason`          | `string`             | Причина отключения (blocked, chat_deleted, etc.)    |
| `occurredAt`      | `DateTimeImmutable`  | Время возникновения события                         |

## Причины отключения

| Reason          | Описание                                        |
| --------------- | ----------------------------------------------- |
| `blocked`       | Пользователь заблокировал бота                  |
| `chat_deleted`  | Чат удалён                                      |
| `kicked`        | Бот удалён из группы                            |
| `manual`        | Ручное отключение пользователем                 |

## Реализация

```php
final readonly class BotUserDisconnected implements DomainEventInterface
{
    public function __construct(
        public string $userId,
        public string $messenger,
        public string $reason,
        public DateTimeImmutable $occurredAt,
    ) {}
}
```

## Когда генерируется

* Telegram: получен update с `my_chat_member` и статус `kicked` или `left`
* При ошибке доставки сообщения с кодом "chat not found" или "bot blocked"

## Потребители

| Контекст | Обработчик | Действие |
| -------- | ---------- | -------- |
| Account  | BotUserDisconnectedHandler | Пометка мессенджера как неактивного |

## Связанные документы

* [Events Overview](overview.md)
* [NotificationEndpointRevoked](notification-endpoint-revoked.md)

## Статус реализации

* [ ] Класс события создан
* [ ] Интеграция с EventBus
* [ ] Обработка webhook updates
* [ ] Unit тесты написаны
