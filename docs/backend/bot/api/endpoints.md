# API Endpoints Bot Context

## Описание

Bot Context предоставляет webhook endpoints для приёма входящих сообщений от мессенджеров.

## Webhook Endpoints

### POST /api/bot/webhook/telegram

Webhook для получения обновлений от Telegram Bot API.

**Защита**: Валидация secret token.

**Request Body**:
```json
{
  "update_id": 123456789,
  "message": {
    "message_id": 1,
    "from": {
      "id": 123456789,
      "is_bot": false,
      "first_name": "John",
      "username": "john_doe"
    },
    "chat": {
      "id": 123456789,
      "first_name": "John",
      "username": "john_doe",
      "type": "private"
    },
    "date": 1234567890,
    "text": "/start"
  }
}
```

**Response**: `200 OK` (пустое тело)

**Обработка**:
1. Валидация source (secret token)
2. Парсинг через TelegramAdapter
3. Передача в Router::handle()
4. Отправка ответа через adapter
5. Возврат 200 OK

---

### POST /api/bot/webhook/whatsapp (Future)

Webhook для WhatsApp Business API.

---

### POST /api/bot/webhook/discord (Future)

Webhook для Discord Interactions.

## Контроллер

```php
#[Route('/api/bot/webhook')]
final class WebhookController
{
    public function __construct(
        private Router $router,
        private MessengerAdapterFactory $adapterFactory,
        private LoggerInterface $logger,
    ) {}

    #[Route('/telegram', methods: ['POST'])]
    public function telegram(Request $request): Response
    {
        try {
            $data = json_decode($request->getContent(), true);

            $adapter = $this->adapterFactory->create(Messenger::TELEGRAM);
            $botRequest = $adapter->parseWebhook($data);

            if ($botRequest === null) {
                // Неподдерживаемый тип update
                return new Response('', Response::HTTP_OK);
            }

            $response = $this->router->handle($botRequest);

            $adapter->sendMessage(
                $botRequest->getExternalUserId(),
                $botRequest->getMessenger(),
                $response->getMessage(),
                $response->getKeyboard(),
            );

            return new Response('', Response::HTTP_OK);

        } catch (\Throwable $e) {
            $this->logger->error('Webhook processing failed', [
                'messenger' => 'telegram',
                'error' => $e->getMessage(),
            ]);

            // Всегда возвращаем 200 чтобы Telegram не ретраил
            return new Response('', Response::HTTP_OK);
        }
    }
}
```

## Безопасность

### Валидация Telegram

```php
private function validateTelegramRequest(Request $request): bool
{
    // Вариант 1: Secret token в заголовке
    $secretToken = $request->headers->get('X-Telegram-Bot-Api-Secret-Token');
    if ($secretToken === $this->telegramSecretToken) {
        return true;
    }

    return false;
}
```

### Rate Limiting

- Лимит на количество запросов от одного chat_id
- Защита от flood атак

## CLI команды

```bash
# Установка webhook
bin/console bot:webhook:set telegram

# Удаление webhook
bin/console bot:webhook:delete telegram

# Информация о боте
bin/console bot:info telegram
```

## Связанные документы

* [Bot Context Overview](../overview.md)
* [Handle Incoming Message Process](../processes/handle-incoming-message.md)
* [TelegramAdapter](../infrastructure/telegram-adapter.md)

## Статус реализации

* [ ] WebhookController создан
* [ ] Telegram endpoint реализован
* [ ] Валидация запросов настроена
* [ ] Rate limiting настроен
* [ ] CLI команды созданы
* [ ] Integration тесты написаны
