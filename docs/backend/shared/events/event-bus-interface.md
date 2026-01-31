# EventBusInterface Specification

## Назначение

**EventBusInterface** — контракт для публикации доменных событий в системе.
Обеспечивает слабую связанность между контекстами и поддержку event-driven архитектуры.

## Интерфейс

Интерфейс определяет метод, который публикует одно или несколько доменных событий
`function publish(DomainEventInterface ...$events): void`

## Паттерн использования

1. Domain Model ([AggregateRoot](../models/aggregate-root.md)) записывает события через `recordEvent()`
2. Application Service сохраняет агрегат
3. Application Service забирает события через `pullEvents()`
4. Application Service публикует события через `EventBus::publish()`

## Связанные документы

- [DomainEventInterface](./domain-event-interface.md)
- [AggregateRoot](../models/aggregate-root.md)
- [Shared Context Overview](../overview.md)

## Статус

[x] Интерфейс EventBusInterface определён
[x] Реализация EventBusInterface создана
