# TariffOption Value Object

## Описание

Value object `TariffOption` представляет опцию (ограничение) тарифа. Не является отдельным AggregateRoot — это часть агрегата Tariff и не может существовать без него.

## Атрибуты

- `id: TariffOptionId` — уникальный идентификатор в контексте тарифа (UUID)
- `name: string` — название опции (60 символов максимум)
- `code: string` — код опции (MAX_GROUPS, MAX_CHANNELS и т.д., 30 символов, [A-Z0-9_])
- `type: TariffOptionType` — тип ограничения (enum: MAX_CONSTRAINT, BOOL, TEXT)
- `value: mixed` — значение опции (число, boolean или строка в зависимости от type)

## Основные методы

### static create(string $name, string $code, TariffOptionType $type, mixed $value): self

Создание новой опции.

**Логика**:
- Генерирует новый optionId (UUID)
- Валидирует все параметры
- Возвращает новый объект TariffOption

**Валидация**:
- `$name` не пусто и не больше 60 символов
- `$code` не пусто, не больше 30 символов, только [A-Z0-9_]
- `$value` валидно для переданного `$type`:
  - MAX_CONSTRAINT: целое число > 0
  - BOOL: boolean
  - TEXT: непустая строка, максимум 255 символов

**Выбрасывает**: ValidationException

**Возвращает**: Новый объект `TariffOption`

### updateValue(mixed $newValue): void

Обновляет значение опции.

**Логика**:
- Валидирует новое значение согласно типу
- Обновляет значение

**Параметры**:
- `$newValue` — новое значение

**Выбрасывает**: ValidationException если значение невалидно для типа

### Методы доступа

```php
public function getId(): string
public function getName(): string
public function getCode(): string
public function getType(): TariffOptionType
public function getValue(): mixed
```

## Статус реализации

- [ ] Класс создан с базовой структурой
- [ ] Методы create и updateValue реализованы
- [ ] Все валидации работают корректно
- [ ] Методы доступа реализованы
- [ ] Unit тесты написаны (10+ тестов)
- [ ] Integration тесты пройдены
- [ ] Код соответствует стилю проекта
- [ ] Документация актуальна

## Связанные сущности

- [Tariff Aggregate Root](tariff-aggregate-root.md)
- [TariffOptionType Enum](../../shared/models/tariff-option-type.md)
- [Billing Context Overview](../overview.md)
