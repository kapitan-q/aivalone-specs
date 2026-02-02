# Задача 0067: TelegramAdapter (Nutgram)

## Контекст

TelegramAdapter реализует MessengerAdapterInterface для Telegram с использованием библиотеки Nutgram.

## Цель

Создать адаптер `TelegramAdapter` для интеграции с Telegram Bot API.

## Спецификация

- [TelegramAdapter](../backend/bot/infrastructure/telegram-adapter.md)

## Файлы для создания

```
src/Context/Bot/Infrastructure/Messenger/TelegramAdapter.php
tests/Unit/Context/Bot/Infrastructure/Messenger/TelegramAdapterTest.php
tests/Integration/Context/Bot/Infrastructure/Messenger/TelegramAdapterTest.php
```

## Важно

1. **BotRequest универсальный** — callback_query парсится в тот же формат что и текстовые команды
2. **Формат команды** — `/{conversationCode}:{stepCode}[:{param1=value1:param2=value2}]`
3. **Keyboard использует action** — `['text' => 'Label', 'action' => '/command:step']`
4. **Action преобразуется в нативный формат** — `/start` → callback_data, `url:...` → url, `web_app:...` → web_app

## Требования

### TelegramAdapter

```php
<?php

declare(strict_types=1);

namespace App\Context\Bot\Infrastructure\Messenger;

use App\Context\Bot\Application\Adapter\MessengerAdapterInterface;
use App\Context\Bot\Domain\Exception\MessageDeliveryException;
use App\Context\Bot\Domain\Model\BotRequest;
use App\Context\Bot\Domain\Model\CommandParser;
use App\Context\Shared\Domain\Enum\Messenger;
use Psr\Log\LoggerInterface;
use SergiX44\Nutgram\Nutgram;
use SergiX44\Nutgram\Telegram\Exceptions\TelegramException;
use SergiX44\Nutgram\Telegram\Types\Keyboard\InlineKeyboardButton;
use SergiX44\Nutgram\Telegram\Types\Keyboard\InlineKeyboardMarkup;
use SergiX44\Nutgram\Telegram\Types\Keyboard\WebAppInfo;

final class TelegramAdapter implements MessengerAdapterInterface
{
    public function __construct(
        private readonly Nutgram $bot,
        private readonly string $secretToken,
        private readonly LoggerInterface $logger,
    ) {}

    public function getMessenger(): Messenger
    {
        return Messenger::TELEGRAM;
    }

    /**
     * @param array<string, mixed> $data
     */
    public function parseWebhook(array $data): ?BotRequest
    {
        if (isset($data['callback_query'])) {
            return $this->parseCallbackQuery($data['callback_query']);
        }

        if (isset($data['message']['text'])) {
            return $this->parseMessage($data['message']);
        }

        return null;
    }

    /**
     * @param array<string, mixed> $callback
     */
    private function parseCallbackQuery(array $callback): BotRequest
    {
        $chatId = (string) $callback['message']['chat']['id'];
        $userId = (string) $callback['from']['id'];
        $data = $callback['data'] ?? '';

        // Callback data содержит команду в том же формате
        return $this->parseCommandString($userId, $chatId, $data);
    }

    /**
     * @param array<string, mixed> $message
     */
    private function parseMessage(array $message): BotRequest
    {
        $text = $message['text'] ?? '';
        $chatId = (string) $message['chat']['id'];
        $userId = (string) $message['from']['id'];

        // Если начинается с / — это команда
        if (str_starts_with($text, '/')) {
            return $this->parseCommandString($userId, $chatId, $text);
        }

        // Обычное сообщение
        return BotRequest::fromMessage(
            Messenger::TELEGRAM,
            $userId,
            $chatId,
            $text,
        );
    }

    private function parseCommandString(
        string $userId,
        string $chatId,
        string $input,
    ): BotRequest {
        $parsed = CommandParser::parse($input);

        if ($parsed === null) {
            return BotRequest::fromMessage(
                Messenger::TELEGRAM,
                $userId,
                $chatId,
                $input,
            );
        }

        return BotRequest::fromCommand(
            Messenger::TELEGRAM,
            $userId,
            $chatId,
            $parsed['command'],
            $parsed['step'],
            $parsed['params'],
        );
    }

    public function sendMessage(
        string $chatId,
        string $message,
        ?array $keyboard = null,
    ): void {
        try {
            $params = [
                'chat_id' => $chatId,
                'text' => $message,
                'parse_mode' => 'HTML',
            ];

            if ($keyboard !== null) {
                $params['reply_markup'] = $this->convertKeyboard($keyboard);
            }

            $this->bot->sendMessage(...$params);

            $this->logger->debug('Telegram message sent', ['chat_id' => '***']);

        } catch (TelegramException $e) {
            $this->handleException($e);
        }
    }

    /**
     * @param array<array<array{text: string, action: string}>> $keyboard
     */
    private function convertKeyboard(array $keyboard): InlineKeyboardMarkup
    {
        $rows = [];

        foreach ($keyboard as $row) {
            $buttons = [];

            foreach ($row as $button) {
                $text = $button['text'];
                $action = $button['action'];

                $buttons[] = $this->createButton($text, $action);
            }

            $rows[] = $buttons;
        }

        return InlineKeyboardMarkup::make($rows);
    }

    private function createButton(string $text, string $action): InlineKeyboardButton
    {
        // web_app:https://...
        if (str_starts_with($action, 'web_app:')) {
            $url = substr($action, 8);
            return InlineKeyboardButton::make(
                text: $text,
                web_app: new WebAppInfo($url),
            );
        }

        // url:https://...
        if (str_starts_with($action, 'url:')) {
            $url = substr($action, 4);
            return InlineKeyboardButton::make(
                text: $text,
                url: $url,
            );
        }

        // По умолчанию — callback_data (команда типа /start:step)
        return InlineKeyboardButton::make(
            text: $text,
            callback_data: $action,
        );
    }

    private function handleException(TelegramException $e): never
    {
        $code = $e->getCode();
        $message = $e->getMessage();

        throw match (true) {
            $code === 403 => MessageDeliveryException::botBlocked('telegram'),
            $code === 400 && str_contains($message, 'chat not found')
                => MessageDeliveryException::chatNotFound('telegram'),
            $code === 400 && str_contains($message, 'user is deactivated')
                => MessageDeliveryException::apiError('telegram', 'USER_DEACTIVATED', $message),
            $code === 429 => MessageDeliveryException::rateLimited('telegram'),
            default => MessageDeliveryException::apiError('telegram', (string) $code, $message),
        };
    }

    public function setWebhook(string $url): void
    {
        try {
            $this->bot->setWebhook($url, [
                'secret_token' => $this->secretToken,
                'allowed_updates' => ['message', 'callback_query', 'my_chat_member'],
                'drop_pending_updates' => true,
            ]);

            $this->logger->info('Telegram webhook set', ['url' => $url]);

        } catch (TelegramException $e) {
            throw new \RuntimeException(
                sprintf('Failed to set webhook: %s', $e->getMessage()),
                previous: $e,
            );
        }
    }

    public function deleteWebhook(): void
    {
        try {
            $this->bot->deleteWebhook();
            $this->logger->info('Telegram webhook deleted');
        } catch (TelegramException $e) {
            throw new \RuntimeException(
                sprintf('Failed to delete webhook: %s', $e->getMessage()),
                previous: $e,
            );
        }
    }

    public function checkHealth(): bool
    {
        try {
            $this->bot->getMe();
            return true;
        } catch (\Throwable) {
            return false;
        }
    }

    /**
     * @return array{id: string, username: string, name: string}
     */
    public function getBotInfo(): array
    {
        $me = $this->bot->getMe();

        return [
            'id' => (string) $me->id,
            'username' => $me->username ?? '',
            'name' => $me->first_name,
        ];
    }
}
```

