# Задача 0098: CommandBusInterface, Symfony Messenger транспорты и Integration Messages

## Контекст

Monitoring Context взаимодействует с auth-service и listener-service через Message Queue. Для этого используется Symfony Messenger с AMQP транспортами — аналогично существующему `python_bridge` транспорту.

Application Layer не должен зависеть от Symfony напрямую. По аналогии с существующим `EventBusInterface` → `SymfonyEventBus` в Shared контексте, создаётся `CommandBusInterface` → `SymfonyCommandBus`. Application Services диспатчат команды через `CommandBusInterface`, а Symfony Messenger роутит их в нужный транспорт по конфигурации. Входящие события обрабатываются `#[AsMessageHandler]` handlers.

## Цель

1. Создать `CommandBusInterface` в Shared Application Layer и его Symfony реализацию
2. Настроить Symfony Messenger транспорты для взаимодействия с auth-service и listener-service
3. Создать Integration Command и Event DTO

## Спецификация

- [Infrastructure Overview](../backend/monitoring/infrastructure/overview.md)
- [Authorize Session Process](../backend/monitoring/processes/authorize-session.md)
- [Subscribe to Group Process](../backend/monitoring/processes/subscribe-to-group.md)

## Файлы для создания/изменения

```
# CommandBusInterface (Shared — аналогично EventBusInterface)
src/Context/Shared/Application/Command/CommandBusInterface.php
src/Context/Shared/Infrastructure/Command/SymfonyCommandBus.php

# Integration Commands (исходящие — Backend → auth/listener)
src/Context/Monitoring/Application/Command/Integration/
├── InitAuthCommand.php
├── SubmitAuthCodeCommand.php
├── SubmitPasswordCommand.php
├── StopSessionCommand.php
├── JoinGroupCommand.php
└── LeaveGroupCommand.php

# Integration Events (входящие — auth/listener → Backend)
src/Context/Monitoring/Application/Event/Integration/
├── AuthCodeSentIntegrationEvent.php
├── Auth2FARequiredIntegrationEvent.php
├── AuthSuccessIntegrationEvent.php
├── AuthFailedIntegrationEvent.php
├── SessionExpiredIntegrationEvent.php
├── GroupJoinedIntegrationEvent.php
├── GroupJoinFailedIntegrationEvent.php
└── MessageReceivedIntegrationEvent.php

# Конфигурация Symfony Messenger
config/packages/messenger.yaml  # (изменение существующего)

tests/Unit/Context/Monitoring/Application/Command/Integration/
├── InitAuthCommandTest.php
├── JoinGroupCommandTest.php
└── LeaveGroupCommandTest.php
```

## Требования

### CommandBusInterface (Shared Application Layer)

По аналогии с существующим `EventBusInterface` в `Shared/Application/Event/`.

```php
<?php

declare(strict_types=1);

namespace App\Context\Shared\Application\Command;

interface CommandBusInterface
{
    public function dispatch(object $command): mixed;
}
```

### SymfonyCommandBus (Shared Infrastructure Layer)

```php
<?php

declare(strict_types=1);

namespace App\Context\Shared\Infrastructure\Command;

use App\Context\Shared\Application\Command\CommandBusInterface;
use Symfony\Component\Messenger\MessageBusInterface;
use Symfony\Component\Messenger\Stamp\HandledStamp;

class SymfonyCommandBus implements CommandBusInterface
{
    public function __construct(
        private readonly MessageBusInterface $messageBus,
    ) {}

    public function dispatch(object $command): mixed
    {
        $envelope = $this->messageBus->dispatch($command);
        $handledStamp = $envelope->last(HandledStamp::class);

        return $handledStamp?->getResult();
    }
}
```

### Архитектура

```
┌──────────────────────────────────────────────────────────────────────┐
│                     Symfony Messenger                                │
│                                                                      │
│  ┌─────────────────────────────────┐                                 │
│  │   monitoring_auth transport     │  ← AMQP exchange               │
│  │   (outgoing to auth-service)    │    monitoring.auth.commands     │
│  └─────────────────────────────────┘                                 │
│                                                                      │
│  ┌─────────────────────────────────┐                                 │
│  │   monitoring_listener transport │  ← AMQP exchange               │
│  │   (outgoing to listener-svc)   │    monitoring.listener.commands  │
│  └─────────────────────────────────┘                                 │
│                                                                      │
│  ┌─────────────────────────────────┐                                 │
│  │   monitoring_events transport   │  ← AMQP exchange               │
│  │   (incoming from auth/listener) │    monitoring.events             │
│  └─────────────────────────────────┘                                 │
└──────────────────────────────────────────────────────────────────────┘
```

**Принцип**: Application Layer диспатчит команды через `CommandBusInterface` (не зависит от Symfony). `SymfonyCommandBus` реализация оборачивает `MessageBusInterface`, Symfony Messenger роутит в нужный транспорт по конфигурации. Входящие события десериализуются и маршрутизируются на `#[AsMessageHandler]` handlers (задача 0095).

