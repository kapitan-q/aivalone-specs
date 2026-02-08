# Задача 0097: BillingLimitsClient

## Контекст

Monitoring Context проверяет лимиты пользователя по тарифу перед созданием фильтров и добавлением групп. Для этого используется синхронный клиент к Billing Context.

## Цель

Создать интерфейс `BillingLimitsClientInterface` и его реализацию для получения лимитов пользователя.

## Спецификация

- [Infrastructure Overview](../backend/monitoring/infrastructure/overview.md)
- [Filter Management Process](../backend/monitoring/processes/filter-management.md)
- [Subscribe to Group Process](../backend/monitoring/processes/subscribe-to-group.md)

## Файлы для создания

```
src/Context/Monitoring/Application/Port/
└── BillingLimitsClientInterface.php

src/Context/Monitoring/Application/DTO/
└── UserLimitsDTO.php

src/Context/Monitoring/Infrastructure/Client/
└── BillingLimitsClient.php

tests/Unit/Context/Monitoring/Infrastructure/Client/
└── BillingLimitsClientTest.php
```

## Требования

### BillingLimitsClientInterface (Application Port)

```php
<?php

declare(strict_types=1);

namespace App\Context\Monitoring\Application\Port;

use App\Context\Account\Domain\Model\UserId;
use App\Context\Monitoring\Application\DTO\UserLimitsDTO;

interface BillingLimitsClientInterface
{
    /**
     * Получить лимиты пользователя по текущему тарифу.
     *
     * @throws BillingServiceUnavailableException при недоступности Billing
     */
    public function getUserLimits(UserId $userId): UserLimitsDTO;
}
```

### UserLimitsDTO

```php
<?php

declare(strict_types=1);

namespace App\Context\Monitoring\Application\DTO;

use App\Context\Shared\Domain\Enum\FilterType;

final readonly class UserLimitsDTO
{
    /**
     * @param FilterType[] $allowedFilterTypes
     */
    public function __construct(
        public int $maxMonitoredGroups,
        public int $maxFilters,
        public array $allowedFilterTypes,
        public int $maxNotificationsPerDay,
    ) {}

    /**
     * Значения по умолчанию для free tier.
     */
    public static function defaults(): self
    {
        return new self(
            maxMonitoredGroups: 3,
            maxFilters: 5,
            allowedFilterTypes: [FilterType::KEYWORD],
            maxNotificationsPerDay: 50,
        );
    }
}
```

### BillingLimitsClient (Infrastructure Implementation)

```php
<?php

declare(strict_types=1);

namespace App\Context\Monitoring\Infrastructure\Client;

use App\Context\Account\Domain\Model\UserId;
use App\Context\Billing\Application\Service\BillingQueryService;
use App\Context\Monitoring\Application\DTO\UserLimitsDTO;
use App\Context\Monitoring\Application\Port\BillingLimitsClientInterface;

class BillingLimitsClient implements BillingLimitsClientInterface
{
    public function __construct(
        private BillingQueryService $billingQueryService,
    ) {}

    public function getUserLimits(UserId $userId): UserLimitsDTO
    {
        // 1. Запросить текущую подписку пользователя через BillingQueryService
        // 2. Получить лимиты из тарифного плана
        // 3. Маппинг в UserLimitsDTO
        // 4. При ошибке — вернуть defaults() как fallback
    }
}
```

### Обработка ошибок

Если Billing Context недоступен:
- **Стратегия**: использовать дефолтные лимиты (free tier) как fallback
- **Логирование**: warning при использовании fallback
- **Мониторинг**: метрика `billing_client_failures_total`

## Тесты

- [x] getUserLimits() возвращает корректные лимиты из Billing
- [x] getUserLimits() возвращает defaults при ошибке Billing (fallback)
- [x] defaults() возвращает лимиты free tier
- [x] UserLimitsDTO содержит все необходимые поля
- [x] allowedFilterTypes содержит корректные типы

## Зависимости

- UserId (Account Context — задача 0017)
- FilterType (Shared Context — задача 0080)
- BillingQueryService (Billing Context)

## Definition of Done

- [x] BillingLimitsClientInterface создан в Application/Port
- [x] UserLimitsDTO реализован
- [x] BillingLimitsClient реализован с fallback стратегией
- [x] Unit-тесты написаны и проходят
- [x] Код соответствует PSR-12
- [ ] PHPStan level 9 без ошибок
