# Задача 0052: EndpointStatus Enum

## Контекст

NotificationEndpoint имеет жизненный цикл с различными статусами. EndpointStatus определяет возможные состояния endpoint.

## Цель

Создать Enum `EndpointStatus` для определения статусов NotificationEndpoint.

## Спецификация

- [EndpointStatus Enum](../backend/bot/models/endpoint-status-enum.md)

## Файлы для создания

```
src/Context/Bot/Domain/Model/EndpointStatus.php
tests/Unit/Context/Bot/Domain/Model/EndpointStatusTest.php
```

## Требования

### EndpointStatus

```php
<?php

declare(strict_types=1);

namespace App\Context\Bot\Domain\Model;

enum EndpointStatus: string
{
    case ACTIVE = 'active';
    case REVOKED = 'revoked';
    case BLOCKED = 'blocked';

    public function isActive(): bool
    {
        return $this === self::ACTIVE;
    }

    public function canReceiveMessages(): bool
    {
        return $this === self::ACTIVE;
    }

    public function isTerminal(): bool
    {
        return $this === self::REVOKED;
    }

    public function label(): string
    {
        return match ($this) {
            self::ACTIVE => 'Активен',
            self::REVOKED => 'Отозван',
            self::BLOCKED => 'Заблокирован',
        };
    }
}
```

## Описание статусов

| Статус | Описание |
|--------|----------|
| `ACTIVE` | Endpoint активен и может получать сообщения |
| `REVOKED` | Endpoint отозван пользователем (/stop) |
| `BLOCKED` | Endpoint заблокирован (пользователь заблокировал бота) |

## Тесты

- [x] Все значения enum доступны
- [x] isActive() возвращает true только для ACTIVE
- [x] canReceiveMessages() возвращает true только для ACTIVE
- [x] isTerminal() возвращает true только для REVOKED
- [x] label() возвращает корректные локализованные названия

## Зависимости

Нет внешних зависимостей.

## Definition of Done

- [x] Enum EndpointStatus создан
- [x] Все методы реализованы
- [x] Unit-тесты написаны и проходят
- [x] Код соответствует PSR-12
- [x] PHPStan level 9 без ошибок
