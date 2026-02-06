# Задача 0080: FilterType Enum (Shared Context)

## Контекст

Monitoring Context требует типы фильтров для классификации пользовательских фильтров. FilterType — общий enum, размещаемый в Shared Context, так как может использоваться Filtering Context и другими контекстами.

## Цель

Создать enum `FilterType` в Shared Context.

## Спецификация

- [Filter Entity — FilterType](../backend/monitoring/models/filter.md)

## Файлы для создания

```
src/Context/Shared/Domain/Enum/FilterType.php
tests/Unit/Context/Shared/Domain/Enum/FilterTypeTest.php
```

## Требования

```php
<?php

declare(strict_types=1);

namespace App\Context\Shared\Domain\Enum;

enum FilterType: string
{
    case KEYWORD = 'keyword';
    case REGEX = 'regex';
    // Post-MVP:
    // case PHRASE = 'phrase';
    // case WILDCARD = 'wildcard';

    public function maxLength(): int
    {
        return match ($this) {
            self::KEYWORD => 100,
            self::REGEX => 500,
        };
    }
}
```

## Тесты

- [x] Все значения enum существуют (KEYWORD, REGEX)
- [x] maxLength() возвращает 100 для KEYWORD
- [x] maxLength() возвращает 500 для REGEX
- [x] Создание из строки (from) работает корректно
- [x] Исключение при невалидном значении

## Зависимости

Нет

## Definition of Done

- [x] Enum FilterType создан
- [x] Unit-тесты написаны и проходят
- [x] Код соответствует PSR-12
- [x] PHPStan level 9 без ошибок
