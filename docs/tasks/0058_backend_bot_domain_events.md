# Задача 0058: Bot Domain Events

## Контекст

Bot Context публикует события для асинхронной интеграции с Account Context. События намеренно НЕ содержат externalTargetId для соблюдения границ контекста.

## Цель

Создать доменные события контекста Bot.

## Спецификации

- [Events Overview](../backend/bot/events/overview.md)
- [BotUserConnected](../backend/bot/events/bot-user-connected.md)
- [BotUserDisconnected](../backend/bot/events/bot-user-disconnected.md)
- [NotificationEndpointRegistered](../backend/bot/events/notification-endpoint-registered.md)
- [NotificationEndpointRevoked](../backend/bot/events/notification-endpoint-revoked.md)

## Файлы для создания

```
src/Context/Bot/Domain/Event/BotUserConnected.php
src/Context/Bot/Domain/Event/BotUserDisconnected.php
src/Context/Bot/Domain/Event/NotificationEndpointRegistered.php
src/Context/Bot/Domain/Event/NotificationEndpointRevoked.php
tests/Unit/Context/Bot/Domain/Event/BotEventsTest.php
```

## Требования

### BotUserConnected

```php
<?php

declare(strict_types=1);

namespace App\Context\Bot\Domain\Event;

use App\Context\Shared\Domain\Event\DomainEventInterface;
use App\Context\Shared\Domain\Enum\Messenger;
use App\Context\Shared\Domain\ValueObject\Uuid;

final readonly class BotUserConnected implements DomainEventInterface
{
    public function __construct(
        private Uuid $userId,
        private Messenger $messenger,
        private \DateTimeImmutable $occurredAt,
    ) {}

    public static function create(Uuid $userId, Messenger $messenger): self
    {
        return new self(
            userId: $userId,
            messenger: $messenger,
            occurredAt: new \DateTimeImmutable(),
        );
    }

    public function getUserId(): Uuid
    {
        return $this->userId;
    }

    public function getMessenger(): Messenger
    {
        return $this->messenger;
    }

    public function getOccurredAt(): \DateTimeImmutable
    {
        return $this->occurredAt;
    }

    public function getEventName(): string
    {
        return 'bot.user.connected';
    }

    /**
     * @return array<string, mixed>
     */
    public function toArray(): array
    {
        return [
            'user_id' => $this->userId->toString(),
            'messenger' => $this->messenger->value,
            'occurred_at' => $this->occurredAt->format(\DateTimeInterface::ATOM),
        ];
    }
}
```

### BotUserDisconnected

```php
<?php

declare(strict_types=1);

namespace App\Context\Bot\Domain\Event;

use App\Context\Shared\Domain\Event\DomainEventInterface;
use App\Context\Shared\Domain\Enum\Messenger;
use App\Context\Shared\Domain\ValueObject\Uuid;

final readonly class BotUserDisconnected implements DomainEventInterface
{
    public function __construct(
        private Uuid $userId,
        private Messenger $messenger,
        private \DateTimeImmutable $occurredAt,
    ) {}

    public static function create(Uuid $userId, Messenger $messenger): self
    {
        return new self(
            userId: $userId,
            messenger: $messenger,
            occurredAt: new \DateTimeImmutable(),
        );
    }

    public function getUserId(): Uuid
    {
        return $this->userId;
    }

    public function getMessenger(): Messenger
    {
        return $this->messenger;
    }

    public function getOccurredAt(): \DateTimeImmutable
    {
        return $this->occurredAt;
    }

    public function getEventName(): string
    {
        return 'bot.user.disconnected';
    }

    /**
     * @return array<string, mixed>
     */
    public function toArray(): array
    {
        return [
            'user_id' => $this->userId->toString(),
            'messenger' => $this->messenger->value,
            'occurred_at' => $this->occurredAt->format(\DateTimeInterface::ATOM),
        ];
    }
}
```

