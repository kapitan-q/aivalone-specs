# Задача 0060: ConversationStateRepository Interface

## Контекст

Репозиторий для работы с ConversationState. Управляет состоянием диалогов пользователей.

## Цель

Создать интерфейс репозитория `ConversationStateRepositoryInterface`.

## Спецификация

- [ConversationState Repository](../backend/bot/infrastructure/conversation-state-repository.md)

## Файлы для создания

```
src/Context/Bot/Domain/Repository/ConversationStateRepositoryInterface.php
```

## Требования

### ConversationStateRepositoryInterface

```php
<?php

declare(strict_types=1);

namespace App\Context\Bot\Domain\Repository;

use App\Context\Bot\Domain\Model\ConversationState;
use App\Context\Shared\Domain\Model\Messenger;
use App\Context\Shared\Domain\Model\UserId;

interface ConversationStateRepositoryInterface
{
    /**
     * Сохраняет состояние диалога (создание или обновление).
     */
    public function save(ConversationState $state): void;

    /**
     * Находит состояние диалога по составному ключу (userId, messenger).
     */
    public function find(UserId $userId, Messenger $messenger): ?ConversationState;

    /**
     * Удаляет состояние диалога (при завершении).
     */
    public function delete(UserId $userId, Messenger $messenger): void;

    /**
     * Удаляет все состояния диалогов пользователя.
     */
    public function removeAllByUser(UserId $userId): void;
}
```

## Методы

| Метод | Описание |
|-------|----------|
| `save()` | Сохраняет или обновляет состояние |
| `find()` | Находит по составному ключу (userId, messenger) |
| `delete()` | Удаляет состояние (при завершении диалога) |
| `removeAllByUser()` | Удаляет все диалоги пользователя |

## Зависимости

- `ConversationState` (задача 0056)
- `App\Context\Shared\Domain\Model\Messenger` (задача 0006)
- `App\Context\Shared\Domain\Model\UserId` (задача 0005)

## Definition of Done

- [x] Интерфейс создан
- [x] Все методы определены с корректными типами
- [x] PHPDoc комментарии добавлены
- [x] Код соответствует PSR-12
- [x] PHPStan level 9 без ошибок
