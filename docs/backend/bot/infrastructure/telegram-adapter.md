# Telegram Adapter Specification

## Назначение

`TelegramAdapter` — реализация `MessengerAdapterInterface` для Telegram Bot API. Использует библиотеку Nutgram.

## Зависимости

```php
final class TelegramAdapter implements MessengerAdapterInterface
{
    public function __construct(
        private Nutgram $bot,
        private LoggerInterface $logger,
    ) {}
}
```

## Реализация методов

### getMessenger()

```php
public function getMessenger(): Messenger
{
    return Messenger::TELEGRAM;
}
```

### parseWebhook()

```php
public function parseWebhook(array $data): ?BotRequest
{
    if (!isset($data['message']) && !isset($data['callback_query'])) {
        return null;
    }

    if (isset($data['callback_query'])) {
        return $this->parseCallbackQuery($data['callback_query']);
    }

    return $this->parseMessage($data['message']);
}

private function parseCallbackQuery(array $callback): BotRequest
{
    $chatId = (string) $callback['message']['chat']['id'];
    $userId = (string) $callback['from']['id'];
    $data = $callback['data'];

    return $this->parseCommand($userId, $chatId, $data)
}

private function parseMessage(array $message): BotRequest
{
    $text = $message['text'] ?? '';
    $chatId = (string) $message['chat']['id'];
    $userId = (string) $message['from']['id'];

    if (str_starts_with($text, '/')) {
        return $this->parseCommand($userId, $chatId, $text);
    }

    return new BotRequest::fromMessage(Messenger::TELEGRAM, $userId, $chatId, null, null, $text);
}

private function parseCommand(string $userId, $chatId, string $command): BotRequest
{
    $party = explode(':', $command);

    $command = array_shift($party);
    $step = array_shift($party) ?? null;
    $params = [];

    while(count($party) > 0) {
        $param = array_shift($party);
        [$paramName, $paramValue] = explode(':', $param);

        if ($paramName && $paramValue) {
            $params[$paramName] = $paramValue;
        }
    }

    return new BotRequest(Messenger::TELEGRAM, $userId, $chatId, $command, $step, null, $params);
}
```

### sendMessage()

```php
public function sendMessage(
    string $chatId,
    string $message,
    ?array $keyboard = null
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

private function convertKeyboard(array $keyboard): array
{
    // conver $keyboard to nutgram inline_keyboard
    
    return ['inline_keyboard' => $keyboard];
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
```

### setWebhook()

```php
public function setWebhook(string $url): void
{
    try {
        $this->bot->setWebhook($url, [
            'allowed_updates' => ['message', 'callback_query', 'my_chat_member'],
            'drop_pending_updates' => true,
        ]);

        $this->logger->info('Telegram webhook set', ['url' => $url]);

    } catch (TelegramException $e) {
        throw new WebhookSetupException(
            sprintf('Failed to set webhook: %s', $e->getMessage()),
            previous: $e
        );
    }
}
```

### checkHealth()

```php
public function checkHealth(): bool
{
    try {
        $this->bot->getMe();
        return true;
    } catch (\Throwable) {
        return false;
    }
}
```

### getBotInfo()

```php
public function getBotInfo(): array
{
    $me = $this->bot->getMe();

    return [
        'id' => (string) $me->id,
        'username' => $me->username,
        'name' => $me->first_name,
    ];
}
```

## Nutgram конфигурация

```yaml
# config/packages/nutgram.yaml
nutgram:
    token: '%env(TELEGRAM_BOT_TOKEN)%'
    config:
        api_url: 'https://api.telegram.org'
        timeout: 10
```

## CLI команды

```php
// bin/console bot:webhook:set telegram
// bin/console bot:webhook:delete telegram
// bin/console bot:webhook:info telegram
```

## Связанные документы

* [MessengerAdapter Interface](messenger-adapter-interface.md)
* [Infrastructure Overview](overview.md)

## Статус реализации

* [ ] TelegramAdapter класс создан
* [ ] parseWebhook реализован
* [ ] sendMessage реализован
* [ ] setWebhook реализован
* [ ] Nutgram интеграция настроена
* [ ] CLI команды созданы
* [ ] Error handling настроен
* [ ] Unit тесты написаны
* [ ] Integration тесты написаны
