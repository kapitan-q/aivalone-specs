# Задача 0034: Tariff Aggregate Root (Domain Model)

## Описание

Создать доменную модель `Tariff` — AggregateRoot для управления тарифами и их опциями.

## Фаза

**Phase 1: Foundation**

## Спецификация

📄 [tariff-aggregate-root.md](../backend/billing/models/tariff-aggregate-root.md)

## Зависимости

- ✅ `AggregateRoot` (Shared Context) — задача 0008
- ⏳ `TariffId` Value Object — задача 0037
- ⏳ `TariffOption` Value Object — задача 0035
- ⏳ `Tariff` Enum (Shared Context) — задача 0039
- ⏳ `TariffOptionType` Enum (Shared Context) — задача 0040

## Расположение файла

```
src/Context/Billing/Domain/Model/Tariff.php
```

## Требования к реализации

### Атрибуты

| Атрибут | Тип | Описание |
| ------- | --- | -------- |
| id | TariffId | Уникальный идентификатор тарифа |
| code | Tariff (enum) | Код тарифа (FREE, BASE, PRO, ENTERPRISE) |
| name | string | Название тарифа |
| priority | int | Приоритет (чем выше, тем важнее) |
| price | float | Цена тарифа |
| options | Collection\<TariffOption\> | Опции тарифа |

### Методы

| Метод | Описание |
| ----- | -------- |
| `create(code, name, priority, price): self` | Фабричный метод создания |
| `getId(): TariffId` | Получить идентификатор |
| `getCode(): Tariff` | Получить код тарифа |
| `getName(): string` | Получить название |
| `getPriority(): int` | Получить приоритет |
| `getPrice(): float` | Получить цену |
| `getOptions(): Collection` | Получить все опции |
| `getOption(code): ?TariffOption` | Получить опцию по коду |
| `addOption(name, code, type, value): void` | Добавить опцию |
| `updateOption(code, value): void` | Обновить значение опции |
| `removeOption(code): void` | Удалить опцию |
| `hasOption(code): bool` | Проверить наличие опции |

### Инварианты

- TariffId генерируется при создании
- Коды опций уникальны в рамках тарифа
- При изменениях генерируется событие `TariffUpdated`

## Пример использования

```php
$tariff = Tariff::create(
    TariffEnum::FREE,
    'Free Plan',
    0,
    0.00
);

$tariff->addOption(
    'Max Groups',
    'MAX_GROUPS',
    TariffOptionType::MAX_CONSTRAINT,
    5
);

$tariff->updateOption('MAX_GROUPS', 10);

$events = $tariff->pullEvents(); // [TariffUpdated, ...]
```

## Критерии готовности

- [x] Создан класс `Tariff` наследующий `AggregateRoot`
- [x] Реализованы все атрибуты и методы
- [x] Реализована работа с коллекцией `TariffOption`
- [x] Генерируется событие `TariffUpdated` при изменениях
- [x] Написаны unit-тесты
- [x] Код соответствует PSR-12

## Статус: ✅ ВЫПОЛНЕНО

## Связанные задачи

- 0035: TariffOption Value Object
- 0037: TariffId Value Object
- 0039: Tariff Enum (Shared)
- 0040: TariffOptionType Enum (Shared)
- 0042: Domain Events
