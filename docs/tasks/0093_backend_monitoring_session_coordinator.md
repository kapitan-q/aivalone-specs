# Задача 0093: SessionCoordinatorService и Auth Commands

## Контекст

`SessionCoordinatorService` координирует многошаговый процесс авторизации пользователя в мессенджере через auth-service. Процесс асинхронный: Backend отправляет команды в auth-service через MQ и получает события в ответ.

## Цель

Создать `SessionCoordinatorService`, Commands и Handlers для управления сессиями и процесса авторизации.

## Спецификация

- [Authorize Session Process](../backend/monitoring/processes/authorize-session.md)
- [Services Overview](../backend/monitoring/services/overview.md)
- [API Endpoints — Sessions](../backend/monitoring/api/endpoints.md)

## Файлы для создания

```
src/Context/Monitoring/Application/
├── Service/
│   └── SessionCoordinatorService.php
├── Command/
│   ├── StartAuth/
|   |   ├── StartAuthCommand.php
|   |   └── StartAuthHandler.php
│   ├── SubmitAuthCode/
|   |   ├── SubmitAuthCodeCommand.php
|   |   └── SubmitAuthCodeHandler.php
│   ├── Submit2FA/
|   |   ├── Submit2FACommand.php
|   |   └── Submit2FAHandler.php
│   └── StopSession/
|       ├── StopSessionCommand.php
|       └── StopSessionHandler.php
├── Query/
│   ├── GetSessions/
|   |   ├── GetSessionsQuery.php
|   |   └── GetSessionsHandler.php
│   └── GetSession/
|       ├── GetSessionQuery.php
|       └── GetSessionHandler.php
└── DTO/
    ├── SessionDTO.php
    └── SessionListDTO.php

tests/Unit/Context/Monitoring/Application/
├── Service/
│   └── SessionCoordinatorServiceTest.php
├── Command/
|   ├── StartAuth/
|   |   └── StartAuthHandlerTest.php
|   ├── SubmitAuthCode/
|   |   └── SubmitAuthCodeHandlerTest.php
|   ├── Submit2FA/
|   |   └── Submit2FAHandlerTest.php
|   └── StopSession/
|       └── StopSessionHandlerTest.php
└── Query/
    ├── GetSessions/
    |   └── GetSessionsHandler.php
    └── GetSession/
        └── GetSessionHandler.php
```

## Требования

### SessionCoordinatorService

```php
<?php

declare(strict_types=1);

namespace App\Context\Monitoring\Application\Service;

use App\Context\Monitoring\Domain\Model\MonitoringProfile;
use App\Context\Monitoring\Domain\Model\SessionId;
use App\Context\Monitoring\Domain\Repository\MonitoringProfileRepositoryInterface;
use App\Context\Shared\Application\Command\CommandBusInterface;

class SessionCoordinatorService
{
    public function __construct(
        private MonitoringProfileRepositoryInterface $profileRepository,
        private CommandBusInterface $commandBus,
    ) {}

    /**
     * Инициация авторизации.
     * Диспатчит InitAuthCommand (Integration) через MessageBus → monitoring_auth транспорт.
     */
    public function startAuth(
        MonitoringProfile $profile,
        Messenger $messengerType,
        array $authData,
    ): SessionDTO;

    /**
     * Отправка кода авторизации.
     * Проверяет состояние сессии и диспатчит SubmitAuthCodeCommand (Integration) через MessageBus.
     */
    public function submitAuthCode(
        MonitoringProfile $profile,
        SessionId $sessionId,
        string $code,
    ): SessionDTO;

    /**
     * Отправка 2FA пароля.
     */
    public function submit2FA(
        MonitoringProfile $profile,
        SessionId $sessionId,
        string $password,
    ): SessionDTO;

    /**
     * Остановка сессии.
     * Деактивирует сессию и диспатчит StopSessionCommand (Integration) через MessageBus.
     */
    public function stopSession(
        MonitoringProfile $profile,
        SessionId $sessionId,
    ): void;
}
```

### Commands