### Конфигурация messenger.yaml

```yaml
framework:
    messenger:
        transports:
            # ... existing transports ...

            # Исходящие команды в auth-service
            monitoring_auth:
                dsn: '%env(MONITORING_AUTH_AMQP_DSN)%'
                options:
                    exchange:
                        name: monitoring_auth_commands
                        type: direct
                    queues:
                        monitoring_auth_commands:
                            binding_keys: [command]

            # Исходящие команды в listener-service
            monitoring_listener:
                dsn: '%env(MONITORING_LISTENER_AMQP_DSN)%'
                options:
                    exchange:
                        name: monitoring_listener_commands
                        type: direct
                    queues:
                        monitoring_listener_commands:
                            binding_keys: [command]

            # Входящие события от auth/listener
            monitoring_events:
                dsn: '%env(MONITORING_EVENTS_AMQP_DSN)%'
                options:
                    exchange:
                        name: monitoring_events
                        type: direct
                    queues:
                        monitoring_events_auth:
                            binding_keys: [auth_event]
                        monitoring_events_listener:
                            binding_keys: [listener_event]

        routing:
            # ... existing routing ...

            # Auth-service commands
            'App\Context\Monitoring\Application\Command\Integration\InitAuthCommand': monitoring_auth
            'App\Context\Monitoring\Application\Command\Integration\SubmitAuthCodeCommand': monitoring_auth
            'App\Context\Monitoring\Application\Command\Integration\SubmitPasswordCommand': monitoring_auth

            # Listener-service commands
            'App\Context\Monitoring\Application\Command\Integration\StopSessionCommand': monitoring_listener
            'App\Context\Monitoring\Application\Command\Integration\JoinGroupCommand': monitoring_listener
            'App\Context\Monitoring\Application\Command\Integration\LeaveGroupCommand': monitoring_listener

            # Integration events → async (handlers в задаче 0095)
            'App\Context\Monitoring\Application\Event\Integration\*': async
```

### Integration Commands (исходящие)

#### InitAuthCommand

```php
<?php

declare(strict_types=1);

namespace App\Context\Monitoring\Application\Command\Integration;

final readonly class InitAuthCommand
{
    /**
     * @param array<string, string> $authData
     */
    public function __construct(
        public string $sessionId,
        public string $messengerType,
        public array $authData,
        public string $correlationId,
    ) {}
}
```

#### SubmitAuthCodeCommand

```php
final readonly class SubmitAuthCodeCommand
{
    public function __construct(
        public string $sessionId,
        public string $code,
        public string $correlationId,
    ) {}
}
```

#### SubmitPasswordCommand

```php
final readonly class SubmitPasswordCommand
{
    public function __construct(
        public string $sessionId,
        public string $password,
        public string $correlationId,
    ) {}
}
```

#### StopSessionCommand

```php
final readonly class StopSessionCommand
{
    public function __construct(
        public string $sessionId,
        public string $messengerType,
        public string $correlationId,
    ) {}
}
```

#### JoinGroupCommand

```php
final readonly class JoinGroupCommand
{
    public function __construct(
        public string $groupId,
        public string $externalGroupId,
        public string $messengerType,
        public ?string $externalSessionId,
        public string $correlationId,
    ) {}
}
```

#### LeaveGroupCommand

```php
final readonly class LeaveGroupCommand
{
    public function __construct(
        public string $groupId,
        public string $externalGroupId,
        public string $messengerType,
        public ?string $externalSessionId,
        public string $correlationId,
    ) {}
}
```

### Integration Events (входящие)

#### AuthCodeSentIntegrationEvent

```php
final readonly class AuthCodeSentIntegrationEvent
{
    public function __construct(
        public string $sessionId,
        public string $messengerType,
        public int $codeLength,
        public string $correlationId,
    ) {}
}
```

#### Auth2FARequiredIntegrationEvent

```php
final readonly class Auth2FARequiredIntegrationEvent
{
    public function __construct(
        public string $sessionId,
        public string $messengerType,
        public ?string $hint,
        public string $correlationId,
    ) {}
}
```

#### AuthSuccessIntegrationEvent

```php
final readonly class AuthSuccessIntegrationEvent
{
    public function __construct(
        public string $sessionId,
        public string $messengerType,
        public string $externalSessionId,
        public ?string $displayName,
        public string $correlationId,
    ) {}
}
```

#### AuthFailedIntegrationEvent

```php
final readonly class AuthFailedIntegrationEvent
{
    public function __construct(
        public string $sessionId,
        public string $messengerType,
        public string $reason,
        public string $message,
        public bool $canRetry,
        public string $correlationId,
    ) {}
}
```

#### SessionExpiredIntegrationEvent

