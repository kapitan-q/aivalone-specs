# REST API Endpoints — Billing Context

## Обзор

REST API для работы с тарифами и подписками пользователей.

## Базовый URL

```
/api/billing
```

## Аутентификация

Все эндпоинты требуют аутентификации через JWT токен в заголовке:

```
Authorization: Bearer <token>
```

---

## Эндпоинты

### 1. GET /api/tariffs

Получение списка всех доступных тарифов с их опциями.

#### Запрос

```http
GET /api/tariffs HTTP/1.1
Authorization: Bearer <token>
```

#### Успешный ответ (200 OK)

```json
{
    "data": [
        {
            "id": "550e8400-e29b-41d4-a716-446655440000",
            "code": "FREE",
            "name": "Free Plan",
            "priority": 0,
            "price": "0.00",
            "options": [
                {
                    "code": "MAX_GROUPS",
                    "name": "Maximum Groups",
                    "type": "MAX_CONSTRAINT",
                    "value": 5
                },
                {
                    "code": "MAX_CHANNELS",
                    "name": "Maximum Channels",
                    "type": "MAX_CONSTRAINT",
                    "value": 10
                }
            ]
        },
        {
            "id": "550e8400-e29b-41d4-a716-446655440001",
            "code": "PRO",
            "name": "Pro Plan",
            "priority": 1,
            "price": "9.99",
            "options": [
                {
                    "code": "MAX_GROUPS",
                    "name": "Maximum Groups",
                    "type": "MAX_CONSTRAINT",
                    "value": 50
                },
                {
                    "code": "MAX_CHANNELS",
                    "name": "Maximum Channels",
                    "type": "MAX_CONSTRAINT",
                    "value": 100
                },
                {
                    "code": "CAN_EXPORT",
                    "name": "Export Feature",
                    "type": "BOOL",
                    "value": true
                }
            ]
        }
    ],
    "meta": {
        "total": 2
    }
}
```

#### Ошибки

| Код | Описание |
|-----|----------|
| 401 | Unauthorized — токен отсутствует или невалидный |

---

### 2. GET /api/users/{userId}/subscriptions

Получение всех активных подписок пользователя.

#### Запрос

```http
GET /api/users/123e4567-e89b-12d3-a456-426614174000/subscriptions HTTP/1.1
Authorization: Bearer <token>
```

#### Параметры URL

| Параметр | Тип | Описание |
|----------|-----|----------|
| userId | UUID | Идентификатор пользователя |

#### Query параметры (опционально)

| Параметр | Тип | По умолчанию | Описание |
|----------|-----|--------------|----------|
| status | string | `active` | Фильтр по статусу: `active`, `expired`, `cancelled`, `all` |

#### Успешный ответ (200 OK)

```json
{
    "data": [
        {
            "id": "660e8400-e29b-41d4-a716-446655440000",
            "userId": "123e4567-e89b-12d3-a456-426614174000",
            "tariff": {
                "id": "550e8400-e29b-41d4-a716-446655440000",
                "code": "FREE",
                "name": "Free Plan"
            },
            "period": null,
            "status": "ACTIVE",
            "createdAt": "2024-01-01T00:00:00+00:00",
            "validUntil": null,
            "isPermanent": true,
            "isRenewal": false
        },
        {
            "id": "660e8400-e29b-41d4-a716-446655440001",
            "userId": "123e4567-e89b-12d3-a456-426614174000",
            "tariff": {
                "id": "550e8400-e29b-41d4-a716-446655440001",
                "code": "PRO",
                "name": "Pro Plan"
            },
            "period": "MONTH",
            "status": "ACTIVE",
            "createdAt": "2024-06-15T10:30:00+00:00",
            "validUntil": "2024-07-15T10:30:00+00:00",
            "isPermanent": false,
            "isRenewal": false
        }
    ],
    "meta": {
        "total": 2,
        "activeCount": 2
    }
}
```

#### Ошибки

| Код | Описание |
|-----|----------|
| 401 | Unauthorized — токен отсутствует или невалидный |
| 403 | Forbidden — нет прав на просмотр подписок этого пользователя |
| 404 | Not Found — пользователь не найден |

---

## Контроллер

### Расположение