### NotificationEndpointRegistered

```php
<?php

declare(strict_types=1);

namespace App\Context\Bot\Domain\Event;

use App\Context\Bot\Domain\Model\EndpointId;
use App\Context\Shared\Domain\Event\DomainEventInterface;
use App\Context\Shared\Domain\Enum\Messenger;
use App\Context\Shared\Domain\ValueObject\Uuid;

final readonly class NotificationEndpointRegistered implements DomainEventInterface
{
    public function __construct(
        private EndpointId $endpointId,
        private Uuid $userId,
        private Messenger $messenger,
        // ВАЖНО: externalTargetId намеренно НЕ включён!
    ) {}

    public function getEndpointId(): EndpointId
    {
        return $this->endpointId;
    }

    public function getUserId(): Uuid
    {
        return $this->userId;
    }

    public function getMessenger(): Messenger
    {
        return $this->messenger;
    }

    public function getOccurredAt(): \DateTimeImmutable
    {
        return new \DateTimeImmutable();
    }

    public function getEventName(): string
    {
        return 'bot.endpoint.registered';
    }

    /**
     * @return array<string, mixed>
     */
    public function toArray(): array
    {
        return [
            'endpoint_id' => $this->endpointId->toString(),
            'user_id' => $this->userId->toString(),
            'messenger' => $this->messenger->value,
        ];
    }
}
```

### NotificationEndpointRevoked

```php
<?php

declare(strict_types=1);

namespace App\Context\Bot\Domain\Event;

use App\Context\Bot\Domain\Model\EndpointId;
use App\Context\Shared\Domain\Event\DomainEventInterface;
use App\Context\Shared\Domain\Enum\Messenger;
use App\Context\Shared\Domain\ValueObject\Uuid;

final readonly class NotificationEndpointRevoked implements DomainEventInterface
{
    public function __construct(
        private EndpointId $endpointId,
        private Uuid $userId,
        private Messenger $messenger,
    ) {}

    public function getEndpointId(): EndpointId
    {
        return $this->endpointId;
    }

    public function getUserId(): Uuid
    {
        return $this->userId;
    }

    public function getMessenger(): Messenger
    {
        return $this->messenger;
    }

    public function getOccurredAt(): \DateTimeImmutable
    {
        return new \DateTimeImmutable();
    }

    public function getEventName(): string
    {
        return 'bot.endpoint.revoked';
    }

    /**
     * @return array<string, mixed>
     */
    public function toArray(): array
    {
        return [
            'endpoint_id' => $this->endpointId->toString(),
            'user_id' => $this->userId->toString(),
            'messenger' => $this->messenger->value,
        ];
    }
}
```

## Критерии безопасности

**ВАЖНО:** Ни одно событие не содержит `externalTargetId` (chat_id). Это принципиальное архитектурное решение — внешние идентификаторы никогда не покидают Bot Context.

## Тесты

- [ ] BotUserConnected создаётся с правильными данными
- [ ] BotUserDisconnected создаётся с правильными данными
- [ ] NotificationEndpointRegistered не содержит externalTargetId
- [ ] NotificationEndpointRevoked не содержит externalTargetId
- [ ] getEventName() возвращает уникальные имена
- [ ] toArray() возвращает корректные данные для сериализации
- [ ] Ни одно событие не сериализует externalTargetId

## Зависимости

- `App\Context\Shared\Domain\Event\DomainEventInterface` (задача 0007)
- `App\Context\Shared\Domain\Enum\Messenger` (задача 0006)
- `App\Context\Shared\Domain\ValueObject\Uuid` (задача 0005)
- `EndpointId` (задача 0051)

## Definition of Done

- [ ] Все 4 класса событий созданы
- [ ] События не содержат externalTargetId
- [ ] Unit-тесты написаны и проходят
- [ ] Код соответствует PSR-12
- [ ] PHPStan level 9 без ошибок
