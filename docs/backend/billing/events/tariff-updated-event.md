# TariffUpdated Event

## Описание

Событие `TariffUpdated` генерируется при изменении параметров тарифа или его опций.

## Когда выбрасывается

- При обновлении названия, приоритета или цены тарифа
- При добавлении/удалении/изменении опции тарифа

## Структура события

```php
{
    eventId: string (UUID),
    aggregateId: string (tariffId),
    aggregateType: 'Tariff',
    eventName: 'TariffUpdated',
    occurredAt: DateTimeImmutable,
    payload: {
        tariffId: string,
        code: string,
        name: string,
        priority: int,
        price: decimal,
        options: [
            {
                optionId: string,
                name: string,
                code: string,
                type: string,
                value: mixed
            }
        ]
    }
}
```

## Основные методы

### static create(Tariff $tariff): self

Создает событие из агрегата Tariff.

**Параметры**:
- `$tariff` — объект Tariff с актуальным состоянием

---

### Методы доступа

```php
public function getTariffId(): string
public function getTariffCode(): string
public function getTariffName(): string
public function getPriority(): int
public function getPrice(): decimal
public function getOptions(): array
```

## Обработчики

Обработчики подписываются на обновления тарифов для синхронизации информации в других контекстах.

## Статус реализации

- [ ] Класс создан с базовой структурой
- [ ] Наследование от DomainEventInterface установлено
- [ ] Метод create реализован
- [ ] Все методы доступа реализованы
- [ ] Payload правильно структурирован
- [ ] Unit тесты написаны (5+ тестов)
- [ ] Integration тесты пройдены
- [ ] Код соответствует стилю проекта
- [ ] Документация актуальна

## Связанные сущности

- [Tariff Aggregate Root](../models/tariff-aggregate-root.md)
- [Events Overview](overview.md)
- [Billing Context Overview](../overview.md)
