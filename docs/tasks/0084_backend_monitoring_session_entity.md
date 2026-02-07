# Задача 0084: MonitoringSession Entity

## Контекст

`MonitoringSession` — сущность внутри агрегата `MonitoringProfile`, представляющая сессию авторизации пользователя в мессенджере. Сессия управляет жизненным циклом авторизации через auth-service.

## Цель

Создать Entity `MonitoringSession` с полной логикой state machine авторизации.

## Спецификация

- [MonitoringSession Entity](../backend/monitoring/models/monitoring-session.md)
- [SessionStatus Enum](../backend/monitoring/models/session-status-enum.md)

## Файлы для создания

```
src/Context/Monitoring/Domain/Model/MonitoringSession.php
tests/Unit/Context/Monitoring/Domain/Model/MonitoringSessionTest.php
```

## Требования

### Поля

| Поле | Тип | Описание |
| ---- | --- | -------- |
| `id` | SessionId | Уникальный идентификатор |
| `messengerType` | Messenger | Тип мессенджера |
| `externalSessionId` | string? | ID сессии в auth-service |
| `status` | SessionStatus | Текущий статус |
| `isActive` | bool | Активна ли сессия |
| `displayName` | string? | Отображаемое имя |
| `lastActivityAt` | DateTimeImmutable? | Время последней активности |
| `createdAt` | DateTimeImmutable | Дата создания |
| `updatedAt` | DateTimeImmutable | Дата обновления |

### Методы

- `static create(messengerType)` — создание новой сессии (status=NEW)
- `static restore(...)` — восстановление из persistence
- Геттеры для всех полей
- `isAuthorized()` — проверка авторизации
- `isPending()` — проверка ожидания
- `canStartAuth()` — можно ли начать авторизацию
- `startAuthorizing()` — начало авторизации (NEW → AUTHORIZING)
- `markAsAuthorized(externalSessionId, displayName)` — успешная авторизация (AUTHORIZING → AUTHORIZED)
- `markAsFailed(reason)` — ошибка авторизации (AUTHORIZING → FAILED)
- `markAsExpired()` — истечение сессии (AUTHORIZED → EXPIRED)
- `markAsRevoked()` — отзыв сессии (AUTHORIZED → EXPIRED)
- `deactivate()` — деактивация пользователем
- `updateActivity()` — обновление lastActivityAt

### State Machine

```
NEW → AUTHORIZING → AUTHORIZED → EXPIRED
                 → FAILED
                 → EXPIRED
FAILED → AUTHORIZING (retry)
EXPIRED → AUTHORIZING (retry)
```

## Тесты

- [x] Создание сессии со статусом NEW
- [x] startAuthorizing() переводит в AUTHORIZING
- [x] markAsAuthorized() устанавливает AUTHORIZED и externalSessionId
- [x] markAsFailed() устанавливает FAILED и сбрасывает externalSessionId
- [x] markAsExpired() устанавливает EXPIRED и деактивирует сессию
- [x] markAsRevoked() аналогичен markAsExpired()
- [x] Исключение при недопустимом переходе (напр. NEW → AUTHORIZED)
- [x] isAuthorized() возвращает true только для AUTHORIZED+active
- [x] canStartAuth() возвращает true для NEW, FAILED, EXPIRED
- [x] deactivate() деактивирует сессию
- [x] Все переходы генерируют соответствующие события
- [x] externalSessionId НЕ передаётся в события (security)
- [x] restore() не генерирует события

## Зависимости

- SessionId (задача 0077)
- SessionStatus (задача 0078)
- `Messenger` (Shared Context) — задача 0006
- Domain Events (задача 0082)
- Exceptions (задача 0081)

## Definition of Done

- [x] Класс MonitoringSession реализован
- [x] State machine корректно работает
- [x] externalSessionId защищён от утечки в события
- [x] Unit-тесты написаны и проходят
- [x] Код соответствует PSR-12
- [x] PHPStan level 9 без ошибок
