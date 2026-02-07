# Задача 0092: Filter Management (Commands, Queries, Handlers)

## Контекст

Управление фильтрами — CRUD операции, привязка/отвязка фильтров к группам, активация/деактивация. Включает проверку лимитов через Billing.

## Цель

Создать Commands, Queries и Handlers для полного цикла управления фильтрами.

## Спецификация

- [Filter Management Process](../backend/monitoring/processes/filter-management.md)
- [Services Overview](../backend/monitoring/services/overview.md)
- [API Endpoints — Filters](../backend/monitoring/api/endpoints.md)

## Файлы для создания

```
src/Context/Monitoring/Application/
├── Command/
│   ├── CreateFilter/
|   │   ├── CreateFilterCommand.php
|   |   └── CreateFilterHandler.php
│   ├── UpdateFilter/
|   │   ├── UpdateFilterCommand.php
|   |   └── UpdateFilterHandler.php
│   ├── DeleteFilter/
|   │   ├── DeleteFilterCommand.php
|   |   └── DeleteFilterHandler.php
│   ├── ActivateFilter/
|   │   ├── ActivateFilterCommand.php
|   |   └── ActivateFilterHandler.php
│   ├── DeactivateFilter/
|   │   ├── DeactivateFilterCommand.php
|   |   └── DeactivateFilterHandler.php
│   ├── BindFilterToGroup/
|   │   ├── BindFilterToGroupCommand.php
|   |   └── BindFilterToGroupHandler.php
│   └── UnbindFilterFromGroup/
|       ├── UnbindFilterFromGroupCommand.php
|       └── UnbindFilterFromGroupHandler.php
├── Query/
│   ├── GetFilters/
|   │   ├── GetFiltersQuery.php
|   |   └── GetFiltersHandler.php
│   └── GetFilter/
|       ├── GetFilterQuery.php
|       └── GetFilterHandler.php
└── DTO/
    ├── FilterDTO.php
    └── FilterListDTO.php

tests/Unit/Context/Monitoring/Application/
├── Command/
|   ├── CreateFilter/
|   |   └── CreateFilterHandlerTest.php
|   ├── UpdateFilter/
|   |   └── UpdateFilterHandlerTest.php
|   ├── DeleteFilter/
|   |   └── DeleteFilterHandlerTest.php
|   ├── ActivateFilter/
|   |   └── ActivateFilterHandlerTest.php
|   ├── DeactivateFilter/
|   |   └── DeactivateFilterHandlerTest.php
|   ├── BindFilterToGroup/
|   |   └── BindFilterToGroupHandlerTest.php
|   └── UnbindFilterFromGroup/
|       └── UnbindFilterFromGroupHandlerTest.php
└── Query/
    ├── GetFilters/
    |   └── GetFiltersHandler.php
    └── GetFilter/
        └── GetFilterHandler.php
```

## Требования

### Commands

```php
final readonly class CreateFilterCommand
{
    public function __construct(
        public UserId $userId,
        public string $value,
        public FilterType $filterType,
        public ?string $name = null,
    ) {}
}

final readonly class UpdateFilterCommand
{
    public function __construct(
        public UserId $userId,
        public FilterId $filterId,
        public ?string $value = null,
        public ?string $name = null,
    ) {}
}

final readonly class DeleteFilterCommand
{
    public function __construct(
        public UserId $userId,
        public FilterId $filterId,
    ) {}
}

final readonly class ActivateFilterCommand
{
    public function __construct(
        public UserId $userId,
        public FilterId $filterId,
    ) {}
}

final readonly class DeactivateFilterCommand
{
    public function __construct(
        public UserId $userId,
        public FilterId $filterId,
    ) {}
}

final readonly class BindFilterToGroupCommand
{
    public function __construct(
        public UserId $userId,
        public FilterId $filterId,
        public GroupId $groupId,
    ) {}
}

final readonly class UnbindFilterFromGroupCommand
{
    public function __construct(
        public UserId $userId,
        public FilterId $filterId,
        public GroupId $groupId,
    ) {}
}
```

### CreateFilterHandler (пример)

```php
final class CreateFilterHandler
{
    public function __construct(
        private MonitoringProfileRepositoryInterface $profileRepository,
        private BillingLimitsClientInterface $billingClient,
    ) {}

    public function __invoke(CreateFilterCommand $command): FilterDTO
    {
        // 1. Получить профиль по userId
        // 2. Проверить активность профиля
        // 3. Проверить лимит фильтров через billingClient
        // 4. Проверить разрешённый тип фильтра
        // 5. Проверить дубликат
        // 6. profile.addFilter(value, filterType, name)
        // 7. Сохранить
        // 8. Вернуть FilterDTO
    }
}
```

### DTO

```php
final readonly class FilterDTO
{
    public function __construct(
        public string $id,
        public string $value,
        public string $filterType,
        public ?string $name,
        public bool $isActive,
        public int $groupsCount,
        public \DateTimeImmutable $createdAt,
        public \DateTimeImmutable $updatedAt,
    ) {}

    public static function fromDomain(Filter $filter, int $groupsCount = 0): self;
}

final readonly class FilterListDTO
{
    /** @param FilterDTO[] $filters */
    public function __construct(
        public array $filters,
        public int $total,
        public int $currentCount,
        public int $maxAllowed,
    ) {}
}
```

### Queries

```php
final readonly class GetFiltersQuery
{
    public function __construct(
        public UserId $userId,
        public ?bool $isActive = null,
        public ?FilterType $filterType = null,
    ) {}
}

final readonly class GetFilterQuery
{
    public function __construct(
        public UserId $userId,
        public FilterId $filterId,
    ) {}
}
```

## Тесты

### CreateFilterHandler
- [x] Создание keyword фильтра
- [x] Создание regex фильтра
- [x] Исключение при неактивном профиле
- [x] Исключение при превышении лимита фильтров
- [x] Исключение при недопустимом типе фильтра
- [x] Исключение при дубликате (value + filterType)
- [x] Исключение при невалидном regex

### UpdateFilterHandler
- [x] Обновление значения фильтра
- [x] Обновление названия фильтра
- [x] Исключение при несуществующем фильтре
- [ ] Исключение при дубликате нового значения

### DeleteFilterHandler
- [x] Удаление фильтра (с каскадным удалением привязок)
- [x] Исключение при несуществующем фильтре

### ActivateFilter / DeactivateFilter
- [x] Активация деактивированного фильтра
- [x] Деактивация активного фильтра
- [ ] Исключение при повторной активации/деактивации

### BindFilterToGroup / UnbindFilterFromGroup
- [x] Привязка фильтра к группе
- [x] Отвязка фильтра от группы
- [x] Исключение при дублировании привязки
- [x] Исключение при несуществующем фильтре или группе

## Зависимости

- MonitoringProfile AggregateRoot (задача 0087)
- Filter Entity (задача 0083)
- FilterGroupBinding (задача 0086)
- MonitoringProfileRepositoryInterface (задача 0088)
- BillingLimitsClientInterface (задача 0097)
- Exceptions (задача 0081)
- Value Objects (задача 0077)

## Definition of Done

- [x] Все Commands реализованы
- [x] Все Handlers реализованы
- [x] Queries и QueryHandlers реализованы
- [x] DTO реализованы
- [x] Интеграция с BillingLimitsClient для проверки лимитов
- [x] Unit-тесты написаны и проходят
- [x] Код соответствует PSR-12
- [ ] PHPStan level 9 без ошибок
