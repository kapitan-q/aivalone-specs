# SubscriptionPeriod Enum

## Описание

Enum `SubscriptionPeriod` определяет период действия подписки. Используется при создании и продлении подписок.

## Расположение

```
src/Context/Billing/Domain/Enum/SubscriptionPeriod.php
```

## Поддерживаемые значения

| Значение | Код     | Описание              | Длительность |
|----------|---------|----------------------|--------------|
| MONTH    | `MONTH` | Месячная подписка    | 30 дней      |
| YEAR     | `YEAR`  | Годовая подписка     | 365 дней     |

## Интерфейс

```php
namespace App\Context\Billing\Domain\Enum;

enum SubscriptionPeriod: string
{
    case MONTH = 'MONTH';
    case YEAR = 'YEAR';

    public static function fromCode(string $code): self;

    public function code(): string;

    public function toDays(): int;

    public function toDateInterval(): \DateInterval;

    public function equals(SubscriptionPeriod $other): bool;
}
```

## Методы

### `fromCode(string $code): self`

Создание экземпляра из строкового кода.

- **Параметры**: `$code` — строковый код периода (MONTH, YEAR)
- **Возвращает**: Экземпляр `SubscriptionPeriod`
- **Исключения**: `InvalidSubscriptionPeriodException` при невалидном коде

### `code(): string`

Возвращает строковый код периода.

- **Возвращает**: Строковый код (MONTH или YEAR)

### `toDays(): int`

Возвращает количество дней в периоде.

- **Возвращает**: `30` для MONTH, `365` для YEAR

### `toDateInterval(): \DateInterval`

Возвращает DateInterval для расчёта даты окончания.

- **Возвращает**: `new \DateInterval('P1M')` для MONTH, `new \DateInterval('P1Y')` для YEAR

### `equals(SubscriptionPeriod $other): bool`

Сравнивает два периода на равенство.

- **Параметры**: `$other` — другой период для сравнения
- **Возвращает**: `true` если периоды равны

## Использование

```php
// Создание из кода
$period = SubscriptionPeriod::fromCode('MONTH');

// Получение кода
echo $period->code(); // 'MONTH'

// Расчёт даты окончания
$validUntil = (new \DateTimeImmutable())->add($period->toDateInterval());

// Прямое использование enum case
$yearPeriod = SubscriptionPeriod::YEAR;
echo $yearPeriod->toDays(); // 365
```

## Статус реализации

- [ ] Enum создан
- [ ] Все методы реализованы
- [ ] Unit тесты написаны (5+ тестов)
- [ ] Документация актуальна

## Связанные документы

- [UserSubscription Aggregate Root](../models/user-subscription-aggregate-root.md)
- [RenewSubscription Process](../processes/renew-subscription-process.md)
- [Enums Overview](../enums/subscription-status.md)
