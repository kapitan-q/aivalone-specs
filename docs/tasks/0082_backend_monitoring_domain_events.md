# Задача 0082: Monitoring Domain Events

## Контекст

Monitoring Context генерирует доменные события при изменении состояния профиля, фильтров, сессий и групп. События используются для координации с внешними сервисами (auth-service, listener-service, Bot Context).

## Цель

Создать все доменные события для Monitoring Context.

## Спецификация

- [Events Overview](../backend/monitoring/events/overview.md)
- [MonitoringSessionAuthorized](../backend/monitoring/events/monitoring-session-authorized.md)
- [GroupSubscriptionCreated](../backend/monitoring/events/group-subscription-created.md)

## Файлы для создания

```
src/Context/Monitoring/Domain/Event/
├── ProfileCreated.php
├── ProfileActivated.php
├── ProfileDeactivated.php
├── FilterCreated.php
├── FilterValueUpdated.php
├── FilterTypeChanged.php
├── FilterActivated.php
├── FilterDeactivated.php
├── FilterDeleted.php
├── FilterBoundToGroup.php
├── FilterUnboundFromGroup.php
├── SessionCreated.php
├── SessionAuthorizationStarted.php
├── SessionAuthorized.php
├── SessionAuthorizationFailed.php
├── SessionExpired.php
├── SessionRevoked.php
├── SessionDeactivated.php
├── GroupCreated.php
├── GroupActivated.php
├── GroupJoinFailed.php
├── GroupDeactivated.php
└── GroupDeleted.php

tests/Unit/Context/Monitoring/Domain/Event/MonitoringEventsTest.php
```

## Требования

Все события реализуют `DomainEventInterface` из Shared Context.

### Profile Events

```php
final class ProfileCreated implements DomainEventInterface
{
    // payload: profileId, userId
    public function getEventName(): string { return 'monitoring.profile.created'; }
}

final class ProfileActivated implements DomainEventInterface
{
    // payload: profileId
    public function getEventName(): string { return 'monitoring.profile.activated'; }
}

final class ProfileDeactivated implements DomainEventInterface
{
    // payload: profileId
    public function getEventName(): string { return 'monitoring.profile.deactivated'; }
}
```

### Filter Events

```php
final class FilterCreated implements DomainEventInterface
{
    // payload: filterId, value, filterType, profileId
    public function getEventName(): string { return 'monitoring.filter.created'; }
}

final class FilterValueUpdated implements DomainEventInterface
{
    // payload: filterId, oldValue, newValue
    public function getEventName(): string { return 'monitoring.filter.value_updated'; }
}

final class FilterTypeChanged implements DomainEventInterface
{
    // payload: filterId, oldType, newType
    public function getEventName(): string { return 'monitoring.filter.type_changed'; }
}

final class FilterActivated implements DomainEventInterface
{
    // payload: filterId
    public function getEventName(): string { return 'monitoring.filter.activated'; }
}

final class FilterDeactivated implements DomainEventInterface
{
    // payload: filterId
    public function getEventName(): string { return 'monitoring.filter.deactivated'; }
}

final class FilterDeleted implements DomainEventInterface
{
    // payload: profileId, filterId
    public function getEventName(): string { return 'monitoring.filter.deleted'; }
}

final class FilterBoundToGroup implements DomainEventInterface
{
    // payload: profileId, filterId, groupId
    public function getEventName(): string { return 'monitoring.filter.bound_to_group'; }
}

final class FilterUnboundFromGroup implements DomainEventInterface
{
    // payload: profileId, filterId, groupId
    public function getEventName(): string { return 'monitoring.filter.unbound_from_group'; }
}
```

### Session Events

```php
final class SessionCreated implements DomainEventInterface
{
    // payload: sessionId, messengerType
    public function getEventName(): string { return 'monitoring.session.created'; }
}

final class SessionAuthorizationStarted implements DomainEventInterface
{
    // payload: sessionId, messengerType
    public function getEventName(): string { return 'monitoring.session.authorization_started'; }
}

final class SessionAuthorized implements DomainEventInterface
{
    // payload: sessionId, messengerType (externalSessionId НЕ передаётся)
    public function getEventName(): string { return 'monitoring.session.authorized'; }
}

final class SessionAuthorizationFailed implements DomainEventInterface
{
    // payload: sessionId, messengerType, reason
    public function getEventName(): string { return 'monitoring.session.authorization_failed'; }
}

final class SessionExpired implements DomainEventInterface
{
    // payload: sessionId, messengerType
    public function getEventName(): string { return 'monitoring.session.expired'; }
}

final class SessionRevoked implements DomainEventInterface
{
    // payload: sessionId, messengerType
    public function getEventName(): string { return 'monitoring.session.revoked'; }
}

final class SessionDeactivated implements DomainEventInterface
{
    // payload: sessionId, messengerType
    public function getEventName(): string { return 'monitoring.session.deactivated'; }
}
```

### Group Events

```php
final class GroupCreated implements DomainEventInterface
{
    // payload: groupId, groupTitle, messengerType, isPrivate, sessionId?
    // ВАЖНО: externalGroupId НЕ передаётся
    public function getEventName(): string { return 'monitoring.group.created'; }
}

final class GroupActivated implements DomainEventInterface
{
    // payload: groupId, messengerType
    public function getEventName(): string { return 'monitoring.group.activated'; }
}

final class GroupJoinFailed implements DomainEventInterface
{
    // payload: groupId, messengerType, reason
    public function getEventName(): string { return 'monitoring.group.join_failed'; }
}

final class GroupDeactivated implements DomainEventInterface
{
    // payload: groupId, messengerType, reason
    public function getEventName(): string { return 'monitoring.group.deactivated'; }
}

final class GroupDeleted implements DomainEventInterface
{
    // payload: profileId, groupId, messengerType
    public function getEventName(): string { return 'monitoring.group.deleted'; }
}
```

## Безопасность

**ВАЖНО**: Следующие данные НЕ должны включаться в payload событий:
- `externalSessionId` — ID сессии в auth-service
- `externalGroupId` — ID группы в мессенджере
- `phoneNumber` — номер телефона

## Тесты

- [x] Каждое событие создаётся корректно
- [x] getEventName() возвращает правильное имя
- [x] getPayload() содержит только сериализуемые данные
- [x] getOccurredAt() возвращает дату
- [x] Чувствительные данные не попадают в payload

## Зависимости

- `DomainEventInterface` (Shared Context) — задача 0007
- Value Objects (задача 0077)
- Enums (задачи 0078-0080)

## Definition of Done

- [x] Все 23 класса событий созданы
- [x] Все события immutable (final классы)
- [x] Чувствительные данные не передаются в payload
- [x] Unit-тесты написаны и проходят
- [x] Код соответствует PSR-12
- [x] PHPStan level 9 без ошибок
