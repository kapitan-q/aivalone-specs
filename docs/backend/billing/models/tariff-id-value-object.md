# TariffId Value Object

## Описание

Value object `TariffId` инкапсулирует UUID идентификатор тарифа. Используется для типобезопасной работы с идентификаторами и предотвращения ошибок.

Наследник `UUID` из Shared Context

## Основные методы

### static fromString(string $id): self

Создание TariffId из строки.

**Логика**:
- Валидирует, что строка является валидным UUID
- Выбрасывает `InvalidUuidException` если невалидный формат (Shared Context)
- Возвращает новый объект TariffId

**Параметры**:
- `$id` — строка с UUID

**Выбрасывает**: InvalidUuidException (из Shared Context)

### static generate(): self

Генерирует новый уникальный TariffId.

**Логика**:
- Генерирует новый UUID версии 4
- Возвращает новый объект TariffId

### toString(): string

Возвращает строковое представление TariffId.

### equals(TariffId $other): bool

Сравнивает два TariffId.

## Статус реализации

- [ ] Класс создан с базовой структурой
- [ ] Методы fromString, generate, toString, equals реализованы
- [ ] Валидация UUID работает корректно
- [ ] Unit тесты написаны (5+ тестов)
- [ ] Integration тесты пройдены
- [ ] Код соответствует стилю проекта
- [ ] Документация актуальна

## Связанные сущности

- [Tariff Aggregate Root](tariff-aggregate-root.md)
- [Billing Context Overview](../overview.md)
- [UUID](../../shared/models/uuid.md)