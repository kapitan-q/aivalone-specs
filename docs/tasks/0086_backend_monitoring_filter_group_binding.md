# Задача 0086: FilterGroupBinding Entity

## Контекст

`FilterGroupBinding` — сущность, реализующая Many-to-Many связь между `Filter` и `MonitoredGroup`. Определяет, какие фильтры работают в каких группах.

## Цель

Создать Entity `FilterGroupBinding`.

## Спецификация

- [FilterGroupBinding Entity](../backend/monitoring/models/filter-group-binding.md)

## Файлы для создания

```
src/Context/Monitoring/Domain/Model/FilterGroupBinding.php
tests/Unit/Context/Monitoring/Domain/Model/FilterGroupBindingTest.php
```

## Требования

### Поля

| Поле | Тип | Описание |
| ---- | --- | -------- |
| `filterId` | FilterId | Идентификатор фильтра |
| `groupId` | GroupId | Идентификатор группы |
| `createdAt` | DateTimeImmutable | Дата создания связи |

### Методы

- `static create(filterId, groupId)` — создание новой связи
- `static restore(filterId, groupId, createdAt)` — восстановление
- `getFilterId()`, `getGroupId()`, `getCreatedAt()` — геттеры
- `equals(other)` — сравнение по паре (filterId, groupId)
- `belongsToFilter(filterId)` — проверка принадлежности к фильтру
- `belongsToGroup(groupId)` — проверка принадлежности к группе

## Тесты

- [x] Создание связи с корректными полями
- [x] equals() возвращает true для одинаковых пар
- [x] equals() возвращает false для разных пар
- [x] belongsToFilter() работает корректно
- [x] belongsToGroup() работает корректно
- [x] restore() восстанавливает с указанной датой

## Зависимости

- FilterId, GroupId (задача 0077)

## Definition of Done

- [x] Класс FilterGroupBinding реализован
- [x] Все методы работают корректно
- [x] Unit-тесты написаны и проходят
- [x] Код соответствует PSR-12
- [x] PHPStan level 9 без ошибок
