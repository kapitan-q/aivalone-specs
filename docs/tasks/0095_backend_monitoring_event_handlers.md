# Задача 0095: Event Handlers (Listener, Auth, External)

## Контекст

Monitoring Context обрабатывает события из нескольких источников:
1. **auth-service** — события авторизации (AuthCodeSent, AuthSuccess, Auth2FARequired, AuthFailed, SessionExpired)
2. **listener-service** — события присоединения к группам (GroupJoined, GroupJoinFailed)
3. **Другие контексты** — UserDeleted (Account), UserSubscriptionUpdated (Billing)

## Цель

Создать все Event Handlers для обработки внешних событий.

## Спецификация

- [Authorize Session Process](../backend/monitoring/processes/authorize-session.md)
- [Subscribe to Group Process](../backend/monitoring/processes/subscribe-to-group.md)
- [Services Overview](../backend/monitoring/services/overview.md)

## Файлы для создания

```
src/Context/Monitoring/Application/EventHandler/
├── Integration/
│   ├── AuthCodeSentHandler.php
│   ├── AuthSuccessHandler.php
│   ├── Auth2FARequiredHandler.php
│   ├── AuthFailedHandler.php
│   └── SessionExpiredHandler.php
├── Account/
│   └── UserDeletedHandler.php
├── Billing/
|   └── TariffChangedHandler.php
├── GroupJoinedHandler.php
└── GroupJoinFailedHandler.php

src/Context/Monitoring/Application/Event/
├── Integration/
|   ├── AuthCodeSentIntegrationEvent.php
|   ├── AuthSuccessIntegrationEvent.php
|   ├── Auth2FARequiredIntegrationEvent.php
|   ├── AuthFailedIntegrationEvent.php
|   ├── SessionExpiredIntegrationEvent.php
├── GroupJoinedIntegrationEvent.php
└── GroupJoinFailedIntegrationEvent.php

tests/Unit/Context/Monitoring/Application/EventHandler/
├── Integration/
│   ├── AuthCodeSentHandlerTest.php
│   ├── AuthSuccessHandlerTest.php
│   ├── Auth2FARequiredHandlerTest.php
│   ├── AuthFailedHandlerTest.php
│   └── SessionExpiredHandlerTest.php
├── Account/
│   └── UserDeletedHandlerTest.php
├── Billing/
|   └── TariffChangedHandlerTest.php
├── GroupJoinedHandlerTest.php
└── GroupJoinFailedHandlerTest.php
```

## Требования

### Auth Event Handlers

#### AuthCodeSentHandler

```php
final class AuthCodeSentHandler
{
    public function __construct(
        private MonitoringProfileRepositoryInterface $profileRepository,
        private MessageBusInterface $messageBus,
    ) {}

    public function __invoke(AuthCodeSentIntegrationEvent $event): void
    {
        // 1. Найти профиль по sessionId
        // 2. Если не найден — логировать и выйти
        // 3. Отправить уведомление пользователю через Bot
        //    NotifyAuthCodeSentCommand(userId, messengerType, codeLength)
    }
}
```

#### AuthSuccessHandler

```php
final class AuthSuccessHandler
{
    public function __invoke(AuthSuccessIntegrationEvent $event): void
    {
        // 1. Найти профиль по sessionId
        // 2. session.markAsAuthorized(externalSessionId, displayName)
        // 3. Сохранить профиль
        // 4. Уведомить пользователя через Bot
    }
}
```

#### Auth2FARequiredHandler

```php
final class Auth2FARequiredHandler
{
    public function __invoke(Auth2FARequiredIntegrationEvent $event): void
    {
        // 1. Найти профиль по sessionId
        // 2. Уведомить пользователя о необходимости 2FA
        //    (статус сессии остаётся AUTHORIZING)
    }
}
```

#### AuthFailedHandler

```php
final class AuthFailedHandler
{
    public function __invoke(AuthFailedIntegrationEvent $event): void
    {
        // 1. Найти профиль по sessionId
        // 2. session.markAsFailed(reason)
        // 3. Сохранить профиль
        // 4. Уведомить пользователя об ошибке (reason, canRetry)
    }
}
```

#### SessionExpiredHandler

```php
final class SessionExpiredHandler
{
    public function __invoke(SessionExpiredIntegrationEvent $event): void
    {
        // 1. Найти профиль по sessionId
        // 2. session.markAsExpired()
        // 3. Деактивировать все приватные группы этой сессии
        //    profile.deactivateGroupsBySession(sessionId)
        // 4. Сохранить профиль
        // 5. Уведомить пользователя
    }
}
```