```php
final readonly class StartAuthCommand
{
    public function __construct(
        public UserId $userId,
        public Messenger $messengerType,
        /** @var array<string, string> */
        public array $authData,
    ) {}
}

final readonly class SubmitAuthCodeCommand
{
    public function __construct(
        public UserId $userId,
        public SessionId $sessionId,
        public string $code,
    ) {}
}

final readonly class Submit2FACommand
{
    public function __construct(
        public UserId $userId,
        public SessionId $sessionId,
        public string $password,
    ) {}
}

final readonly class StopSessionCommand
{
    public function __construct(
        public UserId $userId,
        public SessionId $sessionId,
    ) {}
}
```

### StartAuthHandler

```php
final class StartAuthHandler
{
    public function __construct(
        private MonitoringService $monitoringService,
        private SessionCoordinatorService $sessionCoordinator,
    ) {}

    public function __invoke(StartAuthCommand $command): SessionDTO
    {
        // 1. Получить профиль по userId
        // 2. Проверить активность
        // 3. Проверить нет ли уже сессии для этого мессенджера
        //    - Если AUTHORIZED — выбросить SessionAlreadyExistsException
        //    - Если isPending — вернуть текущий статус
        //    - Если FAILED/EXPIRED — можно начать заново
        // 4. Создать или получить существующую сессию
        // 5. sessionCoordinator.startAuth(profile, messenger, authData)
        // 6. Вернуть SessionDTO
    }
}
```

### DTO

```php
final readonly class SessionDTO
{
    public function __construct(
        public string $id,
        public string $messengerType,
        public string $status,
        public ?string $displayName,
        public bool $isActive,
        public int $groupsCount,
        public ?\DateTimeImmutable $lastActivityAt,
        public \DateTimeImmutable $createdAt,
    ) {}

    public static function fromDomain(MonitoringSession $session, int $groupsCount = 0): self;
}
```

### Безопасность

- `authData` (phoneNumber и т.д.) — передаётся только в auth-service, НЕ сохраняется в домене
- `code` и `password` — НЕ логируются, НЕ сохраняются
- `externalSessionId` — НЕ включается в DTO/API responses

## Тесты

### StartAuthHandler
- [x] Создание новой сессии для мессенджера
- [x] Повторный запрос при FAILED/EXPIRED — начинает заново
- [x] Исключение при уже авторизованной сессии
- [x] Возврат текущего статуса при pending сессии
- [ ] Исключение при неактивном профиле

### SubmitAuthCodeHandler
- [x] Отправка кода для сессии в статусе AUTHORIZING
- [x] Исключение при неверном статусе сессии

### Submit2FAHandler
- [x] Отправка 2FA пароля
- [x] Исключение при неверном статусе сессии

### StopSessionHandler
- [x] Деактивация активной сессии
- [x] Отправка StopSession команды в MQ
- [ ] Деактивация приватных групп при остановке сессии

### SessionCoordinatorService
- [x] startAuth() диспатчит InitAuthCommand (Integration) через commandBus
- [x] submitAuthCode() диспатчит SubmitAuthCodeCommand (Integration) через commandBus
- [x] submit2FA() диспатчит SubmitPasswordCommand (Integration) через commandBus
- [x] stopSession() диспатчит StopSessionCommand (Integration) через commandBus

## Зависимости

- MonitoringService (задача 0091)
- MonitoringSession Entity (задача 0084)
- MonitoringProfile AggregateRoot (задача 0087)
- CommandBusInterface (задача 0098) — Shared Application абстракция
- Integration Commands (задача 0098) — InitAuthCommand, SubmitAuthCodeCommand, SubmitPasswordCommand, StopSessionCommand
- Exceptions (задача 0081)
- Value Objects (задача 0077)

## Definition of Done

- [x] SessionCoordinatorService реализован
- [x] Все Commands и Handlers реализованы
- [x] Queries и QueryHandlers реализованы
- [x] SessionDTO реализован
- [x] Чувствительные данные не логируются и не сохраняются
- [x] Unit-тесты написаны и проходят
- [x] Код соответствует PSR-12
- [ ] PHPStan level 9 без ошибок
