# MonitoredGroup (Entity)

## Описание

**MonitoredGroup** — сущность, представляющая группу/канал мессенджера, на которую подписан пользователь для мониторинга. Группа может быть публичной (прослушивается через сервисный аккаунт listener-service) или приватной (требует авторизованную сессию пользователя).

Группа является **Entity** внутри агрегата `MonitoringProfile`.

## Связи

* **Принадлежит**: MonitoringProfile
* **Опциональная связь**: MonitoringSession (для приватных групп через sessionId)
* **Связан**: Filter через FilterGroupBinding (M:N)

## Поля

| Поле | Тип | Описание | Обязательное |
| ---- | --- | -------- | ------------ |
| `id` | GroupId | Уникальный идентификатор группы (UUID) | Да |
| `externalGroupId` | string | ID группы в мессенджере (username, chat_id и т.д.) | Да |
| `groupTitle` | string | Название группы | Да |
| `messengerType` | MessengerType | Тип мессенджера | Да |
| `sessionId` | SessionId? | Ссылка на сессию (для приватных групп) | Нет |
| `isPrivate` | bool | Приватная ли группа | Да |
| `status` | GroupStatus | Статус группы | Да |
| `lastMessageAt` | DateTime? | Время последнего полученного сообщения | Нет |
| `createdAt` | DateTime | Дата создания | Да |
| `updatedAt` | DateTime | Дата последнего обновления | Да |

## Инварианты

1. **Приватная группа требует сессию** — isPrivate=true → sessionId required
2. **Публичная группа без сессии** — isPrivate=false → sessionId=null
3. **Уникальность в профиле** — (externalGroupId, messengerType) уникальна в рамках профиля
4. **Активная группа с активной сессией** — для приватных групп: status=ACTIVE → session.isAuthorized()
5. **Безопасность** — externalGroupId не передаётся в доменные события

## Конструктор

```pseudocode
CLASS MonitoredGroup

    // Создание публичной группы
    STATIC FUNCTION createPublic(
        externalGroupId: string,
        groupTitle: string,
        messengerType: MessengerType
    ): MonitoredGroup

    PRECONDITIONS:
        - externalGroupId.trim().length > 0
        - groupTitle.trim().length > 0

    IMPLEMENTATION:
        group = NEW MonitoredGroup()
        group.id = GroupId.generate()
        group.externalGroupId = externalGroupId.trim()
        group.groupTitle = groupTitle.trim()
        group.messengerType = messengerType
        group.sessionId = null
        group.isPrivate = false
        group.status = GroupStatus.PENDING
        group.lastMessageAt = null
        group.createdAt = DateTime.now()
        group.updatedAt = DateTime.now()

        group.recordEvent(GroupCreated(
            groupId: group.id,
            groupTitle: group.groupTitle,
            messengerType: messengerType,
            isPrivate: false
            // externalGroupId НЕ передаётся в событие
        ))

        RETURN group
    END IMPLEMENTATION

    // Создание приватной группы
    STATIC FUNCTION createPrivate(
        externalGroupId: string,
        groupTitle: string,
        messengerType: MessengerType,
        sessionId: SessionId
    ): MonitoredGroup

    PRECONDITIONS:
        - externalGroupId.trim().length > 0
        - groupTitle.trim().length > 0
        - sessionId IS NOT NULL

    IMPLEMENTATION:
        group = NEW MonitoredGroup()
        group.id = GroupId.generate()
        group.externalGroupId = externalGroupId.trim()
        group.groupTitle = groupTitle.trim()
        group.messengerType = messengerType
        group.sessionId = sessionId
        group.isPrivate = true
        group.status = GroupStatus.PENDING
        group.lastMessageAt = null
        group.createdAt = DateTime.now()
        group.updatedAt = DateTime.now()

        group.recordEvent(GroupCreated(
            groupId: group.id,
            groupTitle: group.groupTitle,
            messengerType: messengerType,
            isPrivate: true,
            sessionId: sessionId
            // externalGroupId НЕ передаётся в событие
        ))

        RETURN group
    END IMPLEMENTATION

    // Восстановление из persistence
    STATIC FUNCTION restore(
        id: GroupId,
        externalGroupId: string,
        groupTitle: string,
        messengerType: MessengerType,
        sessionId: SessionId | null,
        isPrivate: bool,
        status: GroupStatus,
        lastMessageAt: DateTime | null,
        createdAt: DateTime,
        updatedAt: DateTime
    ): MonitoredGroup

    IMPLEMENTATION:
        group = NEW MonitoredGroup()
        group.id = id
        group.externalGroupId = externalGroupId
        group.groupTitle = groupTitle
        group.messengerType = messengerType
        group.sessionId = sessionId
        group.isPrivate = isPrivate
        group.status = status
        group.lastMessageAt = lastMessageAt
        group.createdAt = createdAt
        group.updatedAt = updatedAt

        RETURN group
    END IMPLEMENTATION

END CLASS
```