### Listener Event Handlers

#### GroupJoinedHandler

```php
final class GroupJoinedHandler
{
    public function __invoke(GroupJoinedIntegrationEvent $event): void
    {
        // 1. Найти профиль по groupId
        // 2. Обновить groupTitle (если изменился)
        // 3. group.activate()  — status = ACTIVE
        // 4. Сохранить профиль
    }
}
```

#### GroupJoinFailedHandler

```php
final class GroupJoinFailedHandler
{
    public function __invoke(GroupJoinFailedIntegrationEvent $event): void
    {
        // 1. Найти профиль по groupId
        // 2. group.markAsFailed(reason)
        // 3. Сохранить профиль
        // 4. Уведомить пользователя через Bot
    }
}
```

### External Event Handlers

#### UserDeletedHandler

```php
final class UserDeletedHandler
{
    public function __invoke(UserDeletedEvent $event): void
    {
        // 1. Найти профиль по userId
        // 2. Если не найден — выйти (пользователь не использовал мониторинг)
        // 3. Деактивировать все активные сессии
        // 4. Отправить StopSession/LeaveGroup команды в MQ
        // 5. Удалить профиль через репозиторий
    }
}
```

#### TariffChangedHandler

```php
final class TariffChangedHandler
{
    public function __invoke(UserSubscriptionUpdatedEvent $event): void
    {
        // 1. Найти профиль по userId
        // 2. Получить новые лимиты через billingClient
        // 3. Проверить текущие показатели vs новые лимиты
        // 4. Если лимиты превышены — уведомить пользователя
        //    (деактивация лишних фильтров — post-MVP)
    }
}
```

### External Event DTO

```php
final readonly class AuthSuccessIntegrationEvent
{
    public function __construct(
        public string $sessionId,
        public string $messengerType,
        public string $externalSessionId,
        public ?string $displayName,
        public string $correlationId,
    ) {}
}

final readonly class GroupJoinedIntegrationEvent
{
    public function __construct(
        public string $groupId,
        public string $externalGroupId,
        public string $messengerType,
        public ?string $groupTitle,
        public ?int $memberCount,
        public string $correlationId,
    ) {}
}
```

## Тесты

### Auth Event Handlers
- [x] AuthCodeSentHandler — уведомление пользователя
- [x] AuthSuccessHandler — сессия переходит в AUTHORIZED
- [x] AuthSuccessHandler — сохраняет externalSessionId и displayName
- [x] Auth2FARequiredHandler — уведомление о 2FA
- [x] AuthFailedHandler — сессия переходит в FAILED
- [x] AuthFailedHandler — уведомление с reason и canRetry
- [x] SessionExpiredHandler — сессия переходит в EXPIRED
- [x] SessionExpiredHandler — приватные группы деактивируются
- [x] Все handlers — graceful при ненайденном профиле (log + return)

### Listener Event Handlers
- [x] GroupJoinedHandler — группа переходит в ACTIVE
- [x] GroupJoinedHandler — обновляет groupTitle
- [x] GroupJoinFailedHandler — группа переходит в FAILED
- [x] GroupJoinFailedHandler — уведомление пользователя

### External Event Handlers
- [x] UserDeletedHandler — удаление профиля со всеми зависимостями
- [x] UserDeletedHandler — диспатч StopSessionCommand/LeaveGroupCommand (Integration) через commandBus
- [x] UserDeletedHandler — graceful при отсутствии профиля
- [x] TariffChangedHandler — проверка лимитов
- [x] TariffChangedHandler — уведомление при превышении

## Зависимости

- MonitoringService (задача 0091)
- SessionCoordinatorService (задача 0093)
- MonitoringProfile AggregateRoot (задача 0087)
- MonitoringProfileRepositoryInterface (задача 0088)
- CommandBusInterface (задача 0098) — Shared Application абстракция
- Integration Commands (задача 0098) — StopSessionCommand, LeaveGroupCommand
- BillingLimitsClientInterface (задача 0097)
- Exceptions (задача 0081)
- Value Objects (задача 0077)

## Definition of Done

- [x] Все Auth Event Handlers реализованы
- [x] Все Listener Event Handlers реализованы
- [x] UserDeletedHandler реализован
- [x] TariffChangedHandler реализован
- [x] External Event DTO реализованы
- [x] Graceful handling при отсутствии профиля
- [x] Unit-тесты написаны и проходят
- [x] Код соответствует PSR-12
- [ ] PHPStan level 9 без ошибок
