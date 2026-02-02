# Задача 0063: MessengerAdapterInterface

## Контекст

MessengerAdapterInterface определяет контракт для интеграции с различными мессенджерами (Telegram, WhatsApp, Discord). Абстрагирует специфику API каждого мессенджера.

## Цель

Создать интерфейс адаптера мессенджера.

## Спецификация

- [MessengerAdapterInterface](../backend/bot/infrastructure/messenger-adapter-interface.md)

## Файлы для создания

```
src/Context/Bot/Application/Adapter/MessengerAdapterInterface.php
src/Context/Bot/Application/Adapter/SendResult.php
```

## Требования

### SendResult

```php
<?php

declare(strict_types=1);

namespace App\Context\Bot\Application\Adapter;

final readonly class SendResult
{
    private function __construct(
        private bool $success,
        private ?string $messageId,
        private ?string $error,
    ) {}

    public static function success(string $messageId): self
    {
        return new self(true, $messageId, null);
    }

    public static function failure(string $error): self
    {
        return new self(false, null, $error);
    }

    public function isSuccess(): bool
    {
        return $this->success;
    }

    public function getMessageId(): ?string
    {
        return $this->messageId;
    }

    public function getError(): ?string
    {
        return $this->error;
    }
}
```

### MessengerAdapterInterface

```php
<?php

declare(strict_types=1);

namespace App\Context\Bot\Application\Adapter;

use App\Context\Bot\Domain\Model\BotRequest;
use App\Context\Bot\Domain\Model\BotResponse;
use App\Context\Shared\Domain\Enum\Messenger;

interface MessengerAdapterInterface
{
    /**
     * Возвращает тип мессенджера, который обрабатывает адаптер
     */
    public function supports(): Messenger;

    /**
     * Отправляет сообщение в мессенджер
     *
     * @param string $externalTargetId Внешний идентификатор чата (chat_id)
     */
    public function send(string $externalTargetId, BotResponse $response): SendResult;

    /**
     * Парсит входящий запрос от мессенджера
     *
     * @param array<string, mixed> $payload Raw данные от webhook
     */
    public function parseRequest(array $payload): ?BotRequest;

    /**
     * Проверяет подпись входящего запроса (защита от подделки)
     *
     * @param array<string, mixed> $payload
     * @param array<string, string> $headers
     */
    public function verifySignature(array $payload, array $headers): bool;

    /**
     * Настраивает webhook для получения обновлений
     */
    public function setupWebhook(string $webhookUrl): bool;

    /**
     * Удаляет webhook
     */
    public function removeWebhook(): bool;

    /**
     * Проверяет статус webhook
     *
     * @return array{url: string, pending_update_count: int}|null
     */
    public function getWebhookInfo(): ?array;
}
```

## Методы

| Метод | Описание |
|-------|----------|
| `supports()` | Какой мессенджер обрабатывает адаптер |
| `send()` | Отправка BotResponse в мессенджер |
| `parseRequest()` | Парсинг webhook payload в BotRequest |
| `verifySignature()` | Проверка подписи webhook (безопасность) |
| `setupWebhook()` | Настройка webhook URL |
| `removeWebhook()` | Удаление webhook |
| `getWebhookInfo()` | Получение информации о webhook |

## Зависимости

- `BotRequest` (задача 0053)
- `BotResponse` (задача 0054)
- `App\Context\Shared\Domain\Enum\Messenger` (задача 0006)

## Definition of Done

- [x] SendResult создан
- [x] MessengerAdapterInterface создан
- [x] Все методы определены с корректными типами
- [x] PHPDoc комментарии добавлены
- [x] Код соответствует PSR-12
- [x] PHPStan level 9 без ошибок
