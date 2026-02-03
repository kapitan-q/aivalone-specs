# StartConversation Specification

## Назначение

`StartConversation` — диалог первичной инициализации пользователя. Вызывается командой `/start`.

## Особенности

- Это "reset" команда — сбрасывает текущий state
- Регистрирует NotificationEndpoint при первом вызове
- Генерирует событие BotUserConnected
- Возвращает WebApp кнопку для Telegram

## Реализация

```php
final class StartConversation extends AbstractConversation
{
    public function __construct(
        private NotificationEndpointRepositoryInterface $endpointRepository,
        private EventBusInterface $eventBus,
        private WebAppUrlGenerator $webAppUrlGenerator,
    ) {}

    public function getCode(): string
    {
        return 'start';
    }

    public function getDescription(): string
    {
        return 'Начало работы с ботом';
    }

    protected function getInitialStep(): string
    {
        return 'start';
    }

    protected function stepStart(?string $message, array $params): BotResponse
    {
        // Регистрация endpoint (если не существует)
        $this->registerEndpointIfNeeded();

        // Генерация события BotUserConnected
        $this->publishBotUserConnected();

        // Формирование приветственного сообщения
        return $this->buildWelcomeResponse();
    }

    private function registerEndpointIfNeeded(): void
    {
        $userId = $this->getContext('userId');
        $messenger = $this->getContext('messenger');
        $chatId = $this->getContext('chatId');

        $existing = $this->endpointRepository->findByUserIdAndMessenger(
            UserId::fromString($userId),
            Messenger::from($messenger)
        );

        if ($existing !== null && $existing->isActive()) {
            return;
        }

        $endpoint = NotificationEndpoint::create(
            UserId::fromString($userId),
            Messenger::from($messenger),
            $chatId
        );

        $this->endpointRepository->save($endpoint);

        foreach ($endpoint->pullEvents() as $event) {
            $this->eventBus->publish($event);
        }
    }

    private function publishBotUserConnected(): void
    {
        $userId = $this->getContext('userId');
        $messenger = $this->getContext('messenger');

        $this->eventBus->publish(new BotUserConnected(
            userId: $userId,
            messenger: $messenger,
            occurredAt: new DateTimeImmutable(),
        ));
    }

    private function buildWelcomeResponse(): BotResponse
    {
        $messenger = $this->getContext('messenger');
        $userId = $this->getContext('userId');

        $welcomeMessage = <<<TEXT
👋 Добро пожаловать в Aivalone!

Я помогу вам настроить мониторинг Telegram-каналов и получать уведомления о важных событиях.

🚀 Нажмите кнопку ниже, чтобы открыть приложение и начать работу.
TEXT;

        $keyboard = $this->buildKeyboard($messenger, $userId);

        return BotResponse::finishWithKeyboard($welcomeMessage, $keyboard);
    }

    private function buildKeyboard(string $messenger, string $userId): array
    {
        if ($messenger === 'telegram') {
            $webAppUrl = $this->webAppUrlGenerator->generate($userId);

            return [
                [['text' => '🚀 Открыть приложение', 'action' => 'web_app:'. $webAppUrl]],
                [['text' => '❓ Помощь', 'action' => '/help']],
            ];
        }

        // Для других мессенджеров — ссылка с токеном
        $authUrl = $this->webAppUrlGenerator->generateWithToken($userId);

        return [
            [['text' => '🚀 Открыть приложение', 'action' => 'url:'. $authUrl]],
            [['text' => '❓ Помощь', 'action' => '/help']],
        ];
    }
}
```

## Контекст диалога

Router устанавливает следующие данные в контекст перед вызовом:

| Ключ        | Описание                                  |
| ----------- | ----------------------------------------- |
| `userId`    | UUID пользователя из Account Context      |
| `messenger` | Код мессенджера (telegram, whatsapp, etc) |
| `chatId`    | Внешний ID чата (только для регистрации)  |

## Безопасность

- `chatId` используется только для регистрации endpoint
- `chatId` не включается в события
- После регистрации endpoint доступ к chatId через него

## Связанные документы

* [Conversations Overview](overview.md)
* [AbstractConversation](abstract-conversation.md)
* [Start Process](../processes/start-conversation.md)
* [NotificationEndpoint](../models/notification-endpoint.md)

## Статус реализации

* [ ] Класс StartConversation создан
* [ ] Регистрация endpoint реализована
* [ ] Событие BotUserConnected публикуется
* [ ] WebApp URL генерируется
* [ ] Telegram keyboard с web_app работает
* [ ] Fallback для других мессенджеров реализован
* [ ] Unit тесты написаны
