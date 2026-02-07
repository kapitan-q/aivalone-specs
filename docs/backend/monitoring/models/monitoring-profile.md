# MonitoringProfile (AggregateRoot)

## Описание

**MonitoringProfile** — корневой агрегат контекста Monitoring. Представляет профиль мониторинга пользователя и инкапсулирует все связанные сущности: фильтры, сессии авторизации, группы на прослушивание и связи между фильтрами и группами.

`MonitoringProfile` является **AggregateRoot** — единственная точка входа для модификации всех дочерних сущностей.

## Поля

| Поле | Тип | Описание | Обязательное |
| ---- | --- | -------- | ------------ |
| `id` | MonitoringProfileId | Уникальный идентификатор профиля (UUID) | Да |
| `userId` | UserId | Идентификатор пользователя (из Account Context) | Да |
| `isActive` | bool | Активен ли профиль мониторинга | Да |
| `filters` | Filter[] | Коллекция фильтров пользователя | Да |
| `sessions` | MonitoringSession[] | Коллекция сессий авторизации | Да |
| `groups` | MonitoredGroup[] | Коллекция групп на прослушивание | Да |
| `filterGroupBindings` | FilterGroupBinding[] | Связи фильтров с группами | Да |
| `createdAt` | DateTime | Дата создания | Да |
| `updatedAt` | DateTime | Дата последнего обновления | Да |

## Конструктор

```pseudocode
CLASS MonitoringProfile EXTENDS AggregateRoot

    // Создание нового профиля
    STATIC FUNCTION create(userId: UserId): MonitoringProfile

    IMPLEMENTATION:
        profile = NEW MonitoringProfile()
        profile.id = MonitoringProfileId.generate()
        profile.userId = userId
        profile.isActive = true
        profile.filters = []
        profile.sessions = []
        profile.groups = []
        profile.filterGroupBindings = []
        profile.createdAt = DateTime.now()
        profile.updatedAt = DateTime.now()

        profile.recordEvent(ProfileCreated(
            profileId: profile.id,
            userId: userId
        ))

        RETURN profile
    END IMPLEMENTATION

    // Восстановление из persistence
    STATIC FUNCTION restore(
        id: MonitoringProfileId,
        userId: UserId,
        isActive: bool,
        filters: Filter[],
        sessions: MonitoringSession[],
        groups: MonitoredGroup[],
        filterGroupBindings: FilterGroupBinding[],
        createdAt: DateTime,
        updatedAt: DateTime
    ): MonitoringProfile

    IMPLEMENTATION:
        profile = NEW MonitoringProfile()
        profile.id = id
        profile.userId = userId
        profile.isActive = isActive
        profile.filters = filters
        profile.sessions = sessions
        profile.groups = groups
        profile.filterGroupBindings = filterGroupBindings
        profile.createdAt = createdAt
        profile.updatedAt = updatedAt

        RETURN profile
    END IMPLEMENTATION

END CLASS
```

## Методы

### Геттеры

```pseudocode
FUNCTION getId(): MonitoringProfileId
FUNCTION getUserId(): UserId
FUNCTION isActive(): bool
FUNCTION getFilters(): Filter[]
FUNCTION getSessions(): MonitoringSession[]
FUNCTION getGroups(): MonitoredGroup[]
FUNCTION getFilterGroupBindings(): FilterGroupBinding[]
FUNCTION getCreatedAt(): DateTime
FUNCTION getUpdatedAt(): DateTime
```

### Управление статусом профиля

```pseudocode
FUNCTION activate(): void

PRECONDITIONS:
    - this.isActive = false

IMPLEMENTATION:
    this.isActive = true
    this.updatedAt = DateTime.now()

    this.recordEvent(ProfileActivated(
        profileId: this.id
    ))
END IMPLEMENTATION


FUNCTION deactivate(): void

PRECONDITIONS:
    - this.isActive = true

IMPLEMENTATION:
    // Деактивируем все сессии
    FOR EACH session IN this.sessions
        IF session.isActive THEN
            session.deactivate()
        END IF
    END FOR

    this.isActive = false
    this.updatedAt = DateTime.now()

    this.recordEvent(ProfileDeactivated(
        profileId: this.id
    ))
END IMPLEMENTATION
```

### Управление фильтрами

