# Tariff Validation Exception

## Назначение

Специализированное исключение `TariffValidationException` для обработки ошибок валидации тарифов при работе с Tariff enum. 

Используется в контекстах (например, Account Context), когда необходимо конвертировать строковые коды в объекты Tariff и валидировать их корректность.

## Реализация

```php
namespace App\Context\Shared\Domain\Exception;

class TariffValidationException extends ValidationException
{
    public static function invalidCode(string $code): self
    {
        return new self(sprintf('Некорректный код тарифа "%s"', $code));
    }
    
    public static function emptyList(): self
    {
        return new self('Список тарифов не может быть пустым');
    }
}
```

## Использование

### В Account Context (UpdateUserTariffsCommandHandler)

```php
use App\Context\Shared\Domain\Exception\TariffValidationException;
use App\Context\Shared\Domain\Model\Tariff;

class UpdateUserTariffsCommandHandler
{
    public function handle(UpdateUserTariffsCommand $command): void
    {
        // Валидация на входе
        if (empty($command->tariffCodes)) {
            throw TariffValidationException::emptyList();
        }

        // Конвертация строк в Tariff объекты
        $tariffs = [];
        foreach ($command->tariffCodes as $code) {
            try {
                $tariff = Tariff::fromCode($code);
                $tariffs[] = $tariff;
            } catch (InvalidArgumentException $e) {
                throw TariffValidationException::invalidCode($code);
            }
        }

        // Дальше работаем с валидными Tariff объектами
        $user = $this->userService->updateTariffs($userId, $tariffs);
        // ...
    }
}
```

## Иерархия исключений

```
Exception (PHP)
  └─ DomainException (Shared)
      └─ ValidationException (Shared)
          └─ TariffValidationException (Shared)
```

## Критерии готовности

- [x] Класс реализован с методами `invalidCode()` и `emptyList()`
- [x] Наследуется от `ValidationException`
- [x] Расположен в Shared Context
- [ ] Покрыт unit-тестами
- [ ] Документация актуальна

## Зависимости

- ValidationException
- Tariff enum

## Примечания

- Исключение выбрасывается на уровне Application Service при конвертации строковых кодов в Tariff объекты
- Валидация enum значений происходит внутри `Tariff::fromCode()`, а TariffValidationException оборачивает это как более специфичное исключение
- Используется только для валидации Tariff, другие validation ошибки используют ValidationException напрямую
