# UserSubscriptionId Value Object

## Описание

Value object `UserSubscriptionId` инкапсулирует UUID идентификатор пользовательской подписки. 
Используется для типобезопасной работы с идентификаторами подписок и предотвращения ошибок.
Наследник `UUID` из Shared Context

## Основные методы

### static fromString(string $id): self

Создание UserSubscriptionId из строки.

**Логика**:
- Валидирует, что строка является валидным UUID
- Выбрасывает `InvalidUuidException` если невалидный формат
- Возвращает новый объект UserSubscriptionId

**Параметры**:
- `$id` — строка с UUID

**Выбрасывает**: InvalidUuidException (из Shared Context)

### static generate(): self

Генерирует новый уникальный UserSubscriptionId.

**Логика**:
- Генерирует новый UUID версии 4
- Возвращает новый объект UserSubscriptionId

### toString(): string

Возвращает строковое представление UserSubscriptionId.

### equals(UserSubscriptionId $other): bool

Сравнивает два UserSubscriptionId.

## Статус реализации

- [ ] Класс создан с базовой структурой
- [ ] Методы fromString, generate, toString, equals реализованы
- [ ] Валидация UUID работает корректно
- [ ] Unit тесты написаны (5+ тестов)
- [ ] Integration тесты пройдены
- [ ] Код соответствует стилю проекта
- [ ] Документация актуальна

## Связанные сущности

- [UserSubscription Aggregate Root](user-subscription-aggregate-root.md)
- [Billing Context Overview](../overview.md)
