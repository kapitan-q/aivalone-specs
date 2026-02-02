# Задача 0059: NotificationEndpointRepository Interface

## Контекст

Репозиторий для работы с NotificationEndpoint. Определяет контракт для сохранения и поиска endpoint без привязки к конкретной реализации.

## Цель

Создать интерфейс репозитория `NotificationEndpointRepositoryInterface`.

## Спецификация

- [NotificationEndpoint Repository](../backend/bot/infrastructure/notification-endpoint-repository.md)

## Файлы для создания

```
src/Context/Bot/Domain/Repository/NotificationEndpointRepositoryInterface.php
```

## Требования

### NotificationEndpointRepositoryInterface

```php
<?php

declare(strict_types=1);

namespace App\Context\Bot\Domain\Repository;

use App\Context\Bot\Domain\Model\EndpointId;
use App\Context\Bot\Domain\Model\NotificationEndpoint;
use App\Context\Shared\Domain\Model\Messenger;
use App\Context\Shared\Domain\Model\UserId;

interface NotificationEndpointRepositoryInterface
{
    /**
     * Сохраняет endpoint (создание или обновление)
     */
    public function save(NotificationEndpoint $endpoint): void;

    /**
     * Находит endpoint по ID
     */
    public function findById(EndpointId $id): ?NotificationEndpoint;

    /**
     * Находит endpoint по ID или выбрасывает исключение
     *
     * @throws \App\Context\Bot\Domain\Exception\EndpointNotFoundException
     */
    public function getById(EndpointId $id): NotificationEndpoint;

    /**
     * Находит активный endpoint пользователя для мессенджера
     */
    public function findActiveByUserAndMessenger(
        UserId $userId,
        Messenger $messenger,
    ): ?NotificationEndpoint;

    /**
     * Находит endpoint по внешнему идентификатору (chat_id)
     * ВАЖНО: Используется только внутри Bot Context
     */
    public function findByExternalTargetId(
        Messenger $messenger,
        string $externalTargetId,
    ): ?NotificationEndpoint;

    /**
     * Находит все активные endpoints пользователя
     *
     * @return NotificationEndpoint[]
     */
    public function findActiveByUserId(UserId $userId): array;

    /**
     * Удаляет endpoint
     */
    public function remove(NotificationEndpoint $endpoint): void;
}
```

## Методы

| Метод | Описание |
|-------|----------|
| `save()` | Сохраняет или обновляет endpoint |
| `findById()` | Поиск по ID, возвращает null если не найден |
| `getById()` | Поиск по ID с исключением |
| `findActiveByUserAndMessenger()` | Поиск активного endpoint для пользователя и мессенджера |
| `findByExternalTargetId()` | Поиск по chat_id (только внутри контекста) |
| `findActiveByUserId()` | Все активные endpoints пользователя |
| `remove()` | Удаление endpoint |

## Зависимости

- `NotificationEndpoint` (задача 0055)
- `EndpointId` (задача 0051)
- `EndpointNotFoundException` (задача 0057)
- `App\Context\Shared\Domain\Enum\Messenger` (задача 0006)
- `App\Context\Shared\Domain\ValueObject\Uuid` (задача 0005)

## Definition of Done

- [x] Интерфейс создан
- [x] Все методы определены с корректными типами
- [x] PHPDoc комментарии добавлены
- [x] Код соответствует PSR-12
- [x] PHPStan level 9 без ошибок
