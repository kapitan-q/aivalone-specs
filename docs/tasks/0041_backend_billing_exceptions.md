# Задача 0041: Billing Domain Exceptions

## Описание

Создать все доменные исключения для Billing Context.

## Фаза

**Phase 2: Exceptions**

## Спецификация

📄 [exceptions/overview.md](../backend/billing/exceptions/overview.md)

## Зависимости

- ✅ `DomainException` (Shared Context) — задача 0001
- ✅ `ValidationException` (Shared Context) — задача 0002

## Расположение файлов

```
src/Context/Billing/Domain/Exception/
├── TariffNotFoundException.php
├── InvalidSubscriptionException.php
├── DuplicateSubscriptionException.php
├── SubscriptionNotFoundException.php
├── InvalidTariffCodeException.php
├── InvalidOptionTypeException.php
└── TariffOptionNotFoundException.php
```

---

## Исключения

### 1. TariffNotFoundException

**Наследует**: `DomainException`

**Когда выбрасывается**: Тариф с указанным ID или кодом не найден

```php
class TariffNotFoundException extends DomainException
{
    public static function withId(TariffId $id): self;
    public static function withCode(Tariff $code): self;
}

// Использование
throw TariffNotFoundException::withCode(Tariff::PRO);
```

---

### 2. InvalidSubscriptionException

**Наследует**: `DomainException`

**Когда выбрасывается**: Попытка выполнить недопустимую операцию над подпиской

```php
class InvalidSubscriptionException extends DomainException
{
    public static function alreadyExpired(UserSubscriptionId $id): self;
    public static function alreadyCancelled(UserSubscriptionId $id): self;
    public static function cannotExpirePermanent(UserSubscriptionId $id): self;
}

// Использование
throw InvalidSubscriptionException::alreadyExpired($subscriptionId);
```

---

### 3. DuplicateSubscriptionException

**Наследует**: `DomainException`

**Когда выбрасывается**: Пользователь уже имеет активную подписку на этот тариф

```php
class DuplicateSubscriptionException extends DomainException
{
    public static function forUserAndTariff(UserId $userId, TariffId $tariffId): self;
}

// Использование
throw DuplicateSubscriptionException::forUserAndTariff($userId, $tariffId);
```

---

### 4. SubscriptionNotFoundException

**Наследует**: `DomainException`

**Когда выбрасывается**: Подписка не найдена

```php
class SubscriptionNotFoundException extends DomainException
{
    public static function withId(UserSubscriptionId $id): self;
    public static function forUserAndTariff(UserId $userId, TariffId $tariffId): self;
}

// Использование
throw SubscriptionNotFoundException::withId($subscriptionId);
```

---

### 5. InvalidTariffCodeException

**Наследует**: `ValidationException`

**Когда выбрасывается**: Невалидный код тарифа

```php
class InvalidTariffCodeException extends ValidationException
{
    public static function fromCode(string $code): self;
}

// Использование
throw InvalidTariffCodeException::fromCode('invalid_code');
```

---

### 6. InvalidOptionTypeException

**Наследует**: `ValidationException`

**Когда выбрасывается**: Невалидный тип опции тарифа

```php
class InvalidOptionTypeException extends ValidationException
{
    public static function fromType(string $type): self;
}

// Использование
throw InvalidOptionTypeException::fromType('unknown_type');
```

---

### 7. TariffOptionNotFoundException

**Наследует**: `DomainException`

**Когда выбрасывается**: Опция тарифа не найдена

```php
class TariffOptionNotFoundException extends DomainException
{
    public static function withCode(string $code, TariffId $tariffId): self;
}

// Использование
throw TariffOptionNotFoundException::withCode('MAX_GROUPS', $tariffId);
```

---

## Критерии готовности

- [x] Созданы все 7 классов исключений
- [x] Каждый класс наследует соответствующий базовый класс
- [x] Реализованы статические фабричные методы
- [x] Сообщения об ошибках информативны
- [ ] Написаны unit-тесты
- [x] Код соответствует PSR-12

## Статус: ✅ ВЫПОЛНЕНО

## Связанные задачи

- 0034: Tariff Aggregate Root
- 0036: UserSubscription Aggregate Root
- 0046: SubscriptionService
