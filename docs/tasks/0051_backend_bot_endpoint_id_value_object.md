# Задача 0051: EndpointId Value Object

## Контекст

Bot Context требует уникальный идентификатор для NotificationEndpoint. EndpointId — это Value Object, инкапсулирующий UUID идентификатора endpoint.

## Цель

Создать Value Object `EndpointId` для идентификации NotificationEndpoint.

## Спецификация

- [EndpointId](../backend/bot/models/endpoint-id.md)

## Файлы для создания

```
src/Context/Bot/Domain/Model/EndpointId.php
tests/Unit/Context/Bot/Domain/Model/EndpointIdTest.php
```

## Требования

### EndpointId

```php
<?php

declare(strict_types=1);

namespace App\Context\Bot\Domain\Model;

use App\Context\Shared\Domain\Model\UUID;

/**
 * Типизированный идентификатор NotificationEndpoint.
 *
 * Уникальный идентификатор для endpoint уведомлений пользователя
 * в конкретном мессенджере.
 */
class EndpointId extends UUID
{
}
```

## Тесты

- [x] Генерация нового EndpointId
- [x] Создание из строки (валидный UUID)
- [x] Исключение при невалидном UUID
- [x] Метод equals() корректно сравнивает идентификаторы
- [x] Метод toString() возвращает строковое представление

## Зависимости

- `App\Context\Shared\Domain\Model\UUID` (задача 0005)

## Definition of Done

- [x] Класс EndpointId создан
- [x] Все методы реализованы
- [x] Unit-тесты написаны и проходят
- [x] Код соответствует PSR-12
- [x] PHPStan level 9 без ошибок