## Конфигурация

```yaml
# config/packages/bot.yaml
parameters:
    telegram_bot_token: '%env(TELEGRAM_BOT_TOKEN)%'
    telegram_secret_token: '%env(TELEGRAM_SECRET_TOKEN)%'

services:
    SergiX44\Nutgram\Nutgram:
        arguments:
            $token: '%telegram_bot_token%'

    App\Context\Bot\Infrastructure\Messenger\TelegramAdapter:
        arguments:
            $secretToken: '%telegram_secret_token%'
```

## Environment Variables

```env
TELEGRAM_BOT_TOKEN=your_bot_token
TELEGRAM_SECRET_TOKEN=your_secret_for_webhook_validation
```

## Тесты

### Unit Tests (с моками)
- [x] getMessenger() возвращает TELEGRAM
- [x] parseWebhook() парсит текстовое сообщение
- [x] parseWebhook() парсит callback query в тот же формат
- [x] parseWebhook() парсит команду с step и params
- [x] parseWebhook() возвращает null для неизвестного типа
- [x] convertKeyboard() создаёт InlineKeyboardMarkup
- [x] createButton() создаёт callback_data для команд
- [x] createButton() создаёт web_app для web_app: action
- [x] createButton() создаёт url для url: action

### Integration Tests (с тестовым ботом)
- [ ] sendMessage() отправляет текстовое сообщение
- [ ] sendMessage() отправляет сообщение с кнопками
- [ ] sendMessage() отправляет сообщение с WebApp кнопкой
- [ ] setWebhook() настраивает webhook
- [ ] getBotInfo() возвращает информацию

## Зависимости

- `MessengerAdapterInterface` (задача 0063)
- `BotRequest` (задача 0053)
- `CommandParser` (задача 0053)
- `MessageDeliveryException` (задача 0057)
- `SergiX44\Nutgram\Nutgram` (composer: nutgram/nutgram)

## Definition of Done

- [x] Класс TelegramAdapter создан
- [ ] Конфигурация Symfony добавлена
- [x] Unit-тесты написаны и проходят
- [ ] Integration-тесты написаны (опционально)
- [x] Код соответствует PSR-12
- [x] PHPStan level 9 без ошибок
