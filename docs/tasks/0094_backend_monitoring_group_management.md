# Задача 0094: Group Management (Commands, Queries, Handlers)

## Контекст

Управление группами — добавление публичных/приватных групп, удаление, получение списка. Включает проверку лимитов через Billing и координацию с listener-service через MQ.

## Цель

Создать Commands, Queries и Handlers для полного цикла управления группами.

## Спецификация

- [Subscribe to Group Process](../backend/monitoring/processes/subscribe-to-group.md)
- [Services Overview](../backend/monitoring/services/overview.md)
- [API Endpoints — Groups](../backend/monitoring/api/endpoints.md)

## Файлы для создания

```
src/Context/Monitoring/Application/
├── Command/
│   ├── AddPublicGroup/
|   |   ├── AddPublicGroupCommand.php
|   |   └── AddPublicGroupHandler.php
│   ├── AddPrivateGroup/
|   |   ├── AddPrivateGroupCommand.php
|   |   └── AddPrivateGroupHandler.php
│   └── RemoveGroup/
|       ├── RemoveGroupCommand.php
|       └── RemoveGroupHandler.php
├── Query/
│   ├── GetGroups/
|   |   ├── GetGroupsQuery.php
|   |   └── GetGroupsHandler.php
│   └── GetGroup/
|       ├── GetGroupQuery.php
|       └── GetGroupHandler.php
└── DTO/
    ├── GroupDTO.php
    └── GroupListDTO.php

tests/Unit/Context/Monitoring/Application/
├── Command/
│   ├── AddPublicGroup/
|   |   └── AddPublicGroupHandlerTest.php
│   ├── AddPrivateGroup/
|   |   └── AddPrivateGroupHandlerTest.php
│   └── RemoveGroup/
|       └── RemoveGroupHandlerTest.php
└── Query/
    ├── GetGroups/
    |   └── GetGroupsHandlerTest.php
    └── GetGroup/
        └── GetGroupHandlerTest.php
```

## Требования

### Commands

```php
final readonly class AddPublicGroupCommand
{
    public function __construct(
        public UserId $userId,
        public string $externalGroupId,
        public string $groupTitle,
        public Messenger $messengerType,
    ) {}
}

final readonly class AddPrivateGroupCommand
{
    public function __construct(
        public UserId $userId,
        public string $externalGroupId,
        public string $groupTitle,
        public Messenger $messengerType,
        public SessionId $sessionId,
    ) {}
}

final readonly class RemoveGroupCommand
{
    public function __construct(
        public UserId $userId,
        public GroupId $groupId,
    ) {}
}
```

### AddPublicGroupHandler

```php
final class AddPublicGroupHandler
{
    public function __construct(
        private MonitoringProfileRepositoryInterface $profileRepository,
        private BillingLimitsClientInterface $billingClient,
        private CommandBusInterface $commandBus,
    ) {}

    public function __invoke(AddPublicGroupCommand $command): GroupDTO
    {
        // 1. Получить профиль по userId
        // 2. Проверить активность
        // 3. Проверить лимит групп через billingClient
        // 4. Проверить дубликат (externalGroupId + messengerType)
        // 5. profile.addPublicGroup(externalGroupId, groupTitle, messengerType)
        //    group.status = PENDING, group.sessionId = null
        // 6. Сохранить
        // 7. Диспатчить JoinGroupCommand (Integration) через commandBus (externalSessionId=null)
        // 8. Вернуть GroupDTO
    }
}
```

### AddPrivateGroupHandler

```php
final class AddPrivateGroupHandler
{
    public function __construct(
        private MonitoringProfileRepositoryInterface $profileRepository,
        private BillingLimitsClientInterface $billingClient,
        private CommandBusInterface $commandBus,
    ) {}

    public function __invoke(AddPrivateGroupCommand $command): GroupDTO
    {
        // 1-4. Аналогично публичной группе
        // 5. Проверить сессию — существует, AUTHORIZED, messengerType совпадает
        // 6. profile.addPrivateGroup(externalGroupId, groupTitle, messengerType, sessionId)
        //    group.status = PENDING, group.sessionId = sessionId
        // 7. Сохранить
        // 8. Диспатчить JoinGroupCommand (Integration) через commandBus с externalSessionId
        // 9. Вернуть GroupDTO
    }
}
```