```pseudocode
// Добавить фильтр
FUNCTION addFilter(
    value: string,
    filterType: FilterType,
    name: string | null = null
): Filter

PRECONDITIONS:
    - this.isActive = true
    - NOT this.hasFilterWithValue(value, filterType)

IMPLEMENTATION:
    filter = Filter.create(value, filterType, name)
    this.filters.add(filter)
    this.updatedAt = DateTime.now()

    RETURN filter
END IMPLEMENTATION


// Получить фильтр по ID
FUNCTION getFilter(filterId: FilterId): Filter

IMPLEMENTATION:
    filter = this.filters.find(f => f.getId().equals(filterId))
    IF filter IS NULL THEN
        THROW FilterNotFoundException(filterId)
    END IF
    RETURN filter
END IMPLEMENTATION


// Проверка наличия фильтра с таким значением
FUNCTION hasFilterWithValue(value: string, filterType: FilterType): bool
    RETURN this.filters.any(
        f => f.getValue() = value AND f.getFilterType() = filterType
    )
END FUNCTION


// Получить активные фильтры
FUNCTION getActiveFilters(): Filter[]
    RETURN this.filters.filter(f => f.isActive())
END FUNCTION


// Удалить фильтр
FUNCTION removeFilter(filterId: FilterId): void

PRECONDITIONS:
    - this.hasFilter(filterId)

IMPLEMENTATION:
    filter = this.getFilter(filterId)

    // Удаляем все связи этого фильтра с группами
    this.removeAllBindingsForFilter(filterId)

    this.filters.remove(filter)
    this.updatedAt = DateTime.now()

    this.recordEvent(FilterDeleted(
        profileId: this.id,
        filterId: filterId
    ))
END IMPLEMENTATION


// Проверка наличия фильтра
FUNCTION hasFilter(filterId: FilterId): bool
    RETURN this.filters.any(f => f.getId().equals(filterId))
END FUNCTION


// Подсчёт фильтров
FUNCTION getFiltersCount(): int
    RETURN this.filters.length
END FUNCTION
```

### Управление сессиями

```pseudocode
// Создать сессию для авторизации
FUNCTION createSession(messengerType: MessengerType): MonitoringSession

PRECONDITIONS:
    - this.isActive = true
    - NOT this.hasSessionForMessenger(messengerType)

IMPLEMENTATION:
    session = MonitoringSession.create(messengerType)
    this.sessions.add(session)
    this.updatedAt = DateTime.now()

    RETURN session
END IMPLEMENTATION


// Получить сессию по ID
FUNCTION getSession(sessionId: SessionId): MonitoringSession

IMPLEMENTATION:
    session = this.sessions.find(s => s.getId().equals(sessionId))
    IF session IS NULL THEN
        THROW SessionNotFoundException(sessionId)
    END IF
    RETURN session
END IMPLEMENTATION


// Получить сессию по типу мессенджера
FUNCTION getSessionByMessenger(messengerType: MessengerType): MonitoringSession | null

IMPLEMENTATION:
    RETURN this.sessions.find(s => s.getMessengerType() = messengerType)
END IMPLEMENTATION


// Проверка наличия сессии для мессенджера
FUNCTION hasSessionForMessenger(messengerType: MessengerType): bool
    RETURN this.sessions.any(s => s.getMessengerType() = messengerType)
END FUNCTION


// Получить авторизованную сессию
FUNCTION getAuthorizedSession(messengerType: MessengerType): MonitoringSession | null

IMPLEMENTATION:
    session = this.getSessionByMessenger(messengerType)
    IF session IS NOT NULL AND session.isAuthorized() THEN
        RETURN session
    END IF
    RETURN null
END IMPLEMENTATION


// Удалить сессию
FUNCTION removeSession(sessionId: SessionId): void

PRECONDITIONS:
    - this.hasSession(sessionId)

IMPLEMENTATION:
    session = this.getSession(sessionId)

    // Деактивируем все приватные группы этой сессии
    FOR EACH group IN this.getGroupsForSession(sessionId)
        group.deactivateBySessionExpiry()
    END FOR

    this.sessions.remove(session)
    this.updatedAt = DateTime.now()

    this.recordEvent(SessionRemoved(
        profileId: this.id,
        sessionId: sessionId
    ))
END IMPLEMENTATION


// Проверка наличия сессии
FUNCTION hasSession(sessionId: SessionId): bool
    RETURN this.sessions.any(s => s.getId().equals(sessionId))
END FUNCTION
```

### Управление группами

