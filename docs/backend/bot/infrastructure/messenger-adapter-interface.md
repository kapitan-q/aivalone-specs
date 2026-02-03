# MessengerAdapter Interface Specification

## Назначение

Интерфейс `MessengerAdapterInterface` определяет контракт для адаптеров мессенджеров. Каждый мессенджер (Telegram, WhatsApp, Discord) реализует этот интерфейс.

## Интерфейс

```php
interface MessengerAdapterInterface
{
    /**
     * Возвращает код мессенджера
     */
    public function getMessenger(): Messenger;

    /**
     * Парсит входящий webhook и создает BotRequest
     *
     * @throws InvalidWebhookException если данные невалидны
     */
    public function parseWebhook(array $data): ?BotRequest;

    /**
     * Отправляет сообщение
     *
     * @param string $chatId Внешний идентификатор чата
     * @param string $message Текст сообщения
     * @param array|null $keyboard Клавиатура (опционально)
     *
     * @throws MessageDeliveryException при ошибке доставки
     */
    public function sendMessage(
        string $chatId,
        string $message,
        ?array $keyboard = null
    ): void;

    /**
     * Проверяет доступность бота
     */
    public function checkHealth(): bool;

    /**
     * Устанавливает webhook URL
     *
     * @throws WebhookSetupException при ошибке установки
     */
    public function setWebhook(string $url): void;

    /**
     * Удаляет webhook
     */
    public function deleteWebhook(): void;

    /**
     * Возвращает информацию о боте
     * @return array{id: string, username: string, name: string}
     */
    public function getBotInfo(): array;
}
```

## Конвертация keyboard

Адаптер конвертирует универсальный формат keyboard в специфичный для мессенджера:

```php
// Универсальный формат (из BotResponse)
$keyboard = [
    [
        ['text' => 'Кнопка 1', 'action' => '/setup:lang'],
        ['text' => 'Кнопка 2', 'action' => '/setup:filters'],
    ],
    [
        ['text' => '🌐 Открыть', 'action' => 'web_app:https://...'],
    ],
];

// Telegram формат (InlineKeyboardMarkup)
$telegramKeyboard = [
    'inline_keyboard' => [
        [
            ['text' => 'Кнопка 1', 'callback_data' => '/setup:lang'],
            ['text' => 'Кнопка 2', 'callback_data' => '/setup:filters'],
        ],
        [
            ['text' => '🌐 Открыть', 'web_app' => ['url' => 'https://...']],
        ],
    ],
];
```

## Обработка ошибок

Адаптер должен преобразовывать специфичные ошибки API в `MessageDeliveryException`:

```php
try {
    $this->api->sendMessage($chatId, $message);
} catch (TelegramException $e) {
    throw match ($e->getCode()) {
        403 => MessageDeliveryException::botBlocked($this->getMessenger()->value),
        400 when str_contains($e->getMessage(), 'chat not found')
            => MessageDeliveryException::chatNotFound($this->getMessenger()->value),
        429 => MessageDeliveryException::rateLimited($this->getMessenger()->value),
        default => MessageDeliveryException::apiError(
            $this->getMessenger()->value,
            (string) $e->getCode(),
            $e->getMessage()
        ),
    };
}
```

## Связанные документы

* [Infrastructure Overview](overview.md)
* [Telegram Adapter](telegram-adapter.md)
* [MessageSender Service](../services/message-sender.md)

## Статус реализации

* [ ] Интерфейс MessengerAdapterInterface создан
* [ ] Telegram Adapter реализован
* [ ] WhatsApp Adapter (future)
* [ ] Discord Adapter (future)
