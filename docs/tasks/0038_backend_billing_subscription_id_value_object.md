# Задача 0038: UserSubscriptionId Value Object

## Описание

Создать Value Object `UserSubscriptionId` — типизированный идентификатор подписки на основе UUID.

## Фаза

**Phase 1: Foundation**

## Спецификация

📄 [user-subscription-id-value-object.md](../backend/billing/models/user-subscription-id-value-object.md)

## Зависимости

- ✅ `Uuid` (Shared Context) — задача 0005
- ✅ `InvalidUuidException` (Shared Context) — задача 0003

## Расположение файла

```
src/Context/Billing/Domain/Model/UserSubscriptionId.php
```

## Требования к реализации

### Методы

| Метод | Описание |
| ----- | -------- |
| `generate(): self` | Создать новый UserSubscriptionId с UUID v4 |
| `fromString(string): self` | Создать из строки UUID |
| `toString(): string` | Преобразовать в строку |
| `equals(self): bool` | Сравнение по значению |
| `__toString(): string` | Магический метод для преобразования |

### Инварианты

- Value Object immutable
- Валидация формата UUID при создании из строки
- При невалидном UUID выбрасывается `InvalidUuidException`

## Пример использования

```php
// Генерация нового ID
$id = UserSubscriptionId::generate();

// Создание из строки
$id = UserSubscriptionId::fromString('123e4567-e89b-12d3-a456-426614174000');

// Использование
$idString = $id->toString();
$id->equals($anotherId); // bool

// Для previousSubscriptionId при продлении
$previousId = $currentSubscription->getId();
$newSubscription = UserSubscription::create($userId, $tariffId, $period, $previousId);
```

## Критерии готовности

- [x] Создан immutable класс `UserSubscriptionId`
- [x] Использует `Uuid` из Shared Context
- [x] Реализованы все методы
- [x] Валидация UUID при создании
- [x] Написаны unit-тесты
- [x] Код соответствует PSR-12

## Статус: ✅ ВЫПОЛНЕНО

## Связанные задачи

- 0036: UserSubscription Aggregate Root
