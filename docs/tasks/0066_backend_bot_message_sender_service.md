# Задача 0066: MessageSender Service

## Контекст

MessageSender — сервис отправки сообщений в мессенджеры. Абстрагирует выбор адаптера и обработку ошибок доставки.

## Цель

Создать сервис `MessageSender` для отправки сообщений.

## Спецификация

- [MessageSender](../backend/bot/services/message-sender.md)

## Файлы для создания

```
src/Context/Bot/Application/Service/MessageSender.php
src/Context/Bot/Application/Service/MessageSenderInterface.php
tests/Unit/Context/Bot/Application/Service/MessageSenderTest.php
```

## Важно

1. **Публичный метод только один** — `sendToUser(Messenger, UserId, BotResponse)`
2. **send и sendToEndpoint приватные** — внутренняя реализация
3. **Вся отправка с указанием типа мессенджера и userId**

## Требования

### MessageSenderInterface

```php
<?php

declare(strict_types=1);

namespace App\Context\Bot\Application\Service;

use App\Context\Bot\Domain\Exception\EndpointNotFoundException;
use App\Context\Bot\Domain\Exception\MessageDeliveryException;
use App\Context\Bot\Domain\Model\BotResponse;
use App\Context\Shared\Domain\Enum\Messenger;
use App\Context\Shared\Domain\ValueObject\UserId;

interface MessageSenderInterface
{
    /**
     * Отправляет сообщение пользователю в конкретный мессенджер.
     *
     * @throws EndpointNotFoundException если endpoint не найден
     * @throws MessageDeliveryException при ошибке доставки
     */
    public function sendToUser(
        Messenger $messenger,
        UserId $userId,
        BotResponse $response,
    ): void;
}
```

### MessageSender

```php
<?php

declare(strict_types=1);

namespace App\Context\Bot\Application\Service;

use App\Context\Bot\Application\Adapter\MessengerAdapterInterface;
use App\Context\Bot\Domain\Exception\EndpointNotFoundException;
use App\Context\Bot\Domain\Exception\MessageDeliveryException;
use App\Context\Bot\Domain\Model\BotResponse;
use App\Context\Bot\Domain\Model\NotificationEndpoint;
use App\Context\Bot\Domain\Repository\NotificationEndpointRepositoryInterface;
use App\Context\Shared\Application\EventBusInterface;
use App\Context\Shared\Domain\Enum\Messenger;
use App\Context\Shared\Domain\ValueObject\UserId;
use Psr\Log\LoggerInterface;

final class MessageSender implements MessageSenderInterface
{
    /**
     * @var array<string, MessengerAdapterInterface>
     */
    private array $adapters = [];

    /**
     * @param iterable<MessengerAdapterInterface> $adapters
     */
    public function __construct(
        iterable $adapters,
        private readonly NotificationEndpointRepositoryInterface $endpointRepository,
        private readonly EventBusInterface $eventBus,
        private readonly LoggerInterface $logger,
    ) {
        foreach ($adapters as $adapter) {
            $this->adapters[$adapter->getMessenger()->value] = $adapter;
        }
    }

    public function sendToUser(
        Messenger $messenger,
        UserId $userId,
        BotResponse $response,
    ): void {
        $endpoint = $this->endpointRepository->findByUserIdAndMessenger(
            $userId,
            $messenger,
        );

        if ($endpoint === null) {
            throw EndpointNotFoundException::forUser(
                $userId->toString(),
                $messenger->value,
            );
        }

        if (!$endpoint->isActive()) {
            throw EndpointNotFoundException::forUser(
                $userId->toString(),
                $messenger->value,
            );
        }

        $this->sendToEndpoint($endpoint, $response);
    }

    private function sendToEndpoint(
        NotificationEndpoint $endpoint,
        BotResponse $response,
    ): void {
        $adapter = $this->getAdapter($endpoint->getMessenger());

        try {
            $this->send(
                $adapter,
                $endpoint->getExternalTargetId(),
                $response,
            );

            $this->logger->debug('Message sent', [
                'endpoint_id' => $endpoint->getId()->toString(),
                'messenger' => $endpoint->getMessenger()->value,
            ]);

        } catch (MessageDeliveryException $e) {
            $this->logger->error('Message delivery failed', [
                'endpoint_id' => $endpoint->getId()->toString(),
                'error' => $e->getMessage(),
            ]);

            if ($e->shouldRevokeEndpoint()) {
                $endpoint->revoke();
                $this->endpointRepository->save($endpoint);

                foreach ($endpoint->pullEvents() as $event) {
                    $this->eventBus->publish($event);
                }
            }

            throw $e;
        }
    }

    private function send(
        MessengerAdapterInterface $adapter,
        string $chatId,
        BotResponse $response,
    ): void {
        $adapter->sendMessage(
            $chatId,
            $response->getMessage(),
            $response->getKeyboard(),
        );
    }

    private function getAdapter(Messenger $messenger): MessengerAdapterInterface
    {
        if (!isset($this->adapters[$messenger->value])) {
            throw new \RuntimeException(
                sprintf('No adapter registered for messenger "%s"', $messenger->value),
            );
        }

        return $this->adapters[$messenger->value];
    }
}
```