```php
final readonly class SessionExpiredIntegrationEvent
{
    public function __construct(
        public string $sessionId,
        public string $messengerType,
        public string $reason,
    ) {}
}
```

#### GroupJoinedIntegrationEvent

```php
final readonly class GroupJoinedIntegrationEvent
{
    public function __construct(
        public string $groupId,
        public string $externalGroupId,
        public string $messengerType,
        public ?string $groupTitle,
        public ?int $memberCount,
        public string $correlationId,
    ) {}
}
```

#### GroupJoinFailedIntegrationEvent

```php
final readonly class GroupJoinFailedIntegrationEvent
{
    public function __construct(
        public string $groupId,
        public string $externalGroupId,
        public string $messengerType,
        public string $reason,
        public string $message,
        public string $correlationId,
    ) {}
}
```

#### MessageReceivedIntegrationEvent

```php
final readonly class MessageReceivedIntegrationEvent
{
    public function __construct(
        public string $messengerType,
        public string $externalGroupId,
        public string $messageId,
        public string $content,
        public ?string $senderName,
        public \DateTimeImmutable $sentAt,
        /** @var array<string, mixed> */
        public array $metadata = [],
    ) {}
}
```

### Использование в Application Layer

Application Services инжектят `CommandBusInterface`, не `MessageBusInterface`:

```php
// В SessionCoordinatorService (задача 0093):
class SessionCoordinatorService
{
    public function __construct(
        private MonitoringProfileRepositoryInterface $profileRepository,
        private CommandBusInterface $commandBus,  // ← Application абстракция
    ) {}

    public function startAuth(...): SessionDTO
    {
        // ...
        $this->commandBus->dispatch(new InitAuthCommand(
            sessionId: $session->getId()->toString(),
            messengerType: $messengerType->value,
            authData: $authData,
            correlationId: Uuid::v4()->toString(),
        ));
    }
}

// В AddPublicGroupHandler (задача 0094):
class AddPublicGroupHandler
{
    public function __construct(
        private MonitoringProfileRepositoryInterface $profileRepository,
        private BillingLimitsClientInterface $billingClient,
        private CommandBusInterface $commandBus,  // ← Application абстракция
    ) {}

    public function __invoke(AddPublicGroupCommand $command): GroupDTO
    {
        // ...
        $this->commandBus->dispatch(new JoinGroupCommand(
            groupId: $group->getId()->toString(),
            externalGroupId: $externalGroupId,
            messengerType: $messengerType->value,
            externalSessionId: null,
            correlationId: Uuid::v4()->toString(),
        ));
    }
}
```

### Безопасность

- `authData`, `code`, `password` — чувствительные данные, передаются через защищённый AMQP канал
- Чувствительные данные НЕ логируются при сериализации/десериализации
- `externalSessionId` в JoinGroupCommand — передаётся только для приватных групп (для публичных — null)

### Обработка ошибок

Настраивается на уровне Symfony Messenger:
- `failure_transport: failed` — уже настроен глобально
- Retry: стандартный Symfony Messenger retry с exponential backoff (3 attempts)
- Dead letter: failed transport для неудачных сообщений

## Тесты

### Integration Commands
- [x] InitAuthCommand — содержит все необходимые поля
- [x] SubmitAuthCodeCommand — содержит sessionId и code
- [x] SubmitPasswordCommand — содержит sessionId и password
- [x] StopSessionCommand — содержит sessionId и messengerType
- [x] JoinGroupCommand — формируется корректно (с externalSessionId и без)
- [x] LeaveGroupCommand — формируется корректно

### Integration Events
- [x] Все Integration Events десериализуются из payload корректно
- [x] AuthSuccessIntegrationEvent содержит externalSessionId и displayName
- [x] AuthFailedIntegrationEvent содержит reason, message, canRetry
- [x] MessageReceivedIntegrationEvent содержит content и metadata

### Конфигурация
- [x] Integration Commands роутятся в нужные транспорты
- [x] Integration Events обрабатываются `#[AsMessageHandler]` handlers (задача 0095)
- [x] Failed messages попадают в failed transport

## Зависимости

- Shared Application Layer — EventBusInterface (как паттерн-образец)
- Symfony Messenger (framework — только в Infrastructure)
- Event Handlers (задача 0095) — обработка входящих Integration Events
- Message Processing (задача 0096) — обработка MessageReceivedIntegrationEvent

## Definition of Done

- [x] `CommandBusInterface` создан в `Shared/Application/Command/`
- [x] `SymfonyCommandBus` реализован в `Shared/Infrastructure/Command/`
- [x] Все Integration Command DTO реализованы
- [x] Все Integration Event DTO реализованы
- [x] Транспорты настроены в messenger.yaml
- [x] Routing настроен для всех команд и событий
- [x] Retry и failure стратегии используют стандартный Symfony Messenger
- [x] Unit-тесты написаны и проходят
- [x] Код соответствует PSR-12
- [ ] PHPStan level 9 без ошибок
