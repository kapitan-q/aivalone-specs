# Задача 01: Реализация DomainException

## Описание

Реализовать базовое исключение `DomainException` для сигнализации об ошибках бизнес-логики и нарушениях инвариантов в доменной модели.

## Требования

- [x] Класс должен наследоваться от `\Exception`
- [x] Должен быть расположен в `src/Context/Shared/Domain/Exception/DomainException.php`
- [x] Может быть унаследовано для конкретных случаев
- [x] Должен поддерживать стандартные параметры Exception (message, code, previous)

## Реализация

```php
namespace App\Context\Shared\Domain\Exception;

class DomainException extends \Exception
{
    // базовое исключение без специфической логики
}
```

## Использование

```php
throw new DomainException("Нарушение бизнес-правила");
```

## Критерии готовности

- [x] Класс реализован
- [ ] Покрыт unit-тестами
- [ ] Документация актуальна

## Зависимости

Нет

## Статус

done