### MessengerAdapterFactory

Также понадобится фабрика адаптеров (из спецификации):

```php
<?php

declare(strict_types=1);

namespace App\Context\Bot\Application\Service;

use App\Context\Bot\Application\Adapter\MessengerAdapterInterface;
use App\Context\Shared\Domain\Enum\Messenger;

final class MessengerAdapterFactory
{
    /** @var array<string, MessengerAdapterInterface> */
    private array $adapters = [];

    public function register(Messenger $messenger, MessengerAdapterInterface $adapter): void
    {
        $this->adapters[$messenger->value] = $adapter;
    }

    public function create(Messenger $messenger): MessengerAdapterInterface
    {
        if (!isset($this->adapters[$messenger->value])) {
            throw new \RuntimeException(
                sprintf('No adapter registered for messenger "%s"', $messenger->value),
            );
        }

        return $this->adapters[$messenger->value];
    }

    public function has(Messenger $messenger): bool
    {
        return isset($this->adapters[$messenger->value]);
    }
}
```

## Symfony Integration

```yaml
# config/services.yaml
services:
    _instanceof:
        App\Context\Bot\Application\Adapter\MessengerAdapterInterface:
            tags: ['bot.messenger_adapter']

    App\Context\Bot\Application\Service\MessageSender:
        arguments:
            $adapters: !tagged_iterator bot.messenger_adapter
```

## Тесты

- [x] sendToUser() находит endpoint по userId и messenger
- [x] sendToUser() выбрасывает EndpointNotFoundException если нет endpoint
- [x] sendToUser() выбрасывает EndpointNotFoundException если endpoint не активен
- [x] sendToUser() выбирает правильный адаптер
- [x] sendToUser() логирует отправку
- [x] sendToEndpoint() блокирует endpoint при ошибке доставки
- [x] sendToEndpoint() публикует события при блокировке
- [x] getAdapter() выбрасывает исключение для неподдерживаемого мессенджера

## Зависимости

- `MessengerAdapterInterface` (задача 0063)
- `NotificationEndpointRepositoryInterface` (задача 0059)
- `NotificationEndpoint` (задача 0055)
- `BotResponse` (задача 0054)
- `MessageDeliveryException` (задача 0057)
- `EndpointNotFoundException` (задача 0057)
- `EventBusInterface` (Shared)

## Definition of Done

- [x] MessageSenderInterface создан
- [x] Класс MessageSender создан
- [x] MessengerAdapterFactory создан
- [ ] Symfony конфигурация добавлена
- [x] Unit-тесты написаны и проходят
- [x] Код соответствует PSR-12
- [x] PHPStan level 9 без ошибок
