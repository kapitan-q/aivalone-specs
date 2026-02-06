# Задача 0081: Monitoring Domain Exceptions

## Контекст

Monitoring Context требует набор доменных исключений для обработки ошибок бизнес-логики: нарушения инвариантов, превышение лимитов, невалидные состояния.

## Цель

Создать все доменные исключения для Monitoring Context.

## Спецификация

- [Exceptions Overview](../backend/monitoring/exceptions/overview.md)
- [ProfileNotFoundException](../backend/monitoring/exceptions/profile-not-found-exception.md)
- [GroupLimitReachedException](../backend/monitoring/exceptions/group-limit-reached-exception.md)

## Файлы для создания

```
src/Context/Monitoring/Domain/Exception/
├── ProfileNotFoundException.php
├── ProfileNotActiveException.php
├── ProfileAlreadyExistsException.php
├── FilterNotFoundException.php
├── FilterAlreadyExistsException.php
├── FilterLimitReachedException.php
├── FilterTypeNotAllowedException.php
├── InvalidRegexPatternException.php
├── SessionNotFoundException.php
├── SessionAlreadyExistsException.php
├── SessionNotAuthorizedException.php
├── InvalidSessionStateException.php
├── GroupNotFoundException.php
├── GroupAlreadyExistsException.php
├── GroupLimitReachedException.php
├── GroupRequiresSessionException.php
├── InvalidGroupStateException.php
├── BindingAlreadyExistsException.php
└── BindingNotFoundException.php

tests/Unit/Context/Monitoring/Domain/Exception/MonitoringExceptionsTest.php
```

## Исключения

### Profile Exceptions

#### ProfileNotFoundException

```php
class ProfileNotFoundException extends DomainException
{
    public static function byId(MonitoringProfileId $id): self;
    public static function byUserId(UserId $userId): self;
}
```

#### ProfileNotActiveException

```php
class ProfileNotActiveException extends DomainException
{
    public static function create(MonitoringProfileId $profileId): self;
}
```

#### ProfileAlreadyExistsException

```php
class ProfileAlreadyExistsException extends DomainException
{
    public static function forUser(UserId $userId): self;
}
```

### Filter Exceptions

#### FilterNotFoundException

```php
class FilterNotFoundException extends DomainException
{
    public static function withId(FilterId $filterId): self;
}
```

#### FilterAlreadyExistsException

```php
class FilterAlreadyExistsException extends DomainException
{
    public static function withValue(string $value, FilterType $filterType): self;
}
```

#### FilterLimitReachedException

```php
class FilterLimitReachedException extends DomainException
{
    public static function create(MonitoringProfileId $profileId, int $currentCount, int $maxAllowed): self;
}
```

#### FilterTypeNotAllowedException

```php
class FilterTypeNotAllowedException extends DomainException
{
    public static function create(FilterType $filterType, array $allowedTypes): self;
}
```

#### InvalidRegexPatternException

```php
class InvalidRegexPatternException extends ValidationException
{
    public static function fromPattern(string $pattern): self;
}
```

### Session Exceptions

#### SessionNotFoundException

```php
class SessionNotFoundException extends DomainException
{
    public static function withId(SessionId $sessionId): self;
}
```

#### SessionAlreadyExistsException

```php
class SessionAlreadyExistsException extends DomainException
{
    public static function forMessenger(Messenger $messenger): self;
}
```

#### SessionNotAuthorizedException

```php
class SessionNotAuthorizedException extends DomainException
{
    public static function create(SessionId $sessionId): self;
}
```

#### InvalidSessionStateException

```php
class InvalidSessionStateException extends DomainException
{
    public static function create(SessionId $sessionId, SessionStatus $currentState, array $expectedStates): self;
}
```

### Group Exceptions

#### GroupNotFoundException

```php
class GroupNotFoundException extends DomainException
{
    public static function withId(GroupId $groupId): self;
}
```

#### GroupAlreadyExistsException

```php
class GroupAlreadyExistsException extends DomainException
{
    public static function create(string $externalGroupId, Messenger $messenger): self;
}
```

#### GroupLimitReachedException

```php
class GroupLimitReachedException extends DomainException
{
    public static function create(MonitoringProfileId $profileId, int $currentCount, int $maxAllowed): self;
}
```

#### GroupRequiresSessionException

```php
class GroupRequiresSessionException extends DomainException
{
    public static function create(Messenger $messenger): self;
}
```

#### InvalidGroupStateException

```php
class InvalidGroupStateException extends DomainException
{
    public static function create(GroupId $groupId, GroupStatus $currentState): self;
}
```

### Binding Exceptions

#### BindingAlreadyExistsException

```php
class BindingAlreadyExistsException extends DomainException
{
    public static function create(FilterId $filterId, GroupId $groupId): self;
}
```

#### BindingNotFoundException

```php
class BindingNotFoundException extends DomainException
{
    public static function create(FilterId $filterId, GroupId $groupId): self;
}
```

## Тесты

- [x] Каждое исключение создаётся через фабричный метод
- [x] Сообщения об ошибках информативны
- [x] Все исключения наследуют DomainException или ValidationException

## Зависимости

- `DomainException` (Shared Context) — задача 0001
- `ValidationException` (Shared Context) — задача 0002
- Value Objects (задача 0077)
- Enums (задачи 0078-0080)

## Definition of Done

- [x] Все 19 классов исключений созданы
- [x] Реализованы статические фабричные методы
- [x] Unit-тесты написаны и проходят
- [x] Код соответствует PSR-12
- [x] PHPStan level 9 без ошибок
