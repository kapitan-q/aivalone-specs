# SubscriptionStatus Enum

## Описание

Enum `SubscriptionStatus` определяет возможные статусы подписки пользователя.

## Доступные типы

| Значение       | Код             | Описание          |
|----------------|-----------------|-------------------|
| ACTIVE         | `ACTIVE`        | Подписка активна  |
| EXPIRED        | `EXPIRED`       | Подписка истекла  |
| CANCELLED      | `CANCELLED`     | Подписка отменена |

## Основные методы

### static fromCode(string $status): self

Создание экземпляра из строкового кода.

- **Параметры**: `$status` — строка с кодом статуса (например, 'ACTIVE')
- **Возвращает**: Экземпляр `SubscriptionStatus`
- **Исключения**: `InvalidSubscriptionStatusException` при невалидном статусе

### code(): string

Возвращает строковый код статуса.

Используется:

* в событиях
* в сериализации
* при передаче между сервисами

### isActive(): bool

Возвращает true если статус = ACTIVE.

### isExpired(): bool

Возвращает true если статус = EXPIRED.

### isCancelled(): bool

Возвращает true если статус = CANCELLED.

## Статус реализации

- [ ] Enum создан с базовой структурой
- [ ] Все значения определены (ACTIVE, EXPIRED, CANCELLED)
- [ ] Методы fromCode, code реализован
- [ ] Методы isActive, isExpired, isCancelled реализованы
- [ ] Unit тесты написаны (5+ тестов)
- [ ] Integration тесты пройдены
- [ ] Код соответствует стилю проекта
- [ ] Документация актуальна

## Связанные сущности

- [UserSubscription Aggregate Root](../models/user-subscription-aggregate-root.md)
- [Billing Context Overview](../overview.md)
