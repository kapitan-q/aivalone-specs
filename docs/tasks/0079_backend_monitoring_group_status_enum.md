# Задача 0079: GroupStatus Enum

## Контекст

Monitoring Context использует перечисление `GroupStatus` для управления жизненным циклом группы на прослушивание.

## Цель

Создать enum `GroupStatus` с методами проверки переходов состояний.

## Спецификация

- [GroupStatus Enum](../backend/monitoring/models/group-status-enum.md)

## Файлы для создания

```
src/Context/Monitoring/Domain/Model/GroupStatus.php
tests/Unit/Context/Monitoring/Domain/Model/GroupStatusTest.php
```

## Требования

```php
<?php

declare(strict_types=1);

namespace App\Context\Monitoring\Domain\Model;

enum GroupStatus: string
{
    case PENDING = 'pending';
    case ACTIVE = 'active';
    case FAILED = 'failed';

    public function canTransitionToActive(): bool
    {
        return $this === self::PENDING;
    }

    public function canTransitionToFailed(): bool
    {
        return $this === self::PENDING || $this === self::ACTIVE;
    }
}
```

## Тесты

- [x] Все значения enum существуют (PENDING, ACTIVE, FAILED)
- [x] canTransitionToActive() возвращает true только для PENDING
- [x] canTransitionToActive() возвращает false для ACTIVE и FAILED
- [x] canTransitionToFailed() возвращает true для PENDING и ACTIVE
- [x] canTransitionToFailed() возвращает false для FAILED

## Зависимости

Нет

## Definition of Done

- [x] Enum GroupStatus создан со всеми значениями
- [x] Все методы проверки переходов реализованы
- [x] Unit-тесты написаны и проходят
- [x] Код соответствует PSR-12
- [x] PHPStan level 9 без ошибок
