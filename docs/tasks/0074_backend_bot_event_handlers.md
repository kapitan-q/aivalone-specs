# Задача 0074: Bot Event Handlers

## Контекст

Event Handlers для реакции на события из других контекстов. Bot Context подписывается на события Account и Billing контекстов.

## Цель

Создать Event Handlers для интеграции с другими контекстами.

## Файлы для создания

```
src/Context/Bot/Application/EventHandler/Account/OnUserRegistered.php
src/Context/Bot/Application/EventHandler/Account/OnUserDeleted.php
src/Context/Bot/Application/EventHandler/Billing/OnSubscriptionExpired.php
tests/Unit/Context/Bot/Application/EventHandler/BotEventHandlersTest.php
```

## Важно

1. **Namespace EventHandler** — не Event
2. **Структура по контексту-источнику** — Account, Billing
3. **Имена On{EventName}** — OnUserRegistered, OnUserDeleted, OnSubscriptionExpired

## Требования

### OnUserRegistered (Account)

Реагирует на регистрацию пользователя в Account Context.

```php
<?php

declare(strict_types=1);

namespace App\Context\Bot\Application\EventHandler\Account;

use App\Context\Account\Domain\Event\UserRegistered;
use Psr\Log\LoggerInterface;
use Symfony\Component\Messenger\Attribute\AsMessageHandler;

#[AsMessageHandler]
final class OnUserRegistered
{
    public function __construct(
        private readonly LoggerInterface $logger,
    ) {}

    public function __invoke(UserRegistered $event): void
    {
        // В MVP: просто логируем
        // В будущем: можно отправить приветственное сообщение
        $this->logger->info('User registered in Account Context', [
            'user_id' => $event->getUserId()->toString(),
        ]);
    }
}
```

### OnUserDeleted (Account)

Реагирует на удаление пользователя — отзывает все endpoints.

```php
<?php

declare(strict_types=1);

namespace App\Context\Bot\Application\EventHandler\Account;

use App\Context\Account\Domain\Event\UserDeleted;
use App\Context\Bot\Domain\Repository\ConversationStateRepositoryInterface;
use App\Context\Bot\Domain\Repository\NotificationEndpointRepositoryInterface;
use App\Context\Shared\Domain\ValueObject\UserId;
use Psr\Log\LoggerInterface;
use Symfony\Component\Messenger\Attribute\AsMessageHandler;

#[AsMessageHandler]
final class OnUserDeleted
{
    public function __construct(
        private readonly NotificationEndpointRepositoryInterface $endpointRepository,
        private readonly ConversationStateRepositoryInterface $stateRepository,
        private readonly LoggerInterface $logger,
    ) {}

    public function __invoke(UserDeleted $event): void
    {
        $userId = $event->getUserId();

        $this->logger->info('Processing user deletion in Bot Context', [
            'user_id' => $userId->toString(),
        ]);

        // 1. Отзываем все endpoints пользователя
        $endpoints = $this->endpointRepository->findActiveByUserId($userId);

        foreach ($endpoints as $endpoint) {
            try {
                $endpoint->revoke();
                $this->endpointRepository->save($endpoint);

                $this->logger->info('Endpoint revoked', [
                    'endpoint_id' => $endpoint->getId()->toString(),
                ]);
            } catch (\Throwable $e) {
                $this->logger->error('Failed to revoke endpoint', [
                    'endpoint_id' => $endpoint->getId()->toString(),
                    'error' => $e->getMessage(),
                ]);
            }
        }

        // 2. Удаляем все активные диалоги
        $this->stateRepository->removeAllByUser($userId);

        $this->logger->info('User data cleaned from Bot Context', [
            'user_id' => $userId->toString(),
            'endpoints_revoked' => count($endpoints),
        ]);
    }
}
```

### OnSubscriptionExpired (Billing)

Реагирует на истечение подписки — отправляет уведомление.

