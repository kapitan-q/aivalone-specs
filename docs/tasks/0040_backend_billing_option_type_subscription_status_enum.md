# Задача 0040: TariffOptionType, SubscriptionStatus, SubscriptionPeriod Enums

## Описание

Создать Enum'ы для Billing Context:
- `TariffOptionType` (Shared Context) — типы опций тарифа
- `SubscriptionStatus` (Billing Context) — статусы подписок
- `SubscriptionPeriod` (Billing Context) — периоды подписок

## Фаза

**Phase 1: Foundation**

## Спецификации

- 📄 [tariff-option-type.md](../backend/shared/models/tariff-option-type.md)
- 📄 [subscription-status.md](../backend/billing/enums/subscription-status.md)
- 📄 [subscription-period.md](../backend/billing/enums/subscription-period.md)

## Зависимости

Нет зависимостей

## Расположение файлов

```
src/Context/Shared/Domain/Enum/TariffOptionType.php
src/Context/Billing/Domain/Enum/SubscriptionStatus.php
src/Context/Billing/Domain/Enum/SubscriptionPeriod.php
```

---

## 1. TariffOptionType Enum (Shared Context)

### Значения

| Значение | Code | Описание |
| -------- | ---- | -------- |
| MAX_CONSTRAINT | `max_constraint` | Числовое ограничение (максимум) |
| BOOL | `bool` | Булево значение (включено/выключено) |
| TEXT | `text` | Текстовое значение |

### Методы

| Метод | Описание |
| ----- | -------- |
| `code(): string` | Получить строковый код |
| `fromCode(string): self` | Создать из строки |

### Пример

```php
$type = TariffOptionType::MAX_CONSTRAINT;
$type->code(); // 'max_constraint'
```

---

## 2. SubscriptionStatus Enum (Billing Context)

### Значения

| Значение | Code | Описание |
| -------- | ---- | -------- |
| ACTIVE | `active` | Подписка активна |
| EXPIRED | `expired` | Подписка истекла |
| CANCELLED | `cancelled` | Подписка отменена |

### Методы

| Метод | Описание |
| ----- | -------- |
| `code(): string` | Получить строковый код |
| `fromCode(string): self` | Создать из строки |
| `isActive(): bool` | Проверить активна ли |

### Пример

```php
$status = SubscriptionStatus::ACTIVE;
$status->isActive(); // true

$status = SubscriptionStatus::fromCode('expired');
$status->isActive(); // false
```

---

## 3. SubscriptionPeriod Enum (Billing Context)

### Значения

| Значение | Code | Days | Описание |
| -------- | ---- | ---- | -------- |
| MONTH | `month` | 30 | Месячная подписка |
| YEAR | `year` | 365 | Годовая подписка |

### Методы

| Метод | Описание |
| ----- | -------- |
| `code(): string` | Получить строковый код |
| `fromCode(string): self` | Создать из строки |
| `toDays(): int` | Получить количество дней |
| `toDateInterval(): DateInterval` | Получить DateInterval |
| `equals(self): bool` | Сравнение |

### Пример

```php
$period = SubscriptionPeriod::MONTH;
$period->toDays(); // 30
$period->toDateInterval(); // DateInterval('P1M')

// Расчёт validUntil
$validUntil = (new \DateTimeImmutable())->add($period->toDateInterval());
```

---

## Критерии готовности

- [x] Создан `TariffOptionType` enum в Shared Context
- [x] Создан `SubscriptionStatus` enum в Billing Context
- [x] Создан `SubscriptionPeriod` enum в Billing Context
- [x] Все enum'ы имеют методы `code()` и `fromCode()`
- [x] `SubscriptionPeriod` имеет методы `toDays()` и `toDateInterval()`
- [x] Написаны unit-тесты для каждого enum
- [x] Код соответствует PSR-12

## Статус: ✅ ВЫПОЛНЕНО

## Связанные задачи

- 0034: Tariff Aggregate Root
- 0035: TariffOption Value Object
- 0036: UserSubscription Aggregate Root
