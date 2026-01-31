# События Shared Context

## Назначение

В данном разделе описаны доменные события, используемые в Shared Context для интеграции между bounded-контекстами и построения event-driven архитектуры.

## Компоненты

- [DomainEventInterface](./domain-event-interface.md) — базовый контракт для событий
- [EventBusInterface](./event-bus-interface.md) — контракт для публикации событий

## Архитектура

```
Domain Model (Aggregate Root)
    │
    ├─ recordEvent(DomainEventInterface)
    │
    └─ pullEvents(): array
           │
           ↓
Application Service (UserService)
    │
    └─ eventBus->publish(...$events)
           │
           ↓
EventBusInterface
    │
    └─ SymfonyEventBus (реализация)
           │
           ↓
Symfony Messenger (async transport)
```

## Связанные документы

- [Shared Context Overview](../overview.md)
- [AggregateRoot](../models/aggregate-root.md)