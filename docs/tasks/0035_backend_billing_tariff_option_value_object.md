# Задача 0035: TariffOption Value Object

## Описание

Создать Value Object `TariffOption` — опция (ограничение) тарифа.

## Фаза

**Phase 1: Foundation**

## Спецификация

📄 [tariff-option-value-object.md](../backend/billing/models/tariff-option-value-object.md)

## Зависимости

- ⏳ `TariffOptionType` Enum (Shared Context) — задача 0040

## Расположение файла

```
src/Context/Billing/Domain/Model/TariffOption.php
```

## Требования к реализации

### Атрибуты

| Атрибут | Тип | Описание |
| ------- | --- | -------- |
| name | string | Человеко-читаемое название опции |
| code | string | Уникальный код опции (например, MAX_GROUPS) |
| type | TariffOptionType | Тип опции (MAX_CONSTRAINT, BOOL, TEXT) |
| value | mixed | Значение опции (int, bool, string) |

### Методы

| Метод | Описание |
| ----- | -------- |
| `__construct(name, code, type, value)` | Конструктор |
| `getName(): string` | Получить название |
| `getCode(): string` | Получить код |
| `getType(): TariffOptionType` | Получить тип |
| `getValue(): mixed` | Получить значение |
| `withValue(value): self` | Создать копию с новым значением |
| `equals(other): bool` | Сравнение по значению |

### Инварианты

- Value Object immutable (неизменяемый)
- Код опции не пустой
- Значение соответствует типу опции

## Пример использования

```php
$option = new TariffOption(
    'Max Groups',
    'MAX_GROUPS',
    TariffOptionType::MAX_CONSTRAINT,
    5
);

$newOption = $option->withValue(10);

$option->equals($newOption); // false
$option->getCode(); // 'MAX_GROUPS'
```

## Критерии готовности

- [x] Создан immutable класс `TariffOption`
- [x] Реализованы все методы
- [x] Метод `withValue()` возвращает новый экземпляр
- [x] Написаны unit-тесты
- [x] Код соответствует PSR-12

## Статус: ✅ ВЫПОЛНЕНО

## Связанные задачи

- 0034: Tariff Aggregate Root
- 0040: TariffOptionType Enum
