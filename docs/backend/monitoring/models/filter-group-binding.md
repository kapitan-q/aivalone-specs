# FilterGroupBinding (Entity)

## Описание

**FilterGroupBinding** — сущность, представляющая связь между фильтром (Filter) и группой (MonitoredGroup). Реализует связь Many-to-Many: один фильтр может быть привязан к нескольким группам, одна группа может иметь несколько фильтров.

## Связи

* **Принадлежит**: MonitoringProfile
* **Связывает**: Filter и MonitoredGroup

## Поля

| Поле | Тип | Описание | Обязательное |
| ---- | --- | -------- | ------------ |
| `filterId` | FilterId | Идентификатор фильтра | Да |
| `groupId` | GroupId | Идентификатор группы | Да |
| `createdAt` | DateTime | Дата создания связи | Да |

## Инварианты

1. **Уникальность пары** — (filterId, groupId) уникальна в рамках профиля
2. **Filter существует** — filterId должен ссылаться на существующий фильтр в профиле
3. **Group существует** — groupId должен ссылаться на существующую группу в профиле
4. **Один профиль** — Filter и Group должны принадлежать одному профилю

## Псевдокод

```pseudocode
CLASS FilterGroupBinding

    // Поля
    PRIVATE filterId: FilterId
    PRIVATE groupId: GroupId
    PRIVATE createdAt: DateTime

    // Создание новой связи
    STATIC FUNCTION create(
        filterId: FilterId,
        groupId: GroupId
    ): FilterGroupBinding

    IMPLEMENTATION:
        binding = NEW FilterGroupBinding()
        binding.filterId = filterId
        binding.groupId = groupId
        binding.createdAt = DateTime.now()

        RETURN binding
    END IMPLEMENTATION

    // Восстановление из persistence
    STATIC FUNCTION restore(
        filterId: FilterId,
        groupId: GroupId,
        createdAt: DateTime
    ): FilterGroupBinding

    IMPLEMENTATION:
        binding = NEW FilterGroupBinding()
        binding.filterId = filterId
        binding.groupId = groupId
        binding.createdAt = createdAt

        RETURN binding
    END IMPLEMENTATION

    // Геттеры
    FUNCTION getFilterId(): FilterId
        RETURN this.filterId
    END FUNCTION

    FUNCTION getGroupId(): GroupId
        RETURN this.groupId
    END FUNCTION

    FUNCTION getCreatedAt(): DateTime
        RETURN this.createdAt
    END FUNCTION

    // Проверка равенства (по паре filterId + groupId)
    FUNCTION equals(other: FilterGroupBinding): bool
        RETURN this.filterId.equals(other.filterId)
           AND this.groupId.equals(other.groupId)
    END FUNCTION

    // Проверка принадлежности к фильтру
    FUNCTION belongsToFilter(filterId: FilterId): bool
        RETURN this.filterId.equals(filterId)
    END FUNCTION

    // Проверка принадлежности к группе
    FUNCTION belongsToGroup(groupId: GroupId): bool
        RETURN this.groupId.equals(groupId)
    END FUNCTION

END CLASS
```

## Использование в MonitoringProfile

```pseudocode
CLASS MonitoringProfile

    PRIVATE filterGroupBindings: FilterGroupBinding[]

    // Привязать фильтр к группе
    FUNCTION bindFilterToGroup(filterId: FilterId, groupId: GroupId): void

    PRECONDITIONS:
        - this.hasFilter(filterId)
        - this.hasGroup(groupId)
        - NOT this.hasBinding(filterId, groupId)

    IMPLEMENTATION:
        binding = FilterGroupBinding.create(filterId, groupId)
        this.filterGroupBindings.add(binding)

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
            .filter(b => b.belongsToGroup(groupId))
            .map(b => b.getFilterId())

        RETURN this.filters.filter(f => filterIds.contains(f.getId()))
    END IMPLEMENTATION

    // Получить все группы для фильтра
    FUNCTION getGroupsForFilter(filterId: FilterId): MonitoredGroup[]
    IMPLEMENTATION:
        groupIds = this.filterGroupBindings
            .filter(b => b.belongsToFilter(filterId))
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

    // Удалить все связи фильтра (при удалении фильтра)
    FUNCTION removeAllBindingsForFilter(filterId: FilterId): void
        this.filterGroupBindings = this.filterGroupBindings
            .filter(b => NOT b.belongsToFilter(filterId))
    END FUNCTION

    // Удалить все связи группы (при удалении группы)
    FUNCTION removeAllBindingsForGroup(groupId: GroupId): void
        this.filterGroupBindings = this.filterGroupBindings
            .filter(b => NOT b.belongsToGroup(groupId))
    END FUNCTION

END CLASS
```

## События

| Событие | Когда генерируется |
| ------- | ------------------ |
| `FilterBoundToGroup` | При создании связи |
| `FilterUnboundFromGroup` | При удалении связи |

## Исключения

| Исключение | Когда выбрасывается |
| ---------- | ------------------- |
| `BindingAlreadyExistsException` | Связь уже существует |
| `BindingNotFoundException` | Связь не найдена при удалении |
| `FilterNotFoundException` | Фильтр не найден в профиле |
| `GroupNotFoundException` | Группа не найдена в профиле |

## Пример использования

```pseudocode
// Создание фильтра и группы
filter = profile.addFilter("bitcoin", FilterType.KEYWORD)
group = profile.addGroup(
    externalGroupId: "cryptonews",
    groupTitle: "Crypto News",
    messengerType: MessengerType.TELEGRAM,
    isPrivate: false
)

// Привязка фильтра к группе
profile.bindFilterToGroup(filter.getId(), group.getId())

// Получение фильтров для группы
filters = profile.getFiltersForGroup(group.getId())

// Отвязка фильтра от группы
profile.unbindFilterFromGroup(filter.getId(), group.getId())

// При удалении фильтра связи удаляются автоматически
profile.removeFilter(filter.getId())
// → автоматически вызывается removeAllBindingsForFilter()

// При удалении группы связи удаляются автоматически
profile.removeGroup(group.getId())
// → автоматически вызывается removeAllBindingsForGroup()
```

## Контрольные точки реализации

- [x] Создан класс FilterGroupBinding с полями
- [x] Реализован статический метод create()
- [x] Реализован статический метод restore()
- [x] Реализованы геттеры
- [x] Реализован метод equals()
- [x] Реализованы методы belongsToFilter() и belongsToGroup()
- [ ] Реализованы методы в MonitoringProfile для управления связями
- [x] Созданы события FilterBoundToGroup, FilterUnboundFromGroup
- [x] Созданы исключения
- [x] Написаны unit-тесты

## Связанные документы

* [Filter Entity](filter.md)
* [MonitoredGroup Entity](monitored-group.md)
* [MonitoringProfile AggregateRoot](monitoring-profile.md)
* [Events Overview](../events/overview.md)