## Методы

### Геттеры

```pseudocode
FUNCTION getId(): GroupId
    RETURN this.id
END FUNCTION

FUNCTION getExternalGroupId(): string
    RETURN this.externalGroupId
END FUNCTION

FUNCTION getGroupTitle(): string
    RETURN this.groupTitle
END FUNCTION

FUNCTION getMessengerType(): MessengerType
    RETURN this.messengerType
END FUNCTION

FUNCTION getSessionId(): SessionId | null
    RETURN this.sessionId
END FUNCTION

FUNCTION isPrivate(): bool
    RETURN this.isPrivate
END FUNCTION

FUNCTION getStatus(): GroupStatus
    RETURN this.status
END FUNCTION

FUNCTION getLastMessageAt(): DateTime | null
    RETURN this.lastMessageAt
END FUNCTION

FUNCTION getCreatedAt(): DateTime
    RETURN this.createdAt
END FUNCTION

FUNCTION getUpdatedAt(): DateTime
    RETURN this.updatedAt
END FUNCTION
```

### Проверки состояния

```pseudocode
FUNCTION isActive(): bool
    RETURN this.status = GroupStatus.ACTIVE
END FUNCTION

FUNCTION isPending(): bool
    RETURN this.status = GroupStatus.PENDING
END FUNCTION

FUNCTION isFailed(): bool
    RETURN this.status = GroupStatus.FAILED
END FUNCTION

FUNCTION canReceiveMessages(): bool
    RETURN this.status = GroupStatus.ACTIVE
END FUNCTION

FUNCTION requiresSession(): bool
    RETURN this.isPrivate
END FUNCTION

FUNCTION hasSession(): bool
    RETURN this.sessionId IS NOT NULL
END FUNCTION
```

### Управление статусом

```pseudocode
// Активация группы (получили событие GroupJoined от listener-service)
FUNCTION activate(): void

PRECONDITIONS:
    - this.status = GroupStatus.PENDING

IMPLEMENTATION:
    this.status = GroupStatus.ACTIVE
    this.updatedAt = DateTime.now()

    this.recordEvent(GroupActivated(
        groupId: this.id,
        messengerType: this.messengerType
    ))
END IMPLEMENTATION


// Ошибка присоединения (получили событие GroupJoinFailed от listener-service)
FUNCTION markAsFailed(reason: string): void

PRECONDITIONS:
    - this.status = GroupStatus.PENDING

IMPLEMENTATION:
    this.status = GroupStatus.FAILED
    this.updatedAt = DateTime.now()

    this.recordEvent(GroupJoinFailed(
        groupId: this.id,
        messengerType: this.messengerType,
        reason: reason
    ))
END IMPLEMENTATION


// Деактивация группы (при истечении сессии для приватных групп)
FUNCTION deactivateBySessionExpiry(): void

PRECONDITIONS:
    - this.isPrivate = true
    - this.status = GroupStatus.ACTIVE

IMPLEMENTATION:
    this.status = GroupStatus.FAILED
    this.updatedAt = DateTime.now()

    this.recordEvent(GroupDeactivated(
        groupId: this.id,
        messengerType: this.messengerType,
        reason: "session_expired"
    ))
END IMPLEMENTATION
```

### Обновление данных

```pseudocode
FUNCTION updateGroupTitle(newTitle: string): void

PRECONDITIONS:
    - newTitle.trim().length > 0
    - newTitle != this.groupTitle

IMPLEMENTATION:
    this.groupTitle = newTitle.trim()
    this.updatedAt = DateTime.now()
END IMPLEMENTATION


FUNCTION recordMessageReceived(): void

IMPLEMENTATION:
    this.lastMessageAt = DateTime.now()
END IMPLEMENTATION
```

## Публичные vs Приватные группы

