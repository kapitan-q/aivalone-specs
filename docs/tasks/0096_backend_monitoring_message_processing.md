# Задача 0096: Message Processing Pipeline

## Контекст

Pipeline обработки входящих сообщений от listener-service. Включает поиск профилей, подписанных на группу, получение активных фильтров, отправку в Filtering Context и обработку совпадений.

## Цель

Создать `MessageReceivedHandler` и `FilterMatchHandler` для полного цикла обработки сообщений.

## Спецификация

- [Message Processing Process](../backend/monitoring/processes/message-processing.md)
- [Events Overview](../backend/monitoring/events/overview.md)

## Файлы для создания

```
src/Context/Monitoring/Application/
├── EventHandler/
|   ├── Integration
│   |   └── MessageReceivedHandler.php
|   └── Filtering
│       └── FilterMatchFoundHandler.php
└── Event/
    └── Integration/
        └── MessageReceivedIntegrationEvent.php

src/Context/Filtering/Application/
├── Command
|   └── FilterMessage
|       └── FilterMessageCommand.php
├── Event/
│   └── FilterMatchFoundEvent.php
└── DTO/
    └── FilterMatchNotificationDTO.php

tests/Unit/Context/Monitoring/Application/EventHandler/
├── Integration
|   └── MessageReceivedHandlerTest.php
└── Filtering
    └── FilterMatchFoundHandlerTest.php
```

## Требования

### MessageReceivedIntegrationEvent

```php
final readonly class MessageReceivedIntegrationEvent
{
    public function __construct(
        public string $messengerType,
        public string $externalGroupId,
        public string $messageId,
        public string $content,
        public ?string $senderName,
        public \DateTimeImmutable $sentAt,
        /** @var array<string, mixed> */
        public array $metadata = [],
    ) {}
}
```

### MessageReceivedHandler

```php
final class MessageReceivedHandler
{
    public function __construct(
        private MonitoringProfileRepositoryInterface $profileRepository,
        private MessageBusInterface $messageBus,
    ) {}

    public function __invoke(MessageReceivedIntegrationEvent $event): void
    {
        // 1. Найти все профили, подписанные на группу
        //    profileRepository.findByExternalGroupId(externalGroupId, messengerType)
        // 2. Если нет профилей — выйти
        // 3. Для каждого профиля:
        //    a. Найти группу в профиле по externalGroupId
        //    b. Проверить что группа активна
        //    c. Обновить lastMessageAt — group.recordMessageReceived()
        //    d. Получить активные фильтры для группы
        //       profile.getActiveFiltersForGroup(groupId)
        //    e. Если нет фильтров — пропустить
        //    f. Отправить FilterMessageCommand в Filtering Context
        //       с messageId, profileId, groupId, content, filters[]
        // 4. Сохранить профили (для обновления lastMessageAt)
    }
}
```

### FilterMatchFoundEvent

```php
final readonly class FilterMatchFoundEvent
{
    public function __construct(
        public string $messageId,
        public string $profileId,
        public string $groupId,
        /** @var string[] */
        public array $matchedFilterIds,
        /** @var array<array{filterId: string, filterValue: string, positions: array}> */
        public array $matchDetails,
        public string $content,
    ) {}
}
```

### FilterMatchFoundHandler

```php
final class FilterMatchFoundHandler
{
    public function __construct(
        private MonitoringProfileRepositoryInterface $profileRepository,
        private BillingLimitsClientInterface $billingClient,
        private MessageBusInterface $messageBus,
    ) {}

    public function __invoke(FilterMatchFoundExternalEvent $event): void
    {
        // 1. Получить профиль по profileId
        // 2. Получить группу и совпавшие фильтры
        // 3. Проверить лимит уведомлений (опционально, post-MVP)
        // 4. Отправить SendFilterMatchNotificationCommand в Bot Context
        //    с userId, groupTitle, messengerType, matchedFilters, messagePreview
        // 5. Записать событие FilterMatchNotificationSent
        // 6. Сохранить профиль
    }
}
```

### FilterMessageCommand (отправляется в Filtering Context)

```php
final readonly class FilterMessageCommand
{
    public function __construct(
        public string $messageId,
        public string $profileId,
        public string $groupId,
        public string $content,
        /** @var array<array{filterId: string, value: string, filterType: string}> */
        public array $filters,
    ) {}
}
```

### SendFilterMatchNotificationCommand (отправляется в Bot Context)

```php
final readonly class SendFilterMatchNotificationCommand
{
    public function __construct(
        public string $userId,
        public string $groupTitle,
        public string $messengerType,
        /** @var string[] */
        public array $matchedFilterValues,
        public string $messagePreview,
        public string $messageId,
    ) {}
}
```

## Тесты

### MessageReceivedHandler
- [x] Находит все профили, подписанные на группу
- [x] Обрабатывает каждый профиль — получает фильтры для группы
- [x] Отправляет FilterMessageCommand с корректными фильтрами
- [x] Обновляет lastMessageAt на группе
- [x] Пропускает профили без активных фильтров
- [x] Пропускает неактивные группы
- [x] Graceful при отсутствии профилей (log + return)

### FilterMatchFoundHandler
- [x] Отправляет уведомление в Bot Context при совпадении
- [x] Формирует корректный messagePreview (truncate до 200 символов)
- [ ] Записывает событие FilterMatchNotificationSent (post-MVP)
- [x] Graceful при отсутствии профиля
- [x] Пропускает если профиль не найден (log + return)

## Зависимости

- MonitoringProfile AggregateRoot (задача 0087)
- Filter Entity (задача 0083)
- MonitoredGroup Entity (задача 0085)
- FilterGroupBinding (задача 0086)
- MonitoringProfileRepositoryInterface (задача 0088)
- BillingLimitsClientInterface (задача 0097)
- Domain Events (задача 0082)

## Definition of Done

- [x] MessageReceivedHandler реализован
- [x] FilterMatchFoundHandler реализован
- [x] External Event DTO реализованы
- [x] Интеграция с Filtering Context (FilterMessageCommand)
- [x] Интеграция с Bot Context (SendFilterMatchNotificationCommand)
- [x] Graceful handling ошибок
- [x] Unit-тесты написаны и проходят
- [x] Код соответствует PSR-12
- [ ] PHPStan level 9 без ошибок
