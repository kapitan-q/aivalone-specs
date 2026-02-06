# Задача 0077: Monitoring Value Objects

## Контекст

Monitoring Context требует типизированные идентификаторы для своих сущностей. Все Value Objects наследуют базовый `UUID` из Shared Context.

## Цель

Создать Value Objects: `MonitoringProfileId`, `FilterId`, `SessionId`, `GroupId`.

## Спецификации

- [MonitoringProfileId](../backend/monitoring/models/monitoring-profile-id.md)
- [FilterId](../backend/monitoring/models/filter-id.md)
- [SessionId](../backend/monitoring/models/session-id.md)
- [GroupId](../backend/monitoring/models/group-id.md)

## Файлы для создания

```
src/Context/Monitoring/Domain/Model/MonitoringProfileId.php
src/Context/Monitoring/Domain/Model/FilterId.php
src/Context/Monitoring/Domain/Model/SessionId.php
src/Context/Monitoring/Domain/Model/GroupId.php
tests/Unit/Context/Monitoring/Domain/Model/MonitoringProfileIdTest.php
tests/Unit/Context/Monitoring/Domain/Model/FilterIdTest.php
tests/Unit/Context/Monitoring/Domain/Model/SessionIdTest.php
tests/Unit/Context/Monitoring/Domain/Model/GroupIdTest.php
```

## Требования

### MonitoringProfileId

```php
<?php

declare(strict_types=1);

namespace App\Context\Monitoring\Domain\Model;

use App\Context\Shared\Domain\Model\UUID;

class MonitoringProfileId extends UUID
{
}
```

### FilterId

```php
<?php

declare(strict_types=1);

namespace App\Context\Monitoring\Domain\Model;

use App\Context\Shared\Domain\Model\UUID;

class FilterId extends UUID
{
}
```

### SessionId

```php
<?php

declare(strict_types=1);

namespace App\Context\Monitoring\Domain\Model;

use App\Context\Shared\Domain\Model\UUID;

class SessionId extends UUID
{
}
```

### GroupId

```php
<?php

declare(strict_types=1);

namespace App\Context\Monitoring\Domain\Model;

use App\Context\Shared\Domain\Model\UUID;

class GroupId extends UUID
{
}
```

## Тесты (для каждого VO)

- [x] Генерация нового ID
- [x] Создание из строки (валидный UUID)
- [x] Исключение при невалидном UUID
- [x] Метод equals() корректно сравнивает идентификаторы
- [x] Метод toString() возвращает строковое представление

## Зависимости

- `App\Context\Shared\Domain\Model\UUID` (задача 0005)

## Definition of Done

- [x] Все 4 класса созданы
- [x] Unit-тесты написаны и проходят
- [x] Код соответствует PSR-12
- [x] PHPStan level 9 без ошибок
