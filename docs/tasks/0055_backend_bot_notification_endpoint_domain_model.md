# Задача 0055: NotificationEndpoint Domain Model

## Контекст

NotificationEndpoint — Aggregate Root контекста Bot. Инкапсулирует внешние идентификаторы (chat_id) и обеспечивает безопасную доставку сообщений без утечки данных в другие контексты.

## Цель

Создать Aggregate Root `NotificationEndpoint` для управления точками доставки сообщений.

## Спецификация

- [NotificationEndpoint](../backend/bot/models/notification-endpoint.md)

## Файлы для создания

```
src/Context/Bot/Domain/Model/NotificationEndpoint.php
tests/Unit/Context/Bot/Domain/Model/NotificationEndpointTest.php
```

## Требования

### NotificationEndpoint

```php
<?php

declare(strict_types=1);

namespace App\Context\Bot\Domain\Model;

use App\Context\Bot\Domain\Event\NotificationEndpointRegistered;
use App\Context\Bot\Domain\Event\NotificationEndpointRevoked;
use App\Context\Bot\Domain\Exception\EndpointAlreadyRevokedException;
use App\Context\Shared\Domain\Model\AggregateRoot;
use App\Context\Shared\Domain\Model\Messenger;
use App\Context\Shared\Domain\Model\UserId;

class NotificationEndpoint extends AggregateRoot
{
    private EndpointId $id;
    private UserId $userId;
    private Messenger $messenger;
    private string $externalTargetId; // chat_id - никогда не выходит за пределы контекста
    private EndpointStatus $status;
    private \DateTimeImmutable $createdAt;
    private ?\DateTimeImmutable $revokedAt;

    private function __construct(
        EndpointId $id,
        UserId $userId,
        Messenger $messenger,
        string $externalTargetId,
    ) {
        $this->id = $id;
        $this->userId = $userId;
        $this->messenger = $messenger;
        $this->externalTargetId = $externalTargetId;
        $this->status = EndpointStatus::ACTIVE;
        $this->createdAt = new \DateTimeImmutable();
        $this->revokedAt = null;
    }

    public static function register(
        UserId $userId,
        Messenger $messenger,
        string $externalTargetId,
    ): self {
        $endpoint = new self(
            id: EndpointId::generate(),
            userId: $userId,
            messenger: $messenger,
            externalTargetId: $externalTargetId,
        );

        $endpoint->recordEvent(new NotificationEndpointRegistered(
            endpointId: $endpoint->id,
            userId: $userId,
            messenger: $messenger,
            // ВАЖНО: externalTargetId НЕ передаётся в событие
        ));

        return $endpoint;
    }

    public function revoke(): void
    {
        if ($this->status === EndpointStatus::REVOKED) {
            throw new EndpointAlreadyRevokedException($this->id);
        }

        $this->status = EndpointStatus::REVOKED;
        $this->revokedAt = new \DateTimeImmutable();

        $this->recordEvent(new NotificationEndpointRevoked(
            endpointId: $this->id,
            userId: $this->userId,
            messenger: $this->messenger,
        ));
    }

    public function block(): void
    {
        $this->status = EndpointStatus::BLOCKED;
    }

    public function reactivate(): void
    {
        if ($this->status === EndpointStatus::REVOKED) {
            throw new EndpointAlreadyRevokedException($this->id);
        }

        $this->status = EndpointStatus::ACTIVE;
    }

    public function getId(): EndpointId
    {
        return $this->id;
    }

    public function getUserId(): UserId
    {
        return $this->userId;
    }

    public function getMessenger(): Messenger
    {
        return $this->messenger;
    }

    /**
     * ВНИМАНИЕ: Этот метод должен использоваться ТОЛЬКО внутри Bot Context.
     * Никогда не передавайте это значение в другие контексты!
     */
    public function getExternalTargetId(): string
    {
        return $this->externalTargetId;
    }

    public function getStatus(): EndpointStatus
    {
        return $this->status;
    }

    public function isActive(): bool
    {
        return $this->status->isActive();
    }

    public function canReceiveMessages(): bool
    {
        return $this->status->canReceiveMessages();
    }

    public function getCreatedAt(): \DateTimeImmutable
    {
        return $this->createdAt;
    }

    public function getRevokedAt(): ?\DateTimeImmutable
    {
        return $this->revokedAt;
    }
}
```

## Инварианты

1. `externalTargetId` никогда не передаётся за пределы Bot Context
2. После revoke() статус терминальный — нельзя изменить
3. Один endpoint на пользователя и мессенджер (в MVP)
4. События не содержат `externalTargetId`

## Тесты

- [x] Регистрация нового endpoint
- [x] Событие NotificationEndpointRegistered генерируется при регистрации
- [x] revoke() меняет статус на REVOKED
- [x] revoke() генерирует событие NotificationEndpointRevoked
- [x] Повторный revoke() выбрасывает EndpointAlreadyRevokedException
- [x] block() меняет статус на BLOCKED
- [x] reactivate() восстанавливает статус ACTIVE из BLOCKED
- [x] reactivate() выбрасывает исключение для REVOKED
- [x] canReceiveMessages() возвращает true только для ACTIVE

## Зависимости

- `App\Context\Shared\Domain\Model\AggregateRoot` (задача 0008)
- `App\Context\Shared\Domain\Model\Messenger` (задача 0006)
- `App\Context\Shared\Domain\Model\UserId` (задача 0005)
- `EndpointId` (задача 0051)
- `EndpointStatus` (задача 0052)
- `EndpointAlreadyRevokedException` (задача 0057)
- События (задача 0058)

## Definition of Done

- [x] Класс NotificationEndpoint создан
- [x] Все методы реализованы
- [x] Инварианты соблюдены
- [x] Unit-тесты написаны и проходят
- [x] Код соответствует PSR-12
- [x] PHPStan level 9 без ошибок
