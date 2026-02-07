# Задача 0087: MonitoringProfile AggregateRoot

## Контекст

`MonitoringProfile` — корневой агрегат Monitoring Context. Единственная точка входа для всех операций с фильтрами, сессиями, группами и связями. Инкапсулирует все бизнес-правила и инварианты домена.

## Цель

Создать AggregateRoot `MonitoringProfile` с полной логикой управления дочерними сущностями.

## Спецификация

- [MonitoringProfile](../backend/monitoring/models/monitoring-profile.md)
- [Models Overview](../backend/monitoring/models/overview.md)

## Файлы для создания

```
src/Context/Monitoring/Domain/Model/MonitoringProfile.php
tests/Unit/Context/Monitoring/Domain/Model/MonitoringProfileTest.php
```

## Требования

### Поля

| Поле | Тип | Описание |
| ---- | --- | -------- |
| `id` | MonitoringProfileId | Уникальный идентификатор |
| `userId` | UserId | Идентификатор пользователя |
| `isActive` | bool | Активен ли профиль |
| `filters` | Filter[] | Коллекция фильтров |
| `sessions` | MonitoringSession[] | Коллекция сессий |
| `groups` | MonitoredGroup[] | Коллекция групп |
| `filterGroupBindings` | FilterGroupBinding[] | Связи фильтр-группа |
| `createdAt` | DateTimeImmutable | Дата создания |
| `updatedAt` | DateTimeImmutable | Дата обновления |

### Методы — Управление профилем

- `static create(userId)` — создание нового профиля (isActive=true)
- `static restore(...)` — восстановление из persistence
- `activate()` / `deactivate()` — управление активностью
- Деактивация каскадно деактивирует все сессии

### Методы — Управление фильтрами

- `addFilter(Filter $filter)` — добавление фильтра
- `getFilter(filterId)` — получение по ID
- `hasFilterWithValue(value, filterType)` — проверка дубликата
- `getActiveFilters()` — получение активных
- `removeFilter(filterId)` — удаление (с каскадным удалением bindings)
- `getFiltersCount()` — количество
- `canAddFilter(maxFilters)` — проверка лимита
- `canUseFilterType(filterType, allowedTypes)` — проверка типа

### Методы — Управление сессиями

- `createSession(messengerType)` — создание сессии
- `getSession(sessionId)` — получение по ID
- `getSessionByMessenger(messengerType)` — получение по типу мессенджера
- `hasSessionForMessenger(messengerType)` — проверка наличия
- `getAuthorizedSession(messengerType)` — получение авторизованной сессии
- `removeSession(sessionId)` — удаление (деактивация приватных групп)

### Методы — Управление группами

- `addGroup(MonitoredGroup $group)` - добавление группы
- `getGroup(groupId)` — получение по ID
- `getGroupByExternalId(externalGroupId, messengerType)` — поиск по внешнему ID
- `hasGroupWithExternalId(externalGroupId, messengerType)` — проверка дубликата
- `getGroupsForSession(sessionId)` — группы для сессии
- `getActiveGroups()` — активные группы
- `removeGroup(groupId)` — удаление (с каскадным удалением bindings)
- `getGroupsCount()` — количество
- `canAddGroup(maxGroups)` — проверка лимита

### Методы — Управление связями

- `bindFilterToGroup(Filter $filter, MonitoredGroup $group)` — привязка
- `unbindFilterFromGroup(Filter $filter, MonitoredGroup $group)` — отвязка
- `getFiltersForGroup(groupId)` — фильтры для группы
- `getActiveFiltersForGroup(groupId)` — активные фильтры для группы
- `getGroupsForFilter(filterId)` — группы для фильтра
- `hasBinding(filterId, groupId)` — проверка связи

### Инварианты

1. Один профиль на пользователя (userId уникален)
2. Операции только на активном профиле
3. Одна сессия на messengerType
4. Приватная группа требует авторизованную сессию
5. Каскадная деактивация при деактивации профиля
6. Каскадное удаление bindings при удалении filter/group
7. Уникальность (externalGroupId, messengerType) в профиле
8. Уникальность (value, filterType) в профиле

## Тесты

### Profile

- [x] Создание профиля с активным состоянием
- [x] activate() / deactivate() работают корректно
- [x] Деактивация каскадно деактивирует сессии
- [x] Операции на неактивном профиле выбрасывают исключение

### Filters

- [x] Добавление фильтра
- [x] Исключение при дубликате value+type
- [x] Удаление фильтра каскадно удаляет bindings
- [x] canAddFilter() корректно проверяет лимит

### Sessions

- [x] Создание сессии
- [x] Исключение при дубликате messengerType
- [x] Получение авторизованной сессии

### Groups

- [x] Добавление публичной группы
- [x] Добавление приватной группы с авторизованной сессией
- [x] Исключение при добавлении приватной группы без сессии
- [x] Удаление группы каскадно удаляет bindings

### Bindings

- [x] Привязка фильтра к группе
- [x] Отвязка фильтра от группы
- [x] Исключение при дубликате связи
- [x] getFiltersForGroup() / getGroupsForFilter() работают корректно

## Зависимости

- Filter (задача 0083)
- MonitoringSession (задача 0084)
- MonitoredGroup (задача 0085)
- FilterGroupBinding (задача 0086)
- Value Objects (задача 0077)
- `AggregateRoot` (Shared Context) — задача 0008
- `UserId` (Account Context) — задача 0017
- Domain Events (задача 0082)
- Exceptions (задача 0081)

## Definition of Done

- [x] Класс MonitoringProfile реализован со всеми методами
- [x] Все инварианты соблюдены
- [x] Каскадные операции работают корректно
- [x] Unit-тесты написаны и проходят (минимум 20 тестов)
- [x] Код соответствует PSR-12
- [x] PHPStan level 9 без ошибок