```
┌─────────────────────────────────────────────────────────────────────────┐
│                         Публичная группа                                │
│                         isPrivate = false                               │
├─────────────────────────────────────────────────────────────────────────┤
│  • sessionId = null                                                     │
│  • Прослушивается через сервисный аккаунт listener-service              │
│  • Не требует авторизации пользователя                                  │
│  • externalGroupId = username, invite link                              │
│  • Примеры: @cryptonews, https://t.me/publicgroup                       │
└─────────────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────────────┐
│                         Приватная группа                                │
│                         isPrivate = true                                │
├─────────────────────────────────────────────────────────────────────────┤
│  • sessionId = required (AUTHORIZED session)                            │
│  • Прослушивается через аккаунт пользователя                            │
│  • Требует авторизованную сессию                                        │
│  • externalGroupId = chat_id, internal identifier                       │
│  • Примеры: приватные чаты, закрытые группы                             │
└─────────────────────────────────────────────────────────────────────────┘
```

## Диаграмма состояний

```
              add()
                │
                ▼
           ┌─────────┐
           │ PENDING │
           └────┬────┘
                │
      ┌─────────┴─────────┐
      │                   │
  GroupJoined       GroupJoinFailed
      │                   │
      ▼                   ▼
 ┌─────────┐         ┌─────────┐
 │ ACTIVE  │         │ FAILED  │
 └────┬────┘         └─────────┘
      │                   │
      │ session_expired   │
      │ (private only)    │
      └───────────────────┘
              │
              ▼
         [delete]
```

## События

| Событие | Когда генерируется |
| ------- | ------------------ |
| `GroupCreated` | При создании группы |
| `GroupActivated` | При успешном присоединении |
| `GroupJoinFailed` | При ошибке присоединения |
| `GroupDeactivated` | При деактивации (session expired) |
| `GroupDeleted` | При удалении (генерируется в Profile) |

## Исключения

| Исключение | Когда выбрасывается |
| ---------- | ------------------- |
| `InvalidGroupStateException` | Недопустимый переход состояния |
| `GroupRequiresSessionException` | Приватная группа без sessionId |
| `GroupAlreadyExistsException` | Дубликат (externalGroupId, messengerType) |
| `GroupLimitReachedException` | Превышен лимит групп |

## Пример использования

```pseudocode
// Добавление публичной группы
publicGroup = MonitoredGroup.createPublic(
    externalGroupId: "@cryptonews",
    groupTitle: "Crypto News Channel",
    messengerType: MessengerType.TELEGRAM
)
// publicGroup.status = PENDING
// publicGroup.sessionId = null

// Добавление приватной группы (требует авторизованную сессию)
session = profile.getSession(MessengerType.TELEGRAM)
IF session.isAuthorized() THEN
    privateGroup = MonitoredGroup.createPrivate(
        externalGroupId: "123456789",
        groupTitle: "Private Chat",
        messengerType: MessengerType.TELEGRAM,
        sessionId: session.getId()
    )
    // privateGroup.status = PENDING
    // privateGroup.sessionId = session.getId()
END IF

// При получении события GroupJoined от listener-service
publicGroup.activate()
// publicGroup.status = ACTIVE

// При получении события GroupJoinFailed
privateGroup.markAsFailed("INVITE_HASH_INVALID")
// privateGroup.status = FAILED
```

## Контрольные точки реализации

- [x] Создан класс MonitoredGroup с полями
- [x] Реализован статический метод createPublic()
- [x] Реализован статический метод createPrivate()
- [x] Реализован статический метод restore()
- [x] Реализованы геттеры
- [x] Реализованы методы проверки состояния
- [x] Реализован метод activate()
- [x] Реализован метод markAsFailed()
- [x] Реализован метод deactivateBySessionExpiry()
- [x] Реализованы методы обновления данных
- [x] Созданы события
- [x] Созданы исключения
- [x] Написаны unit-тесты

## Связанные документы

* [GroupId Value Object](group-id.md)
* [GroupStatus Enum](group-status-enum.md)
* [MonitoringProfile AggregateRoot](monitoring-profile.md)
* [MonitoringSession Entity](monitoring-session.md)
* [FilterGroupBinding Entity](filter-group-binding.md)
* [Events Overview](../events/overview.md)
* [Subscribe to Group Process](../processes/subscribe-to-group.md)
