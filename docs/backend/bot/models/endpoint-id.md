# EndpointId Value Object Specification

## Назначение

`EndpointId` — Value Object для уникальной идентификации `NotificationEndpoint`. Основан на UUID v4.

## Реализация

Расширяет базовый класс [Uuid](../../backend/shared/models/uuid.md) из Shared Context.

```php
final class EndpointId extends Uuid
{
    // Наследует всю функциональность от Uuid
}
```

## Методы

Наследуются от Uuid:

* `static generate(): self` — генерация нового UUID
* `static fromString(string $value): self` — создание из строки
* `getValue(): string` — получение значения
* `equals(Uuid $other): bool` — сравнение
* `__toString(): string` — строковое представление

## Валидация

* Формат UUID v4 проверяется при создании
* При невалидном формате выбрасывается `InvalidUuidException`

## Связанные документы

* [Uuid Value Object](../../backend/shared/models/uuid.md)
* [NotificationEndpoint](notification-endpoint.md)

## Статус реализации

* [ ] Класс EndpointId создан (extends Uuid)
* [ ] Unit тесты написаны
