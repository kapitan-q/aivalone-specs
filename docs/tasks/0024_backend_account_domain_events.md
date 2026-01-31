# Задача 0024: Реализация событий Account Context

## Описание

Реализовать три доменных события для Account Context:
- `UserRegistered`
- `UserMessengersUpdated`
- `UserTariffsUpdated`

## Требования

- [x] Все события наследуют `DomainEventInterface` из Shared Context
- [x] Все события immutable
- [x] Все события содержат только сериализуемые данные

## Событие 1: UserRegistered

**Назначение**: Публикуется при создании нового пользователя

**Поля события**:
```php
class UserRegistered implements DomainEventInterface
{
    private UserId $userId;
    /**
     * @var UserMessengers[]
     */
    private array $messengers;
    /**
     * @var Tariff[]
     */
    private array $tariffs;
    private DateTimeImmutable $occurredAt;
}
```

**Методы интерфейса**:
- `getEventName(): string` → `'user.registered'`
- `getOccurredAt(): DateTimeImmutable` → дата события
- `getPayload(): array` → сериализуемые данные (мессенджеры и тарифы конвертируются в скалярные значения)

**Payload пример**:
```json
{
  "userId": "550e8400-e29b-41d4-a716-446655440000",
  "messengers": [
    {"messenger": "TELEGRAM", "messengerId": "123456"}
  ],
  "tariffs": ["FREE"]
}
```

**Примечание**: В конструкторе события принимаются объекты UserMessengers и Tariff, но в `getPayload()` они преобразуются.

## Событие 2: UserMessengersUpdated

**Назначение**: Публикуется при добавлении или удалении мессенджера

**Поля события**:
```php
class UserMessengersUpdated implements DomainEventInterface
{
    private UserId $userId;
    /**
     * @var UserMessengers[]
     */
    private array $messengers;
    private DateTimeImmutable $occurredAt;
}
```

**Методы интерфейса**:
- `getEventName(): string` → `'user.messengers.updated'`
- `getOccurredAt(): DateTimeImmutable` → дата события
- `getPayload(): array` → сериализуемые данные

**Payload пример**:
```json
{
  "userId": "550e8400-e29b-41d4-a716-446655440000",
  "messengers": [
    {"messenger": "TELEGRAM", "messengerId": "123456"},
    {"messenger": "WHATSAPP", "messengerId": "wa_123"}
  ]
}
```

**Примечание**: В конструкторе события принимаются объекты UserMessengers, но в `getPayload()` они преобразуются в массив.

## Событие 3: UserTariffsUpdated

**Назначение**: Публикуется при изменении тарифов пользователя

**Поля события**:
```php
class UserTariffsUpdated implements DomainEventInterface
{
    private UserId $userId;
    /**
     * @var Tariff[]
     */
    private array $tariffs;
    private DateTimeImmutable $occurredAt;
}
```

**Методы интерфейса**:
- `getEventName(): string` → `'user.tariffs.updated'`
- `getOccurredAt(): DateTimeImmutable` → дата события
- `getPayload(): array` → сериализуемые данные (тарифы конвертируются в строки)

**Payload пример**:
```json
{
  "userId": "550e8400-e29b-41d4-a716-446655440000",
  "tariffs": ["FREE", "BASE"]
}
```

**Примечание**: В конструкторе события принимаются объекты Tariff, но в `getPayload()` они преобразуются в строковые коды.

## Требования для всех событий

- [x] Классы расположены в `src/Context/Account/Domain/Event/`
- [x] Все события immutable (final классы, private поля)
- [x] Конструктор передает все параметры
- [x] `getOccurredAt()` возвращает время события (обычно текущее время)
- [x] `getPayload()` возвращает только сериализуемые данные (строки, массивы, числа, null)
- [x] UserId преобразуется в строку в payload

## Критерии готовности

- [x] Все три события реализованы
- [x] Покрыты unit-тестами (минимум 2 теста на каждое)
- [x] Документация актуальна

## Зависимости

- Shared Context: DomainEventInterface
- Shared Context: Tariff
- Account Context: UserId
- Shared Context: Messenger

## Примечания

- События должны быть создаваемы только через конструктор
- События должны быть сериализуемы в JSON
- События не должны содержать domain объекты (только primitives)
- Дата события может быть передана в конструкторе или установлена как текущее время

## Статус

done