```
src/Context/Billing/Presentation/Http/Controller/TariffController.php
src/Context/Billing/Presentation/Http/Controller/SubscriptionController.php
```

### TariffController

```php
namespace App\Context\Billing\Presentation\Http\Controller;

use App\Context\Billing\Application\Query\GetTariffListQuery;
use Symfony\Component\HttpFoundation\JsonResponse;
use Symfony\Component\Routing\Annotation\Route;

#[Route('/api/tariffs')]
final class TariffController
{
    public function __construct(
        private readonly QueryBusInterface $queryBus,
    ) {}

    #[Route('', methods: ['GET'])]
    public function list(): JsonResponse
    {
        $tariffs = $this->queryBus->handle(new GetTariffListQuery());

        return new JsonResponse([
            'data' => array_map(fn($t) => $this->serializeTariff($t), $tariffs),
            'meta' => ['total' => count($tariffs)],
        ]);
    }

    private function serializeTariff(Tariff $tariff): array
    {
        return [
            'id' => $tariff->getId()->toString(),
            'code' => $tariff->getCode()->code(),
            'name' => $tariff->getName(),
            'priority' => $tariff->getPriority(),
            'price' => number_format($tariff->getPrice(), 2, '.', ''),
            'options' => array_map(fn($o) => [
                'code' => $o->getCode(),
                'name' => $o->getName(),
                'type' => $o->getType()->value,
                'value' => $o->getValue(),
            ], $tariff->getOptions()->toArray()),
        ];
    }
}
```

### SubscriptionController

```php
namespace App\Context\Billing\Presentation\Http\Controller;

use App\Context\Billing\Application\Service\SubscriptionService;
use App\Context\Shared\Domain\Model\UserId;
use Symfony\Component\HttpFoundation\JsonResponse;
use Symfony\Component\HttpFoundation\Request;
use Symfony\Component\Routing\Annotation\Route;

#[Route('/api/users/{userId}/subscriptions')]
final class SubscriptionController
{
    public function __construct(
        private readonly SubscriptionService $subscriptionService,
    ) {}

    #[Route('', methods: ['GET'])]
    public function list(string $userId, Request $request): JsonResponse
    {
        $status = $request->query->get('status', 'active');
        $userIdVo = UserId::fromString($userId);

        $subscriptions = match ($status) {
            'active' => $this->subscriptionService->getActiveSubscriptions($userIdVo),
            'all' => $this->subscriptionService->getAllSubscriptions($userIdVo),
            default => $this->subscriptionService->getActiveSubscriptions($userIdVo),
        };

        return new JsonResponse([
            'data' => array_map(fn($s) => $this->serializeSubscription($s), $subscriptions),
            'meta' => [
                'total' => count($subscriptions),
                'activeCount' => count(array_filter($subscriptions, fn($s) => $s->isActive())),
            ],
        ]);
    }

    private function serializeSubscription(UserSubscription $sub): array
    {
        return [
            'id' => $sub->getId()->toString(),
            'userId' => $sub->getUserId()->toString(),
            'tariff' => [
                'id' => $sub->getTariffId()->toString(),
                'code' => $sub->getTariffCode(),
                'name' => $sub->getTariffName(),
            ],
            'period' => $sub->getPeriod()?->code(),
            'status' => $sub->getStatus()->value,
            'createdAt' => $sub->getCreatedAt()->format(\DateTimeInterface::ATOM),
            'validUntil' => $sub->getValidUntil()?->format(\DateTimeInterface::ATOM),
            'isPermanent' => $sub->isPermanent(),
            'isRenewal' => $sub->isRenewal(),
        ];
    }
}
```

---

## Статус реализации

- [ ] TariffController создан
- [ ] SubscriptionController создан
- [ ] Роутинг настроен
- [ ] Сериализация реализована
- [ ] Аутентификация подключена
- [ ] Авторизация (проверка прав) реализована
- [ ] Валидация параметров добавлена
- [ ] Unit тесты написаны (6+ тестов)
- [ ] Integration тесты пройдены
- [ ] OpenAPI/Swagger документация добавлена

## Связанные документы

- [GetTariffList Process](../processes/get-tariff-list-process.md)
- [SubscriptionService](../services/subscription-service.md)
- [Billing Context Overview](../overview.md)
