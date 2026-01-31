# Tariff Enum Specification

## Назначение

**Tariff** — enum для типобезопасного представления тарифных планов в системе Aivalone.
Используется во всех контекстах для унификации работы с тарифами.

## Расположение

```
src/Context/Shared/Domain/Model/Tariff.php
```

## Поддерживаемые значения

| Значение    | Код            | Описание           | Статус    |
|-------------|----------------|--------------------|-----------|
| FREE        | `FREE`         | Бесплатный тариф   | Активен   |
| BASE        | `BASE`         | Базовый тариф      | Активен   |
| PRO         | `PRO`          | Профессиональный   | Активен   |
| ENTERPRISE  | `ENTERPRISE`   | Корпоративный      | План      |

## Интерфейс

```php
namespace App\Context\Shared\Domain\Model;

enum Tariff: string
{
    case FREE = 'FREE';
    case BASE = 'BASE';
    case PRO = 'PRO';
    case ENTERPRISE = 'ENTERPRISE';

    public static function fromCode(string $code): self;

    public function code(): string;

    public function equals(Tariff $other): bool;
}
```

## Методы

### `fromCode(string $code): self`

Создание экземпляра из строкового кода.

- **Параметры**: `$code` — строковый код тарифа (например, 'FREE')
- **Возвращает**: Экземпляр `Tariff`
- **Исключения**: `TariffValidationException` при невалидном коде

### `code(): string`

Возвращает строковый код тарифа.

- **Возвращает**: Строковый код (например, 'FREE')

### `equals(Tariff $other): bool`

Сравнивает два тарифа на равенство.

- **Параметры**: `$other` — другой тариф для сравнения
- **Возвращает**: `true` если тарифы равны

## Использование

```php
// Создание из кода
$free = Tariff::fromCode('FREE');

// Получение кода
echo $free->code(); // 'FREE'

// Сравнение
$free->equals(Tariff::FREE); // true

// Прямое использование enum case
$pro = Tariff::PRO;
```

## В контексте User

Пользователь может иметь несколько активных тарифов одновременно.
Тарифы хранятся как массив строковых кодов в БД и конвертируются в объекты Tariff при загрузке.

```php
// В User Model
private array $tariffs = []; // array<Tariff>

public function replaceTariffs(array $tariffs): void
{
    $this->tariffs = $tariffs;
    $this->recordEvent(new UserTariffsUpdated($this->id, $tariffs));
}
```

## Связанные документы

- [Models Overview](./overview.md)
- [User Model](../../account/model/user.md)
- [UpdateUserTariffs Process](../../account/processes/update-user-tariffs.md)
- [TariffValidationException](../exceptions/tariff-validation-exception.md)

## Статус

[x] Enum реализован
[x] Методы fromCode, code, equals реализованы
[x] Интегрирован с User Model
