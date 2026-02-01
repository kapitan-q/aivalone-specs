# Задача 0037: TariffId Value Object

## Описание

Создать Value Object `TariffId` — типизированный идентификатор тарифа на основе UUID.

## Фаза

**Phase 1: Foundation**

## Спецификация

📄 [tariff-id-value-object.md](../backend/billing/models/tariff-id-value-object.md)

## Зависимости

- ✅ `Uuid` (Shared Context) — задача 0005
- ✅ `InvalidUuidException` (Shared Context) — задача 0003

## Расположение файла

```
src/Context/Billing/Domain/Model/TariffId.php
```

## Требования к реализации

### Методы

| Метод | Описание |
| ----- | -------- |
| `generate(): self` | Создать новый TariffId с UUID v4 |
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
$id = TariffId::generate();

// Создание из строки
$id = TariffId::fromString('550e8400-e29b-41d4-a716-446655440000');

// Использование
$idString = $id->toString();
$id->equals($anotherId); // bool
```

## Критерии готовности

- [x] Создан immutable класс `TariffId`
- [x] Использует `Uuid` из Shared Context
- [x] Реализованы все методы
- [x] Валидация UUID при создании
- [x] Написаны unit-тесты
- [x] Код соответствует PSR-12

## Статус: ✅ ВЫПОЛНЕНО

## Связанные задачи

- 0034: Tariff Aggregate Root
- 0036: UserSubscription Aggregate Root
