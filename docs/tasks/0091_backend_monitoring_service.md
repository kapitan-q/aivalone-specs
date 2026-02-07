# Задача 0091: MonitoringService (Application Service)

## Контекст

`MonitoringService` — основной Application Service контекста Monitoring. Координирует работу домена, проверяет бизнес-правила и взаимодействует с инфраструктурным слоем.

## Цель

Создать `MonitoringService` с базовыми методами управления профилем и `CreateMonitoringProfileCommand` + Handler.

## Спецификация

- [Services Overview](../backend/monitoring/services/overview.md)
- [Create Monitoring Profile](../backend/monitoring/processes/create-monitoring-profile.md)

## Файлы для создания

```
src/Context/Monitoring/Application/
├── Service/
│   └── MonitoringService.php
├── Command/
│   ├── CreateMonitoringProfile/
|   |   ├── CreateMonitoringProfileCommand.php
|   |   └── CreateMonitoringProfileHandler.php
│   ├── ActivateProfile/
|   |   ├── ActivateProfileCommand.php
|   |   └── ActivateProfileHandler.php
│   └── DeactivateProfile/
|       ├── DeactivateProfileCommand.php
|       └── DeactivateProfileHandler.php
├── Query/
│   ├── GetMonitoringProfile/
|   |   ├── GetMonitoringProfileQuery.php
|   |   └── GetMonitoringProfileHandler.php
│   └── GetMonitoringProfileByUserId/
|       ├── GetMonitoringProfileByUserIdQuery.php
|       └── GetMonitoringProfileByUserIdHandler.php
└── DTO/
    └── MonitoringProfileDTO.php

tests/Unit/Context/Monitoring/Application/
├── Command/
|   ├── CreateMonitoringProfile
│   |   └── CreateMonitoringProfileHandlerTest.php
│   ├── ActivateProfile/
│   |   └── ActivateProfileHandlerTest.php
│   └── DeactivateProfile/
│       └── DeactivateProfileHandlerTest.php
└── Query/
    ├── GetMonitoringProfile/
    |   └── GetMonitoringProfileHandlerTest.php
    └── GetMonitoringProfileByUserId
        └── GetMonitoringProfileByUserIdHandler.php
```

## Требования

### MonitoringService

```php
<?php

declare(strict_types=1);

namespace App\Context\Monitoring\Application\Service;

use App\Context\Account\Domain\Model\UserId;
use App\Context\Monitoring\Domain\Model\MonitoringProfile;
use App\Context\Monitoring\Domain\Model\MonitoringProfileId;
use App\Context\Monitoring\Domain\Repository\MonitoringProfileRepositoryInterface;

class MonitoringService
{
    public function __construct(
        private MonitoringProfileRepositoryInterface $profileRepository,
    ) {}

    public function createProfile(UserId $userId): MonitoringProfile
    {
        // Проверить что профиль не существует
        // Создать MonitoringProfile::create(userId)
        // Сохранить через репозиторий
        // Вернуть профиль
    }

    public function getProfile(MonitoringProfileId $profileId): MonitoringProfile
    {
        // Найти или выбросить ProfileNotFoundException
    }

    public function getProfileByUserId(UserId $userId): MonitoringProfile
    {
        // Найти или выбросить ProfileNotFoundException
    }

    public function activateProfile(MonitoringProfileId $profileId): void
    {
        // Получить профиль, активировать, сохранить
    }

    public function deactivateProfile(MonitoringProfileId $profileId): void
    {
        // Получить профиль, деактивировать, сохранить
    }
}
```

### CreateMonitoringProfileCommand

```php
final readonly class CreateMonitoringProfileCommand
{
    public function __construct(
        public UserId $userId,
    ) {}
}
```

### CreateMonitoringProfileHandler

```php
final class CreateMonitoringProfileHandler
{
    public function __construct(
        private MonitoringService $monitoringService,
    ) {}

    public function __invoke(CreateMonitoringProfileCommand $command): MonitoringProfileDTO
    {
        $profile = $this->monitoringService->createProfile($command->userId);
        return MonitoringProfileDTO::fromDomain($profile);
    }
}
```

### MonitoringProfileDTO

```php
final readonly class MonitoringProfileDTO
{
    public function __construct(
        public string $id,
        public string $userId,
        public bool $isActive,
        public int $filtersCount,
        public int $sessionsCount,
        public int $groupsCount,
        public \DateTimeImmutable $createdAt,
        public \DateTimeImmutable $updatedAt,
    ) {}

    public static function fromDomain(MonitoringProfile $profile): self;
}
```

### ActivateProfileCommand / DeactivateProfileCommand

```php
final readonly class ActivateProfileCommand
{
    public function __construct(
        public MonitoringProfileId $profileId,
    ) {}
}

final readonly class DeactivateProfileCommand
{
    public function __construct(
        public MonitoringProfileId $profileId,
    ) {}
}
```

## Тесты

- [x] createProfile() создаёт профиль для нового userId
- [x] createProfile() выбрасывает исключение при дублировании userId
- [x] getProfile() находит существующий профиль
- [x] getProfile() выбрасывает ProfileNotFoundException
- [x] getProfileByUserId() находит по userId
- [x] activateProfile() активирует деактивированный профиль
- [x] deactivateProfile() деактивирует активный профиль
- [x] CreateMonitoringProfileHandler корректно обрабатывает команду
- [x] ActivateProfileHandler корректно обрабатывает команду
- [x] DeactivateProfileHandler корректно обрабатывает команду
- [x] GetMonitoringProfileHandler возвращает DTO

## Зависимости

- MonitoringProfile AggregateRoot (задача 0087)
- MonitoringProfileRepositoryInterface (задача 0088)
- Exceptions (задача 0081)
- Value Objects (задача 0077)

## Definition of Done

- [x] MonitoringService реализован
- [x] Commands и Handlers реализованы
- [x] Queries и Handlers реализованы
- [x] MonitoringProfileDTO реализован
- [x] Unit-тесты написаны и проходят
- [x] Код соответствует PSR-12
- [ ] PHPStan level 9 без ошибок
