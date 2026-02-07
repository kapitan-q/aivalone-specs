# Задача 0088: MonitoringProfileRepository Interface

## Контекст

Monitoring Context использует единственный репозиторий для работы с агрегатом `MonitoringProfile`. Все дочерние сущности (Filter, Session, Group, Binding) сохраняются и загружаются вместе с агрегатом.

## Цель

Создать интерфейс `MonitoringProfileRepositoryInterface` в Domain Layer.

## Спецификация

- [MonitoringProfileRepository](../backend/monitoring/infrastructure/monitoring-profile-repository.md)

## Файлы для создания

```
src/Context/Monitoring/Domain/Repository/MonitoringProfileRepositoryInterface.php
```

## Требования

```php
<?php

declare(strict_types=1);

namespace App\Context\Monitoring\Domain\Repository;

use App\Context\Account\Domain\Model\UserId;
use App\Context\Monitoring\Domain\Model\GroupId;
use App\Context\Monitoring\Domain\Model\MonitoringProfile;
use App\Context\Monitoring\Domain\Model\MonitoringProfileId;
use App\Context\Monitoring\Domain\Model\SessionId;
use App\Context\Shared\Domain\Enum\Messenger;

interface MonitoringProfileRepositoryInterface
{
    public function findById(MonitoringProfileId $id): ?MonitoringProfile;

    public function findByUserId(UserId $userId): ?MonitoringProfile;

    public function existsByUserId(UserId $userId): bool;

    public function save(MonitoringProfile $profile): void;

    public function delete(MonitoringProfile $profile): void;

    /** Поиск по ID сессии (для обработки событий от auth-service) */
    public function findBySessionId(SessionId $sessionId): ?MonitoringProfile;

    /** Поиск по ID группы (для обработки событий от listener-service) */
    public function findByGroupId(GroupId $groupId): ?MonitoringProfile;

    /**
     * Поиск всех профилей, подписанных на группу по внешнему ID
     * (для обработки MessageReceived)
     *
     * @return MonitoringProfile[]
     */
    public function findByExternalGroupId(string $externalGroupId, Messenger $messengerType): array;
}
```

## Тесты

Интерфейс не требует unit-тестов. Тестирование реализации — в задаче 0089.

## Зависимости

- MonitoringProfile (задача 0087)
- Value Objects (задача 0077)
- `UserId` (Account Context) — задача 0017
- `Messenger` (Shared Context) — задача 0006

## Definition of Done

- [x] Интерфейс создан
- [x] Все методы определены с корректными типами
- [x] Код соответствует PSR-12
- [x] PHPStan level 9 без ошибок
