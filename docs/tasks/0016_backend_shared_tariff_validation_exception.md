# Задача 0016: Реализация TariffValidationException

## Описание

Реализовать специализированное исключение `TariffValidationException` для обработки ошибок валидации тарифов на уровне Application Service (Command Handler) при конвертации строковых кодов в объекты Tariff.

## Требования

- [x] Класс должен наследоваться от `ValidationException` (Shared Context)
- [x] Должен быть расположен в `src/Context/Shared/Domain/Exception/TariffValidationException.php`
- [x] Выбрасывается при попытке конвертировать строковый код в некорректный Tariff объект
- [x] Должен содержать static методы для создания с различными контекстами

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

```php
// В UpdateUserTariffsCommandHandler при конвертации строк в Tariff объекты
if (empty($command->tariffCodes)) {
    throw TariffValidationException::emptyList();
}

foreach ($command->tariffCodes as $code) {
    try {
        $tariff = Tariff::fromCode($code);
        $tariffs[] = $tariff;
    } catch (InvalidArgumentException $e) {
        throw TariffValidationException::invalidCode($code);
    }
}

// Дальше передаем валидные объекты Tariff в UserService
$user = $this->userService->updateTariffs($userId, $tariffs);
```

## Критерии готовности

- [x] Класс реализован с методами `invalidCode()` и `emptyList()`
- [ ] Покрыт unit-тестами
- [ ] Документация актуальна

## Зависимости

- Shared Context: ValidationException
- Shared Context: Tariff

## Статус

done