```php
<?php

declare(strict_types=1);

namespace App\Context\Bot\Application\EventHandler\Billing;

use App\Context\Billing\Domain\Event\SubscriptionExpired;
use App\Context\Bot\Application\Service\MessageSenderInterface;
use App\Context\Bot\Domain\Model\BotResponse;
use App\Context\Bot\Domain\Repository\NotificationEndpointRepositoryInterface;
use App\Context\Shared\Domain\Enum\Messenger;
use Psr\Log\LoggerInterface;
use Symfony\Component\Messenger\Attribute\AsMessageHandler;

#[AsMessageHandler]
final class OnSubscriptionExpired
{
    public function __construct(
        private readonly NotificationEndpointRepositoryInterface $endpointRepository,
        private readonly MessageSenderInterface $messageSender,
        private readonly LoggerInterface $logger,
        private readonly string $webAppUrl = 'https://app.aivalone.com',
    ) {}

    public function __invoke(SubscriptionExpired $event): void
    {
        $userId = $event->getUserId();

        $this->logger->info('Subscription expired, sending notification', [
            'user_id' => $userId->toString(),
            'subscription_id' => $event->getSubscriptionId()->toString(),
        ]);

        // Находим все активные endpoints
        $endpoints = $this->endpointRepository->findActiveByUserId($userId);

        if (count($endpoints) === 0) {
            $this->logger->warning('No active endpoints to notify about subscription expiration', [
                'user_id' => $userId->toString(),
            ]);
            return;
        }

        // Отправляем уведомление в каждый мессенджер
        foreach ($endpoints as $endpoint) {
            try {
                $this->messageSender->sendToUser(
                    $endpoint->getMessenger(),
                    $userId,
                    $this->buildNotificationResponse(),
                );
            } catch (\Throwable $e) {
                $this->logger->error('Failed to send subscription expiration notification', [
                    'user_id' => $userId->toString(),
                    'messenger' => $endpoint->getMessenger()->value,
                    'error' => $e->getMessage(),
                ]);
            }
        }
    }

    private function buildNotificationResponse(): BotResponse
    {
        $text = <<<'TEXT'
⚠️ Ваша подписка истекла

Для продолжения использования AIvalone продлите подписку.

Без активной подписки мониторинг приостановлен.
TEXT;

        return BotResponse::messageWithKeyboard($text, [
            [
                [
                    'text' => '💳 Продлить подписку',
                    'action' => 'web_app:' . $this->webAppUrl . '/billing',
                ],
            ],
        ]);
    }
}
```

## События для обработки

| Событие | Источник | Handler | Действие |
|---------|----------|---------|----------|
| `UserRegistered` | Account | `OnUserRegistered` | Логирование (MVP) |
| `UserDeleted` | Account | `OnUserDeleted` | Отзыв endpoints, очистка данных |
| `SubscriptionExpired` | Billing | `OnSubscriptionExpired` | Уведомление пользователя |

## Тесты

### OnUserRegistered
- [ ] Handler логирует событие

### OnUserDeleted
- [ ] Handler отзывает все endpoints пользователя
- [ ] Handler удаляет все conversation states
- [ ] Handler обрабатывает ошибки отзыва gracefully
- [ ] Handler логирует операции

### OnSubscriptionExpired
- [ ] Handler находит активные endpoints
- [ ] Handler отправляет уведомление через MessageSender
- [ ] Handler не падает если нет endpoints
- [ ] Handler обрабатывает ошибки отправки gracefully

## Зависимости

- `NotificationEndpointRepositoryInterface` (задача 0059)
- `ConversationStateRepositoryInterface` (задача 0060)
- `MessageSenderInterface` (задача 0066)
- События из Account Context (задача 0024)
- События из Billing Context (задача 0042)

## Definition of Done

- [ ] OnUserRegistered создан
- [ ] OnUserDeleted создан
- [ ] OnSubscriptionExpired создан
- [ ] Unit-тесты написаны и проходят
- [ ] Код соответствует PSR-12
- [ ] PHPStan level 9 без ошибок