```pseudocode
// Добавить публичную группу
FUNCTION addPublicGroup(
    externalGroupId: string,
    groupTitle: string,
    messengerType: MessengerType
): MonitoredGroup

PRECONDITIONS:
    - this.isActive = true
    - NOT this.hasGroupWithExternalId(externalGroupId, messengerType)

IMPLEMENTATION:
    group = MonitoredGroup.createPublic(
        externalGroupId,
        groupTitle,
        messengerType
    )
    this.groups.add(group)
    this.updatedAt = DateTime.now()

    RETURN group
END IMPLEMENTATION


// Добавить приватную группу
FUNCTION addPrivateGroup(
    externalGroupId: string,
    groupTitle: string,
    messengerType: MessengerType,
    sessionId: SessionId
): MonitoredGroup

PRECONDITIONS:
    - this.isActive = true
    - NOT this.hasGroupWithExternalId(externalGroupId, messengerType)
    - this.hasSession(sessionId)
    - this.getSession(sessionId).isAuthorized()

IMPLEMENTATION:
    group = MonitoredGroup.createPrivate(
        externalGroupId,
        groupTitle,
        messengerType,
        sessionId
    )
    this.groups.add(group)
    this.updatedAt = DateTime.now()

    RETURN group
END IMPLEMENTATION


// Получить группу по ID
FUNCTION getGroup(groupId: GroupId): MonitoredGroup

IMPLEMENTATION:
    group = this.groups.find(g => g.getId().equals(groupId))
    IF group IS NULL THEN
        THROW GroupNotFoundException(groupId)
    END IF
    RETURN group
END IMPLEMENTATION


// Получить группу по внешнему ID
FUNCTION getGroupByExternalId(
    externalGroupId: string,
    messengerType: MessengerType
): MonitoredGroup | null

IMPLEMENTATION:
    RETURN this.groups.find(
        g => g.getExternalGroupId() = externalGroupId
         AND g.getMessengerType() = messengerType
    )
END IMPLEMENTATION


// Проверка наличия группы с таким внешним ID
FUNCTION hasGroupWithExternalId(externalGroupId: string, messengerType: MessengerType): bool
    RETURN this.groups.any(
        g => g.getExternalGroupId() = externalGroupId
         AND g.getMessengerType() = messengerType
    )
END FUNCTION


// Получить группы для сессии (приватные)
FUNCTION getGroupsForSession(sessionId: SessionId): MonitoredGroup[]
    RETURN this.groups.filter(
        g => g.getSessionId() IS NOT NULL
         AND g.getSessionId().equals(sessionId)
    )
END FUNCTION


// Получить активные группы
FUNCTION getActiveGroups(): MonitoredGroup[]
    RETURN this.groups.filter(g => g.isActive())
END FUNCTION


// Удалить группу
FUNCTION removeGroup(groupId: GroupId): void

PRECONDITIONS:
    - this.hasGroup(groupId)

IMPLEMENTATION:
    group = this.getGroup(groupId)

    // Удаляем все связи этой группы с фильтрами
    this.removeAllBindingsForGroup(groupId)

    this.groups.remove(group)
    this.updatedAt = DateTime.now()

    this.recordEvent(GroupDeleted(
        profileId: this.id,
        groupId: groupId,
        messengerType: group.getMessengerType()
    ))
END IMPLEMENTATION


// Проверка наличия группы
FUNCTION hasGroup(groupId: GroupId): bool
    RETURN this.groups.any(g => g.getId().equals(groupId))
END FUNCTION


// Подсчёт групп
FUNCTION getGroupsCount(): int
    RETURN this.groups.length
END FUNCTION
```

### Управление связями фильтров и групп

```pseudocode
// Привязать фильтр к группе
FUNCTION bindFilterToGroup(filterId: FilterId, groupId: GroupId): void

PRECONDITIONS:
    - this.hasFilter(filterId)
    - this.hasGroup(groupId)
    - NOT this.hasBinding(filterId, groupId)

IMPLEMENTATION:
    binding = FilterGroupBinding.create(filterId, groupId)
    this.filterGroupBindings.add(binding)
    this.updatedAt = DateTime.now()

    this.recordEvent(FilterBoundToGroup(
        profileId: this.id,
        filterId: filterId,
        groupId: groupId
    ))
END IMPLEMENTATION


// Отвязать фильтр от группы
FUNCTION unbindFilterFromGroup(filterId: FilterId, groupId: GroupId): void

PRECONDITIONS:
    - this.hasBinding(filterId, groupId)

IMPLEMENTATION:
    binding = this.findBinding(filterId, groupId)
    this.filterGroupBindings.remove(binding)
    this.updatedAt = DateTime.now()

    this.recordEvent(FilterUnboundFromGroup(
        profileId: this.id,
        filterId: filterId,
        groupId: groupId
    ))
END IMPLEMENTATION


// Получить все фильтры для группы
FUNCTION getFiltersForGroup(groupId: GroupId): Filter[]

IMPLEMENTATION:
    filterIds = this.filterGroupBindings
        .filter(b => b.getGroupId().equals(groupId))
        .map(b => b.getFilterId())

    RETURN this.filters.filter(f => filterIds.contains(f.getId()))
END IMPLEMENTATION


// Получить активные фильтры для группы
FUNCTION getActiveFiltersForGroup(groupId: GroupId): Filter[]
    RETURN this.getFiltersForGroup(groupId).filter(f => f.isActive())
END FUNCTION


// Получить все группы для фильтра
FUNCTION getGroupsForFilter(filterId: FilterId): MonitoredGroup[]

IMPLEMENTATION:
    groupIds = this.filterGroupBindings
        .filter(b => b.getFilterId().equals(filterId))
        .map(b => b.getGroupId())

    RETURN this.groups.filter(g => groupIds.contains(g.getId()))
END IMPLEMENTATION


// Проверка существования связи
FUNCTION hasBinding(filterId: FilterId, groupId: GroupId): bool
    RETURN this.filterGroupBindings.any(
        b => b.getFilterId().equals(filterId)
         AND b.getGroupId().equals(groupId)
    )
END FUNCTION


// Найти связь
FUNCTION findBinding(filterId: FilterId, groupId: GroupId): FilterGroupBinding | null
    RETURN this.filterGroupBindings.find(
        b => b.getFilterId().equals(filterId)
         AND b.getGroupId().equals(groupId)
    )
END FUNCTION


// Удалить все связи фильтра (при удалении фильтра)
PRIVATE FUNCTION removeAllBindingsForFilter(filterId: FilterId): void
    this.filterGroupBindings = this.filterGroupBindings.filter(
        b => NOT b.getFilterId().equals(filterId)
    )
END FUNCTION


// Удалить все связи группы (при удалении группы)
PRIVATE FUNCTION removeAllBindingsForGroup(groupId: GroupId): void
    this.filterGroupBindings = this.filterGroupBindings.filter(
        b => NOT b.getGroupId().equals(groupId)
    )
END FUNCTION
```