### RemoveGroupHandler

```php
final class RemoveGroupHandler
{
    public function __construct(
        private MonitoringProfileRepositoryInterface $profileRepository,
        private CommandBusInterface $commandBus,
    ) {}

    public function __invoke(RemoveGroupCommand $command): void
    {
        // 1. Получить профиль
        // 2. Получить группу
        // 3. Если группа активна — диспатчить LeaveGroupCommand (Integration) через commandBus
        // 4. profile.removeGroup(groupId) — каскадно удалит FilterGroupBindings
        // 5. Сохранить
    }
}
```

### DTO

```php
final readonly class GroupDTO
{
    public function __construct(
        public string $id,
        public string $externalGroupId,
        public string $groupTitle,
        public string $messengerType,
        public bool $isPrivate,
        public ?string $sessionId,
        public string $status,
        public int $filtersCount,
        public ?\DateTimeImmutable $lastMessageAt,
        public \DateTimeImmutable $createdAt,
    ) {}

    public static function fromDomain(MonitoredGroup $group, int $filtersCount = 0): self;
}

final readonly class GroupListDTO
{
    /** @param GroupDTO[] $groups */
    public function __construct(
        public array $groups,
        public int $total,
        public int $currentCount,
        public int $maxAllowed,
    ) {}
}
```

### Queries

```php
final readonly class GetGroupsQuery
{
    public function __construct(
        public UserId $userId,
        public ?Messenger $messengerType = null,
        public ?string $status = null,
        public ?bool $isPrivate = null,
    ) {}
}

final readonly class GetGroupQuery
{
    public function __construct(
        public UserId $userId,
        public GroupId $groupId,
    ) {}
}
```

## Тесты

### AddPublicGroupHandler
- [x] Добавление публичной группы (sessionId=null)
- [x] Диспатч JoinGroupCommand (Integration) через commandBus без externalSessionId
- [x] Исключение при превышении лимита групп
- [x] Исключение при дублировании (externalGroupId + messengerType)
- [ ] Исключение при неактивном профиле

### AddPrivateGroupHandler
- [x] Добавление приватной группы с авторизованной сессией
- [x] Диспатч JoinGroupCommand (Integration) через commandBus с externalSessionId
- [x] Исключение при неавторизованной сессии
- [ ] Исключение при несовпадении messengerType сессии и группы
- [ ] Исключение при несуществующей сессии
- [x] Исключение при превышении лимита групп

### RemoveGroupHandler
- [x] Удаление активной группы — диспатч LeaveGroupCommand (Integration)
- [x] Удаление неактивной группы — без отправки в MQ
- [ ] Каскадное удаление FilterGroupBindings
- [x] Исключение при несуществующей группе

## Зависимости

- MonitoringProfile AggregateRoot (задача 0087)
- MonitoredGroup Entity (задача 0085)
- MonitoringSession Entity (задача 0084)
- MonitoringProfileRepositoryInterface (задача 0088)
- BillingLimitsClientInterface (задача 0097)
- CommandBusInterface (задача 0098) — Shared Application абстракция
- Integration Commands (задача 0098) — JoinGroupCommand, LeaveGroupCommand
- Exceptions (задача 0081)
- Value Objects (задача 0077)

## Definition of Done

- [x] Все Commands и Handlers реализованы
- [x] Queries и QueryHandlers реализованы
- [x] DTO реализованы
- [x] Интеграция с BillingLimitsClient для проверки лимитов
- [x] Диспатч Integration Commands через CommandBusInterface
- [x] Unit-тесты написаны и проходят
- [x] Код соответствует PSR-12
- [ ] PHPStan level 9 без ошибок
