# BotUserConnected Event Specification

## Назначение

Событие `BotUserConnected` генерируется при первом подключении пользователя к боту через мессенджер (обычно при команде /start).

## Атрибуты

| Поле              | Тип                  | Описание                                            |
| ----------------- | -------------------- | --------------------------------------------------- |
| `userId`          | `string`             | UUID пользователя (из Account Context)              |
| `messenger`       | `string`             | Код мессенджера (telegram, whatsapp, и т.д.)        |
| `occurredAt`      | `DateTimeImmutable`  | Время возникновения события                         |

## Реализация

```php
final readonly class BotUserConnected implements DomainEventInterface
{
    public function __construct(
        public string $userId,
        public string $messenger,
        public DateTimeImmutable $occurredAt,
    ) {}

    public function getOccurredAt(): DateTimeImmutable
    {
        return $this->occurredAt;
    }
}
```

## Когда генерируется

* При первом /start от нового пользователя (после создания User в Account Context)
* При повторном /start от существующего пользователя

## Безопасность

**ВАЖНО**: Событие НЕ содержит:
- externalUserId (telegram user_id)
- externalChatId (chat_id)
- Любых других внешних идентификаторов

## Потребители

| Контекст | Обработчик | Действие |
| -------- | ---------- | -------- |
| Account  | BotUserConnectedHandler | Обновление lastActivityAt пользователя |

## Связанные документы

* [Events Overview](overview.md)
* [NotificationEndpointRegistered](notification-endpoint-registered.md)

## Статус реализации

* [ ] Класс события создан
* [ ] Интеграция с EventBus
* [ ] Unit тесты написаны