### Подсчёт лимитов

```pseudocode
FUNCTION canAddFilter(maxFilters: int): bool
    RETURN this.getFiltersCount() < maxFilters
END FUNCTION

FUNCTION canAddGroup(maxGroups: int): bool
    RETURN this.getGroupsCount() < maxGroups
END FUNCTION

FUNCTION canUseFilterType(filterType: FilterType, allowedTypes: FilterType[]): bool
    RETURN allowedTypes.contains(filterType)
END FUNCTION
```

## Инварианты

1. **Один профиль на пользователя** — userId уникален в системе
2. **Операции только на активном профиле** — большинство операций требуют isActive=true
3. **Одна сессия на messengerType** — не может быть двух сессий одного типа
4. **Приватная группа требует авторизованную сессию** — sessionId ссылается на AUTHORIZED session
5. **Каскадная деактивация** — при деактивации профиля сессии деактивируются
6. **Каскадное удаление связей** — при удалении Filter или Group связи удаляются
7. **Уникальность группы** — (externalGroupId, messengerType) уникальна в профиле
8. **Уникальность фильтра** — (value, filterType) уникальна в профиле

## События

| Событие | Когда генерируется |
| ------- | ------------------ |
| `ProfileCreated` | При создании профиля |
| `ProfileActivated` | При активации |
| `ProfileDeactivated` | При деактивации |
| `FilterDeleted` | При удалении фильтра |
| `GroupDeleted` | При удалении группы |
| `SessionRemoved` | При удалении сессии |
| `FilterBoundToGroup` | При привязке фильтра к группе |
| `FilterUnboundFromGroup` | При отвязке фильтра от группы |

## Исключения

| Исключение | Когда выбрасывается |
| ---------- | ------------------- |
| `ProfileNotActiveException` | Операция на неактивном профиле |
| `FilterNotFoundException` | Фильтр не найден |
| `FilterAlreadyExistsException` | Дубликат фильтра (value+type) |
| `SessionNotFoundException` | Сессия не найдена |
| `SessionAlreadyExistsException` | Сессия для messengerType уже существует |
| `SessionNotAuthorizedException` | Требуется авторизованная сессия |
| `GroupNotFoundException` | Группа не найдена |
| `GroupAlreadyExistsException` | Дубликат группы (externalGroupId+messenger) |
| `BindingAlreadyExistsException` | Связь фильтра с группой уже существует |
| `BindingNotFoundException` | Связь не найдена |

## Контрольные точки реализации

- [x] Создан класс MonitoringProfile
- [x] Реализован статический метод create()
- [x] Реализован статический метод restore()
- [x] Реализованы геттеры
- [x] Реализованы методы управления статусом (activate, deactivate)
- [x] Реализованы методы управления фильтрами
- [x] Реализованы методы управления сессиями
- [x] Реализованы методы управления группами
- [x] Реализованы методы управления связями
- [x] Реализованы методы подсчёта лимитов
- [x] Созданы события
- [x] Созданы исключения
- [x] Написаны unit-тесты

## Связанные документы

* [MonitoringProfileId Value Object](monitoring-profile-id.md)
* [Filter Entity](filter.md)
* [MonitoringSession Entity](monitoring-session.md)
* [MonitoredGroup Entity](monitored-group.md)
* [FilterGroupBinding Entity](filter-group-binding.md)
* [Events Overview](../events/overview.md)
* [Exceptions Overview](../exceptions/overview.md)
* [Repository Interface](../infrastructure/monitoring-profile-repository.md)
