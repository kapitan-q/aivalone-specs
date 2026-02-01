# Задача 0050: REST API Controllers

## Описание

Создать REST API контроллеры для Billing Context:
- `GET /api/tariffs` — список тарифов
- `GET /api/users/{userId}/subscriptions` — подписки пользователя

## Фаза

**Phase 6: REST API**

## Спецификация

📄 [endpoints.md](../backend/billing/api/endpoints.md)

## Зависимости

- ⏳ `GetTariffListHandler` — задача 0045
- ⏳ `SubscriptionService` — задача 0046
- ⏳ Doctrine Infrastructure — задача 0048
- ✅ Symfony Framework

## Расположение файлов

```
src/Context/Billing/Presentation/Http/Controller/
├── TariffController.php
└── SubscriptionController.php
```

---

## 1. GET /api/tariffs

### Описание

Возвращает список всех доступных тарифов с их опциями.

### Controller

```php
#[Route('/api/tariffs', name: 'api_tariffs_')]
class TariffController extends AbstractController
{
    public function __construct(
        private GetTariffListHandler $getTariffListHandler
    ) {}

    #[Route('', name: 'list', methods: ['GET'])]
    public function list(): JsonResponse
    {
        $result = ($this->getTariffListHandler)(new GetTariffListQuery());

        return $this->json([
            'data' => array_map(
                fn(TariffDto $tariff) => [
                    'id' => $tariff->id,
                    'code' => $tariff->code,
                    'name' => $tariff->name,
                    'priority' => $tariff->priority,
                    'price' => $tariff->price,
                    'options' => array_map(
                        fn(TariffOptionDto $option) => [
                            'name' => $option->name,
                            'code' => $option->code,
                            'type' => $option->type,
                            'value' => $option->value,
                        ],
                        $tariff->options
                    ),
                ],
                $result->tariffs
            ),
        ]);
    }
}
```

### Response Example

```json
{
  "data": [
    {
      "id": "550e8400-e29b-41d4-a716-446655440000",
      "code": "free",
      "name": "Free Plan",
      "priority": 0,
      "price": 0.00,
      "options": [
        {
          "name": "Max Groups",
          "code": "MAX_GROUPS",
          "type": "max_constraint",
          "value": 5
        }
      ]
    },
    {
      "id": "550e8400-e29b-41d4-a716-446655440001",
      "code": "pro",
      "name": "Pro Plan",
      "priority": 2,
      "price": 9.99,
      "options": [
        {
          "name": "Max Groups",
          "code": "MAX_GROUPS",
          "type": "max_constraint",
          "value": 50
        },
        {
          "name": "Priority Support",
          "code": "PRIORITY_SUPPORT",
          "type": "bool",
          "value": true
        }
      ]
    }
  ]
}
```

---

## 2. GET /api/users/{userId}/subscriptions

### Описание

Возвращает все подписки пользователя (активные и исторические).

### Controller

```php
#[Route('/api/users/{userId}/subscriptions', name: 'api_user_subscriptions_')]
class SubscriptionController extends AbstractController
{
    public function __construct(
        private SubscriptionServiceInterface $subscriptionService,
        private TariffRepositoryInterface $tariffRepository
    ) {}

    #[Route('', name: 'list', methods: ['GET'])]
    public function list(string $userId): JsonResponse
    {
        try {
            $userIdVo = UserId::fromString($userId);
        } catch (InvalidUuidException $e) {
            throw new BadRequestHttpException('Invalid userId format');
        }

        $subscriptions = $this->subscriptionService->getAllSubscriptions($userIdVo);

        return $this->json([
            'data' => array_map(
                fn(UserSubscription $sub) => $this->mapSubscription($sub),
                $subscriptions
            ),
        ]);
    }

    private function mapSubscription(UserSubscription $subscription): array
    {
        $tariff = $this->tariffRepository->findById($subscription->getTariffId());

        return [
            'id' => $subscription->getId()->toString(),
            'tariff' => $tariff ? [
                'id' => $tariff->getId()->toString(),
                'code' => $tariff->getCode()->code(),
                'name' => $tariff->getName(),
            ] : null,
            'status' => $subscription->getStatus()->code(),
            'period' => $subscription->getPeriod()?->code(),
            'validUntil' => $subscription->getValidUntil()?->format('c'),
            'isPermanent' => $subscription->isPermanent(),
            'isRenewal' => $subscription->isRenewal(),
            'previousSubscriptionId' => $subscription->getPreviousSubscriptionId()?->toString(),
            'createdAt' => $subscription->getCreatedAt()->format('c'),
        ];
    }
}
```

### Response Example

```json
{
  "data": [
    {
      "id": "123e4567-e89b-12d3-a456-426614174000",
      "tariff": {
        "id": "550e8400-e29b-41d4-a716-446655440000",
        "code": "free",
        "name": "Free Plan"
      },
      "status": "active",
      "period": null,
      "validUntil": null,
      "isPermanent": true,
      "isRenewal": false,
      "previousSubscriptionId": null,
      "createdAt": "2024-01-15T10:30:00+00:00"
    },
    {
      "id": "123e4567-e89b-12d3-a456-426614174001",
      "tariff": {
        "id": "550e8400-e29b-41d4-a716-446655440001",
        "code": "pro",
        "name": "Pro Plan"
      },
      "status": "active",
      "period": "year",
      "validUntil": "2025-01-15T10:30:00+00:00",
      "isPermanent": false,
      "isRenewal": true,
      "previousSubscriptionId": "123e4567-e89b-12d3-a456-426614173999",
      "createdAt": "2024-01-15T10:35:00+00:00"
    }
  ]
}
```

---

## Обработка ошибок

### Error Response Format

```json
{
  "error": {
    "code": "TARIFF_NOT_FOUND",
    "message": "Tariff with code 'invalid' not found"
  }
}
```

### HTTP Status Codes

| Код | Когда |
| --- | ----- |
| 200 | Успех |
| 400 | Невалидные параметры (например, невалидный UUID) |
| 404 | Ресурс не найден |
| 500 | Внутренняя ошибка |

---

## Критерии готовности

- [x] Создан `TariffController` с GET /api/tariffs
- [x] Создан `SubscriptionController` с GET /api/users/{userId}/subscriptions
- [x] Реализована обработка ошибок
- [x] Ответы соответствуют JSON формату из спецификации
- [ ] Написаны функциональные тесты
- [x] Код соответствует PSR-12

## Статус: ✅ ВЫПОЛНЕНО

## Связанные задачи

- 0045: GetTariffList Query Handler
- 0046: SubscriptionService
- 0048: Doctrine Infrastructure
