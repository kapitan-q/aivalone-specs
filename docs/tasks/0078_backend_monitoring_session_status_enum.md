# Задача 0078: SessionStatus Enum

## Контекст

Monitoring Context использует перечисление `SessionStatus` для управления жизненным циклом авторизации сессии в мессенджере. Enum содержит логику проверки допустимых переходов состояний.

## Цель

Создать enum `SessionStatus` с методами проверки переходов состояний.

## Спецификация

- [SessionStatus Enum](../backend/monitoring/models/session-status-enum.md)

## Файлы для создания

```
src/Context/Monitoring/Domain/Model/SessionStatus.php
tests/Unit/Context/Monitoring/Domain/Model/SessionStatusTest.php
```

## Требования

```php
<?php

declare(strict_types=1);

namespace App\Context\Monitoring\Domain\Model;

enum SessionStatus: string
{
    case NEW = 'new';
    case AUTHORIZING = 'authorizing';
    case AUTHORIZED = 'authorized';
    case FAILED = 'failed';
    case EXPIRED = 'expired';

    public function canStartAuth(): bool
    {
        return in_array($this, [self::NEW, self::FAILED, self::EXPIRED], true);
    }

    public function canTransitionToAuthorized(): bool
    {
        return $this === self::AUTHORIZING;
    }

    public function canTransitionToFailed(): bool
    {
        return $this === self::AUTHORIZING;
    }

    public function canTransitionToExpired(): bool
    {
        return $this === self::AUTHORIZING || $this === self::AUTHORIZED;
    }

    public function isActive(): bool
    {
        return $this === self::AUTHORIZED;
    }

    public function isPending(): bool
    {
        return $this === self::NEW || $this === self::AUTHORIZING;
    }

    public function isTerminal(): bool
    {
        return $this === self::FAILED || $this === self::EXPIRED;
    }

    public function canTransitionTo(self $newStatus): bool
    {
        return match ($this) {
            self::NEW => $newStatus === self::AUTHORIZING,
            self::AUTHORIZING => in_array($newStatus, [self::AUTHORIZED, self::FAILED, self::EXPIRED], true),
            self::AUTHORIZED => $newStatus === self::EXPIRED,
            self::FAILED, self::EXPIRED => $newStatus === self::AUTHORIZING,
        };
    }
}
```

## Тесты

- [x] Все значения enum существуют (NEW, AUTHORIZING, AUTHORIZED, FAILED, EXPIRED)
- [x] canStartAuth() возвращает true для NEW, FAILED, EXPIRED
- [x] canStartAuth() возвращает false для AUTHORIZING, AUTHORIZED
- [x] canTransitionToAuthorized() возвращает true только для AUTHORIZING
- [x] canTransitionToFailed() возвращает true только для AUTHORIZING
- [x] canTransitionToExpired() возвращает true для AUTHORIZING и AUTHORIZED
- [x] isActive() возвращает true только для AUTHORIZED
- [x] isPending() возвращает true для NEW и AUTHORIZING
- [x] isTerminal() возвращает true для FAILED и EXPIRED
- [x] canTransitionTo() проверяет все допустимые переходы
- [x] canTransitionTo() возвращает false для недопустимых переходов

## Зависимости

Нет

## Definition of Done

- [x] Enum SessionStatus создан со всеми значениями
- [x] Все методы проверки переходов реализованы
- [x] Unit-тесты написаны и проходят
- [x] Код соответствует PSR-12
- [x] PHPStan level 9 без ошибок
