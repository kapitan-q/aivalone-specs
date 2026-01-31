# Обзор контекста Shared

## Назначение

**Shared Context** содержит общие элементы системы Aivalone, которые используются несколькими bounded-контекстами.  
Этот контекст не содержит бизнес-логики конкретного домена и служит единым источником правды для переиспользуемых типов и контрактов.

Основные цели:

* исключить дублирование кода между контекстами
* обеспечить типобезопасность и единые правила валидации
* зафиксировать общие интерфейсы и базовые соглашения системы


## Архитектурные ограничения

* Shared **не знает** о других контекстах (Account, Billing, Bot и т.д.)
* Контексты могут зависеть от Shared, но не наоборот
* В Shared запрещено:
  * бизнес-логика конкретного контекста
  * Application-процессы
  * инфраструктурные реализации, завязанные на конкретный домен

---

## Структура контекста (файловая)

```
src/Context/Shared/
├── Domain/
│   ├── Model/              # Value Objects, Enums (UUID, Messenger, Tariff)
│   ├── Event/              # DomainEventInterface, AggregateRoot
│   └── Exception/          # Базовые исключения (DomainException, ValidationException и др.)
├── Application/
│   └── Event/              # EventBusInterface
├── Infrastructure/
│   └── Event/              # SymfonyEventBus (реализация EventBusInterface)
└── Presentation/
    └── Http/
        └── Controller/     # HealthCheckController
```

## Структура Shared Context

- [Модели](./models/overview.md) — доменные сущности, value object и enum'ы, используемые в разных контекстах.
- [События](./events/overview.md) — базовые интерфейсы и спецификации событий.
- [Исключения](./exceptions/overview.md) — типовые исключения для валидации и защиты домена.
- [Права и permissions](./permissions.md) — система проверки прав (в разработке).

## Связанные документы

- [Backend Overview](../overview.md)

## Статус реализации

* [x] Domain Layer (UUID, Messenger, Tariff, AggregateRoot, DomainEventInterface, Exceptions)
* [x] Application Layer (EventBusInterface)
* [x] Infrastructure Layer (SymfonyEventBus)
* [x] Presentation Layer (HealthCheckController)
* [ ] Unit-тесты