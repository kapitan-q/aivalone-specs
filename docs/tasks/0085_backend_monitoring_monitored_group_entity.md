# Задача 0085: MonitoredGroup Entity

## Контекст

`MonitoredGroup` — сущность внутри агрегата `MonitoringProfile`, представляющая группу/канал мессенджера для мониторинга. Группы могут быть публичными (через сервисный аккаунт) или приватными (через авторизованную сессию пользователя).

## Цель

Создать Entity `MonitoredGroup` с поддержкой публичных и приватных групп.

## Спецификация

- [MonitoredGroup Entity](../backend/monitoring/models/monitored-group.md)
- [GroupStatus Enum](../backend/monitoring/models/group-status-enum.md)

## Файлы для создания

```
src/Context/Monitoring/Domain/Model/MonitoredGroup.php
tests/Unit/Context/Monitoring/Domain/Model/MonitoredGroupTest.php
```

## Требования

### Поля

| Поле | Тип | Описание |
| ---- | --- | -------- |
| `id` | GroupId | Уникальный идентификатор |
| `externalGroupId` | string | ID группы в мессенджере |
| `groupTitle` | string | Название группы |
| `messengerType` | Messenger | Тип мессенджера |
| `sessionId` | SessionId? | Ссылка на сессию (для приватных) |
| `isPrivate` | bool | Приватная ли группа |
| `status` | GroupStatus | Статус группы |
| `lastMessageAt` | DateTimeImmutable? | Время последнего сообщения |
| `createdAt` | DateTimeImmutable | Дата создания |
| `updatedAt` | DateTimeImmutable | Дата обновления |

### Методы

- `static createPublic(externalGroupId, groupTitle, messengerType)` — создание публичной группы
- `static createPrivate(externalGroupId, groupTitle, messengerType, sessionId)` — создание приватной группы
- `static restore(...)` — восстановление из persistence
- Геттеры для всех полей
- `isActive()`, `isPending()`, `isFailed()` — проверки состояния
- `canReceiveMessages()` — может ли получать сообщения
- `requiresSession()`, `hasSession()` — проверки сессии
- `activate()` — активация (PENDING → ACTIVE)
- `markAsFailed(reason)` — ошибка (PENDING → FAILED)
- `deactivateBySessionExpiry()` — деактивация при истечении сессии (ACTIVE → FAILED)
- `updateGroupTitle(newTitle)` — обновление названия
- `recordMessageReceived()` — запись времени получения сообщения

### Инварианты

- Публичная группа: `isPrivate=false`, `sessionId=null`
- Приватная группа: `isPrivate=true`, `sessionId` required

## Тесты

- [x] Создание публичной группы (sessionId=null, isPrivate=false)
- [x] Создание приватной группы (sessionId required, isPrivate=true)
- [x] activate() переводит из PENDING в ACTIVE
- [x] markAsFailed() переводит из PENDING в FAILED
- [x] deactivateBySessionExpiry() работает только для приватных ACTIVE групп
- [x] Исключение при недопустимом переходе состояния
- [x] updateGroupTitle() обновляет название
- [x] recordMessageReceived() обновляет lastMessageAt
- [x] externalGroupId НЕ передаётся в события (security)
- [x] Все переходы генерируют соответствующие события
- [x] restore() не генерирует события

## Зависимости

- GroupId (задача 0077)
- SessionId (задача 0077)
- GroupStatus (задача 0079)
- `Messenger` (Shared Context) — задача 0006
- Domain Events (задача 0082)
- Exceptions (задача 0081)

## Definition of Done

- [x] Класс MonitoredGroup реализован
- [x] Публичные и приватные группы работают корректно
- [x] externalGroupId защищён от утечки в события
- [x] Unit-тесты написаны и проходят
- [x] Код соответствует PSR-12
- [x] PHPStan level 9 без ошибок
