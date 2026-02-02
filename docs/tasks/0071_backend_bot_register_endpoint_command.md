# Задача 0071: RegisterEndpoint Command Handler

## Контекст

Command для регистрации нового NotificationEndpoint. Используется при первом взаимодействии пользователя с ботом (/start).

## Цель

Создать Command и Handler для регистрации endpoint.

## Спецификация

- [Register Endpoint Process](../backend/bot/processes/register-endpoint.md)

## Файлы для создания

```
src/Context/Bot/Application/Command/RegisterEndpoint/RegisterEndpointCommand.php
src/Context/Bot/Application/Command/RegisterEndpoint/RegisterEndpointHandler.php
tests/Unit/Context/Bot/Application/Command/RegisterEndpoint/RegisterEndpointHandlerTest.php
```

## Требования

### RegisterEndpointCommand

```php
<?php

declare(strict_types=1);

namespace App\Context\Bot\Application\Command\RegisterEndpoint;

use App\Context\Shared\Domain\Enum\Messenger;

final readonly class RegisterEndpointCommand
{
    public function __construct(
        public string $userId,
        public Messenger $messenger,
        public string $externalTargetId,
    ) {}
}
```

### RegisterEndpointHandler

```php
<?php

declare(strict_types=1);

namespace App\Context\Bot\Application\Command\RegisterEndpoint;

use App\Context\Bot\Domain\Model\NotificationEndpoint;
use App\Context\Bot\Domain\Repository\NotificationEndpointRepositoryInterface;
use App\Context\Shared\Domain\ValueObject\Uuid;
use Psr\Log\LoggerInterface;
use Symfony\Component\Messenger\Attribute\AsMessageHandler;

#[AsMessageHandler]
final class RegisterEndpointHandler
{
    public function __construct(
        private readonly NotificationEndpointRepositoryInterface $endpointRepository,
        private readonly LoggerInterface $logger,
    ) {}

    public function __invoke(RegisterEndpointCommand $command): string
    {
        $userId = Uuid::fromString($command->userId);

        // Проверяем, есть ли уже активный endpoint
        $existing = $this->endpointRepository->findActiveByUserAndMessenger(
            $userId,
            $command->messenger,
        );

        if ($existing !== null) {
            $this->logger->info('Endpoint already exists', [
                'user_id' => $command->userId,
                'messenger' => $command->messenger->value,
                'endpoint_id' => $existing->getId()->toString(),
            ]);

            return $existing->getId()->toString();
        }

        // Создаём новый endpoint
        $endpoint = NotificationEndpoint::register(
            userId: $userId,
            messenger: $command->messenger,
            externalTargetId: $command->externalTargetId,
        );

        $this->endpointRepository->save($endpoint);

        $this->logger->info('New endpoint registered', [
            'user_id' => $command->userId,
            'messenger' => $command->messenger->value,
            'endpoint_id' => $endpoint->getId()->toString(),
        ]);

        // TODO: Отправить события в EventBus
        // foreach ($endpoint->pullEvents() as $event) {
        //     $this->eventBus->dispatch($event);
        // }

        return $endpoint->getId()->toString();
    }
}
```

## Сценарии

1. **Новый пользователь** — создаётся endpoint, генерируется событие NotificationEndpointRegistered
2. **Существующий endpoint** — возвращается ID существующего, событие не генерируется
3. **Реактивация** — если старый endpoint был revoked, создаётся новый

## Тесты

- [ ] Handler создаёт новый endpoint если не существует
- [ ] Handler возвращает существующий endpoint ID если уже есть
- [ ] Handler сохраняет endpoint в репозиторий
- [ ] Handler логирует операции
- [ ] Событие NotificationEndpointRegistered генерируется при создании

## Зависимости

- `NotificationEndpoint` (задача 0055)
- `NotificationEndpointRepositoryInterface` (задача 0059)
- `App\Context\Shared\Domain\ValueObject\Uuid` (задача 0005)
- `App\Context\Shared\Domain\Enum\Messenger` (задача 0006)

## Definition of Done

- [x] Command класс создан
- [x] Handler реализован
- [x] Unit-тесты написаны и проходят
- [x] Код соответствует PSR-12
- [x] PHPStan level 9 без ошибок
