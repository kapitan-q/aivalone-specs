# Задача 0083: Filter Entity

## Контекст

`Filter` — сущность внутри агрегата `MonitoringProfile`, представляющая пользовательский фильтр для мониторинга сообщений. Фильтр может быть ключевым словом или регулярным выражением.

## Цель

Создать Entity `Filter` с полной логикой создания, редактирования, активации/деактивации и валидации.

## Спецификация

- [Filter Entity](../backend/monitoring/models/filter.md)

## Файлы для создания

```
src/Context/Monitoring/Domain/Model/Filter.php
tests/Unit/Context/Monitoring/Domain/Model/FilterTest.php
```

## Требования

### Поля

| Поле | Тип | Описание |
| ---- | --- | -------- |
| `id` | FilterId | Уникальный идентификатор |
| `value` | string | Значение фильтра (keyword или regex pattern) |
| `filterType` | FilterType | Тип фильтра |
| `name` | string? | Опциональное название |
| `isActive` | bool | Активен ли фильтр |
| `createdAt` | DateTimeImmutable | Дата создания |
| `updatedAt` | DateTimeImmutable | Дата обновления |

### Методы

- `static create(value, filterType, name?)` — создание нового фильтра с валидацией
- `static restore(...)` — восстановление из persistence
- Геттеры для всех полей
- `updateValue(newValue)` — изменение значения с валидацией
- `changeFilterType(newType)` — изменение типа с проверкой совместимости
- `updateName(newName)` — изменение названия
- `activate()` / `deactivate()` — управление активностью

### Валидация

- `value` не пустой, минимум 3 символа
- `value` не превышает maxLength для данного filterType
- Если filterType=REGEX — value должен быть валидным regex

### События

- `FilterCreated` — при создании
- `FilterValueUpdated` — при изменении значения
- `FilterTypeChanged` — при изменении типа
- `FilterActivated` / `FilterDeactivated` — при изменении активности

## Тесты

- [x] Создание фильтра типа KEYWORD
- [x] Создание фильтра типа REGEX
- [x] Исключение при пустом value
- [x] Исключение при слишком длинном value
- [x] Исключение при невалидном regex pattern
- [x] updateValue() корректно обновляет значение
- [x] updateValue() выбрасывает исключение при невалидном regex
- [x] changeFilterType() корректно меняет тип
- [x] activate() / deactivate() работают корректно
- [x] Исключение при повторной активации/деактивации
- [x] События генерируются правильно
- [x] restore() не генерирует события

## Зависимости

- FilterId (задача 0077)
- FilterType (задача 0080)
- Domain Events (задача 0082)
- Exceptions (задача 0081)

## Definition of Done

- [x] Класс Filter реализован со всеми методами
- [x] Валидация value и regex работает
- [x] События генерируются при изменениях
- [x] Unit-тесты написаны и проходят
- [x] Код соответствует PSR-12
- [x] PHPStan level 9 без ошибок
