# MessageSender Service Specification

## Назначение

`MessageSender` — сервис для отправки сообщений пользователям через соответствующие адаптеры мессенджеров.

## Интерфейс

```php
interface MessageSenderInterface
{
    /**
     * Отправляет сообщение пользователю
     *
     * @throws EndpointNotFoundException если endpoint не найден
     * @throws MessageDeliveryException при ошибке доставки
     */
    public function send(
        UserId $userId,
        Messenger $messenger,
        BotResponse $response
    ): void;

    /**
     * Отправляет сообщение через конкретный endpoint
     *
     * @throws MessageDeliveryException при ошибке доставки
     */
    public function sendToEndpoint(
        NotificationEndpoint $endpoint,
        BotResponse $response
    ): void;
}
```

## Реализация

```php
final class MessageSender implements MessageSenderInterface
{
    public function __construct(
        private NotificationEndpointRepositoryInterface $endpointRepository,
        private MessengerAdapterFactory $adapterFactory,
        private EventBusInterface $eventBus,
        private LoggerInterface $logger,
    ) {}

    public function send(
        UserId $userId,
        Messenger $messenger,
        BotResponse $response
    ): void {
        $endpoint = $this->endpointRepository->findByUserIdAndMessenger(
            $userId,
            $messenger
        );

        if ($endpoint === null) {
            throw EndpointNotFoundException::forUser(
                $userId->getValue(),
                $messenger->value
            );
        }

        if (!$endpoint->isActive()) {
            throw EndpointNotFoundException::forUser(
                $userId->getValue(),
                $messenger->value
            );
        }

        $this->sendToEndpoint($endpoint, $response);
    }

    public function sendToEndpoint(
        NotificationEndpoint $endpoint,
        BotResponse $response
    ): void {
        $adapter = $this->adapterFactory->create($endpoint->getMessenger());

        try {
            $adapter->sendMessage(
                $endpoint->getExternalTargetId(),
                $response->getMessage(),
                $response->getKeyboard()
            );

            $this->logger->debug('Message sent', [
                'endpoint_id' => $endpoint->getId()->getValue(),
                'messenger' => $endpoint->getMessenger()->value,
            ]);

        } catch (MessageDeliveryException $e) {
            $this->logger->error('Message delivery failed', [
                'endpoint_id' => $endpoint->getId()->getValue(),
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
}
```

## Фабрика адаптеров

```php
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
                sprintf('No adapter registered for messenger "%s"', $messenger->value)
            );
        }

        return $this->adapters[$messenger->value];
    }
}
```

## Связанные документы

* [Services Overview](overview.md)
* [MessengerAdapter Interface](../infrastructure/messenger-adapter-interface.md)
* [NotificationEndpoint](../models/notification-endpoint.md)
* [Send Message Process](../processes/send-message-to-user.md)

## Статус реализации

* [ ] Интерфейс MessageSenderInterface создан
* [ ] Класс MessageSender реализован
* [ ] MessengerAdapterFactory реализован
* [ ] Интеграция с репозиторием endpoint
* [ ] Обработка ошибок доставки
* [ ] Логирование реализовано
* [ ] Unit тесты написаны
