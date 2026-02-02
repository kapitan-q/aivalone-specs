# Задача 0075: Bot Webhook Controller

## Контекст

HTTP Controller для приёма webhook запросов от мессенджеров. Универсальная точка входа для всех типов мессенджеров.

## Цель

Создать универсальный Controller для обработки webhook от любого мессенджера.

## Спецификация

- [API Endpoints](../backend/bot/api/endpoints.md)

## Файлы для создания

```
src/Context/Bot/Presentation/Http/Controller/BotWebhookController.php
tests/Integration/Context/Bot/Presentation/Http/BotWebhookControllerTest.php
```

## Важно

1. **Универсальный контроллер** — один endpoint для всех мессенджеров
2. **Параметр {messenger}** — telegram, whatsapp, discord
3. **Добавление нового мессенджера = добавление адаптера** — не нужно трогать контроллер

## Требования

### BotWebhookController

```php
<?php

declare(strict_types=1);

namespace App\Context\Bot\Presentation\Http\Controller;

use App\Context\Bot\Application\Command\HandleMessage\HandleIncomingMessageCommand;
use App\Context\Shared\Domain\Enum\Messenger;
use Psr\Log\LoggerInterface;
use Symfony\Bundle\FrameworkBundle\Controller\AbstractController;
use Symfony\Component\HttpFoundation\JsonResponse;
use Symfony\Component\HttpFoundation\Request;
use Symfony\Component\HttpFoundation\Response;
use Symfony\Component\Messenger\MessageBusInterface;
use Symfony\Component\Routing\Attribute\Route;

#[Route('/bot/webhook')]
final class BotWebhookController extends AbstractController
{
    public function __construct(
        private readonly MessageBusInterface $commandBus,
        private readonly LoggerInterface $logger,
    ) {}

    /**
     * Универсальный webhook endpoint для всех мессенджеров.
     */
    #[Route('/{messenger}', name: 'bot_webhook', methods: ['POST'])]
    public function webhook(Request $request, string $messenger): Response
    {
        $this->logger->debug('Webhook received', [
            'messenger' => $messenger,
            'content_length' => $request->headers->get('Content-Length'),
        ]);

        // Валидация messenger
        try {
            $messengerEnum = Messenger::from($messenger);
        } catch (\ValueError) {
            $this->logger->warning('Unknown messenger in webhook', [
                'messenger' => $messenger,
            ]);

            return new JsonResponse(['ok' => false, 'error' => 'Unknown messenger'], Response::HTTP_BAD_REQUEST);
        }

        // Парсим JSON payload
        try {
            $payload = json_decode(
                $request->getContent(),
                true,
                512,
                JSON_THROW_ON_ERROR,
            );
        } catch (\JsonException $e) {
            $this->logger->warning('Invalid JSON in webhook payload', [
                'messenger' => $messenger,
                'error' => $e->getMessage(),
            ]);

            return new JsonResponse(['ok' => false], Response::HTTP_BAD_REQUEST);
        }

        // Собираем заголовки для проверки подписи
        $headers = [];
        foreach ($request->headers->all() as $key => $values) {
            $headers[strtolower($key)] = $values[0] ?? '';
        }

        // Отправляем команду на обработку
        try {
            $this->commandBus->dispatch(new HandleIncomingMessageCommand(
                messenger: $messengerEnum,
                payload: $payload,
                headers: $headers,
            ));
        } catch (\Throwable $e) {
            // Логируем, но возвращаем 200 — иначе мессенджер будет ретраить
            $this->logger->error('Failed to dispatch webhook command', [
                'messenger' => $messenger,
                'error' => $e->getMessage(),
            ]);
        }

        // Мессенджеры ожидают 200 OK
        return new JsonResponse(['ok' => true]);
    }

    /**
     * Health check endpoint для мониторинга.
     */
    #[Route('/{messenger}/health', name: 'bot_webhook_health', methods: ['GET'])]
    public function health(string $messenger): Response
    {
        try {
            $messengerEnum = Messenger::from($messenger);
        } catch (\ValueError) {
            return new JsonResponse([
                'status' => 'error',
                'error' => 'Unknown messenger',
            ], Response::HTTP_BAD_REQUEST);
        }

        return new JsonResponse([
            'status' => 'ok',
            'messenger' => $messengerEnum->value,
        ]);
    }
}
```

## Routing Configuration

```yaml
# config/routes/bot.yaml
bot_webhook:
    resource: '../src/Context/Bot/Presentation/Http/Controller/'
    type: attribute
```

## Endpoints

| Method | Path | Description |
|--------|------|-------------|
| POST | `/bot/webhook/{messenger}` | Универсальный webhook endpoint |
| GET | `/bot/webhook/{messenger}/health` | Health check |

### Примеры URL

- `POST /bot/webhook/telegram` — webhook для Telegram
- `POST /bot/webhook/whatsapp` — webhook для WhatsApp
- `GET /bot/webhook/telegram/health` — health check для Telegram

## Важные аспекты

### Безопасность

1. **Проверка подписи** — выполняется в Handler через адаптер мессенджера
2. **Rate limiting** — рекомендуется настроить на уровне nginx/load balancer
3. **HTTPS only** — мессенджеры отправляют webhook только на HTTPS

### Reliability

1. **Всегда возвращаем 200** — иначе мессенджер будет ретраить до 24 часов
2. **Async обработка** — через Symfony Messenger для reliability
3. **Logging** — логируем всё для отладки

### Telegram Requirements

- Webhook URL должен быть HTTPS
- Порт: 443, 80, 88, или 8443
- Сертификат должен быть валидным
- Ответ должен быть < 60 секунд

## Тесты

- [ ] POST /bot/webhook/telegram возвращает 200 для валидного payload
- [ ] POST /bot/webhook/telegram возвращает 400 для невалидного JSON
- [ ] POST /bot/webhook/telegram диспатчит HandleIncomingMessageCommand
- [ ] POST /bot/webhook/telegram возвращает 200 даже при ошибке обработки
- [ ] POST /bot/webhook/unknown возвращает 400 для неизвестного мессенджера
- [ ] GET /bot/webhook/telegram/health возвращает status ok
- [ ] Заголовки передаются в command

## Зависимости

- `HandleIncomingMessageCommand` (задача 0073)
- `App\Context\Shared\Domain\Enum\Messenger` (задача 0006)
- Symfony Messenger

## Definition of Done

- [x] Controller создан (универсальный для всех мессенджеров)
- [ ] Routing настроен
- [ ] Integration-тесты написаны и проходят
- [x] Код соответствует PSR-12
- [x] PHPStan level 9 без ошибок
