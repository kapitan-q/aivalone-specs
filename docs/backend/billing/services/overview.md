# Обзор сервисов Billing Context

## Основные сервисы

| Сервис | Описание | Файл |
| ------ | -------- | ---- |
| **SubscriptionService** | Управление подписками и тарифами (бизнес-логика) | [subscription-service.md](subscription-service.md) |

## Структура сервисов

```
docs/backend/billing/services/
├── overview.md                    # Этот файл (оглавление)
└── subscription-service.md        # Спецификация сервиса
```

## SubscriptionService

Application Service, который содержит всю бизнес-логику управления подписками.

**Основные методы**:
- `addSubscription(UserId $userId, Tariff $tariff): UserSubscription`
- `removeSubscription(UserId $userId, Tariff $tariff): void`
- `getSubscriptionsByUserId(UserId $userId): array<UserSubscription>`
- `getActiveSubscriptionByUserAndTariff(UserId $userId, TariffId $tariffId): UserSubscription|null`

**Ответственность**:
- Выполняет всю бизнес-логику
- Управляет репозиториями
- Регистрирует события в агрегатах
- Command Handlers используют этот сервис и только валидируют входные данные

## Использование в архитектуре

```
Command Handler
  ↓
(валидирует, конвертирует)
  ↓
SubscriptionService
  ↓
(выполняет бизнес-логику)
  ↓
Repository / Domain Model
  ↓
EventBus (публикует события)
```

## Связанные документы

* [Billing Context Overview](../overview.md)
* [Модели](../models/overview.md)
* [События](../events/overview.md)

