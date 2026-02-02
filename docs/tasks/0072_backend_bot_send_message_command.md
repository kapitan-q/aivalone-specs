# Задача 0072: SendMessageToUser Command Handler

## Контекст

Command для отправки сообщения пользователю из других контекстов. Является публичным API Bot Context для исходящих сообщений.

## Цель

Создать Command и Handler для отправки сообщений пользователю.

## Спецификация

- [Send Message To User Process](../backend/bot/processes/send-message-to-user.md)

## Файлы для создания

```
src/Context/Bot/Application/Command/SendMessage/SendMessageToUserCommand.php
src/Context/Bot/Application/Command/SendMessage/SendMessageToUserHandler.php
tests/Unit/Context/Bot/Application/Command/SendMessage/SendMessageToUserHandlerTest.php
```

## Требования

### SendMessageToUserCommand

```php
<?php

declare(strict_types=1);

namespace App\Context\Bot\Application\Command\SendMessage;

final readonly class SendMessageToUserCommand
{
    /**
     * @param array<array<array{label: string, callback_data?: string, url?: string, web_app_url?: string}>> $keyboard
     */
    public function __construct(
        public string $userId,
        public string $text,
        public array $keyboard = [],
        public bool $parseMarkdown = true,
    ) {}
}
```

### SendMessageToUserHandler

```php
<?php

declare(strict_types=1);

namespace App\Context\Bot\Application\Command\SendMessage;

use App\Context\Bot\Application\Service\MessageSender;
use App\Context\Bot\Domain\Exception\EndpointNotFoundException;
use App\Context\Bot\Domain\Model\BotResponse;
use App\Context\Bot\Domain\Model\ResponseButton;
use App\Context\Bot\Domain\Repository\NotificationEndpointRepositoryInterface;
use App\Context\Shared\Domain\ValueObject\Uuid;
use Psr\Log\LoggerInterface;
use Symfony\Component\Messenger\Attribute\AsMessageHandler;

#[AsMessageHandler]
final class SendMessageToUserHandler
{
    public function __construct(
        private readonly NotificationEndpointRepositoryInterface $endpointRepository,
        private readonly MessageSender $messageSender,
        private readonly LoggerInterface $logger,
    ) {}

    public function __invoke(SendMessageToUserCommand $command): void
    {
        $userId = Uuid::fromString($command->userId);

        // Находим активные endpoints пользователя
        $endpoints = $this->endpointRepository->findActiveByUserId($userId);

        if (count($endpoints) === 0) {
            $this->logger->warning('No active endpoints for user', [
                'user_id' => $command->userId,
            ]);

            throw EndpointNotFoundException::byUserId($userId);
        }

        // Формируем ответ
        $response = $this->buildResponse($command);

        // В MVP отправляем в первый активный endpoint
        // В будущем: multi-endpoint, fallback
        $endpoint = $endpoints[0];

        try {
            $this->messageSender->sendToEndpoint($endpoint, $response);

            $this->logger->info('Message sent to user', [
                'user_id' => $command->userId,
                'endpoint_id' => $endpoint->getId()->toString(),
                'messenger' => $endpoint->getMessenger()->value,
            ]);
        } catch (\Throwable $e) {
            $this->logger->error('Failed to send message', [
                'user_id' => $command->userId,
                'error' => $e->getMessage(),
            ]);

            throw $e;
        }
    }

    private function buildResponse(SendMessageToUserCommand $command): BotResponse
    {
        if (count($command->keyboard) === 0) {
            return BotResponse::text($command->text, $command->parseMarkdown);
        }

        $keyboard = [];

        foreach ($command->keyboard as $row) {
            $buttonRow = [];

            foreach ($row as $buttonData) {
                $button = $this->createButton($buttonData);
                if ($button !== null) {
                    $buttonRow[] = $button;
                }
            }

            if (count($buttonRow) > 0) {
                $keyboard[] = $buttonRow;
            }
        }

        return BotResponse::withKeyboard(
            $command->text,
            $keyboard,
            $command->parseMarkdown,
        );
    }

    /**
     * @param array{label: string, callback_data?: string, url?: string, web_app_url?: string} $data
     */
    private function createButton(array $data): ?ResponseButton
    {
        if (isset($data['web_app_url'])) {
            return ResponseButton::webApp($data['label'], $data['web_app_url']);
        }

        if (isset($data['url'])) {
            return ResponseButton::url($data['label'], $data['url']);
        }

        if (isset($data['callback_data'])) {
            return ResponseButton::callback($data['label'], $data['callback_data']);
        }

        return null;
    }
}
```

## Пример использования

```php
// Из другого контекста (например, Monitoring)
$this->commandBus->dispatch(new SendMessageToUserCommand(
    userId: $userId,
    text: '🚨 **Обнаружен нежелательный контент!**\n\nПодробности в приложении.',
    keyboard: [
        [
            ['label' => '📱 Открыть', 'web_app_url' => 'https://app.aivalone.com/alerts/123'],
        ],
    ],
));
```

## Тесты

- [ ] Handler находит активные endpoints
- [ ] Handler выбрасывает исключение если нет endpoints
- [ ] Handler формирует BotResponse из command
- [ ] Handler отправляет сообщение через MessageSender
- [ ] Handler поддерживает кнопки всех типов (callback, url, web_app)
- [ ] Handler логирует успешную отправку
- [ ] Handler логирует и пробрасывает ошибки

## Зависимости

- `MessageSender` (задача 0066)
- `NotificationEndpointRepositoryInterface` (задача 0059)
- `BotResponse` (задача 0054)
- `ResponseButton` (задача 0054)
- `EndpointNotFoundException` (задача 0057)

## Definition of Done

- [x] Command класс создан
- [x] Handler реализован
- [x] Unit-тесты написаны и проходят
- [x] Код соответствует PSR-12
- [x] PHPStan level 9 без ошибок
