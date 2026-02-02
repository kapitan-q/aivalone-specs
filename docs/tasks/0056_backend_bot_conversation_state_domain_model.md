# Задача 0056: ConversationState Domain Model

## Контекст

ConversationState хранит состояние активного диалога пользователя. Позволяет продолжить диалог после прерывания и отслеживать текущий шаг FSM.

## Цель

Создать Entity `ConversationState` для управления состоянием диалогов.

## Спецификация

- [ConversationState](../backend/bot/models/conversation-state.md)

## Файлы для создания

```
src/Context/Bot/Domain/Model/ConversationState.php
tests/Unit/Context/Bot/Domain/Model/ConversationStateTest.php
```

## Важно

1. Составной первичный ключ: `(userId, messenger)` — гарантирует один активный диалог на пользователя и мессенджер
2. Используется `UserId` Value Object из Shared Context (не общий `Uuid`)
3. При завершении диалога state удаляется (не меняет статус)

## Требования

### ConversationState

```php
<?php

declare(strict_types=1);

namespace App\Context\Bot\Domain\Model;

use App\Context\Shared\Domain\Model\Messenger;
use App\Context\Shared\Domain\Model\UserId;

/**
 * Состояние активного диалога пользователя.
 *
 * Entity (не AggregateRoot, жизненный цикл привязан к диалогу).
 * Первичный ключ: (userId, messenger).
 */
class ConversationState
{
    private UserId $userId;
    private Messenger $messenger;
    private string $conversationCode;
    private string $stepCode;
    /** @var array<string, mixed> */
    private array $contextData;
    private \DateTimeImmutable $updatedAt;

    /**
     * @param array<string, mixed> $contextData
     */
    private function __construct(
        UserId $userId,
        Messenger $messenger,
        string $conversationCode,
        string $stepCode,
        array $contextData,
    ) {
        $this->userId = $userId;
        $this->messenger = $messenger;
        $this->conversationCode = $conversationCode;
        $this->stepCode = $stepCode;
        $this->contextData = $contextData;
        $this->updatedAt = new \DateTimeImmutable();
    }

    /**
     * Создание нового состояния диалога.
     *
     * @param array<string, mixed> $contextData
     */
    public static function create(
        UserId $userId,
        Messenger $messenger,
        string $conversationCode,
        string $stepCode,
        array $contextData = [],
    ): self {
        return new self(
            userId: $userId,
            messenger: $messenger,
            conversationCode: $conversationCode,
            stepCode: $stepCode,
            contextData: $contextData,
        );
    }

    /**
     * Восстановление из хранилища.
     *
     * @param array<string, mixed> $contextData
     */
    public static function restore(
        UserId $userId,
        Messenger $messenger,
        string $conversationCode,
        string $stepCode,
        array $contextData,
        \DateTimeImmutable $updatedAt,
    ): self {
        $state = new self(
            userId: $userId,
            messenger: $messenger,
            conversationCode: $conversationCode,
            stepCode: $stepCode,
            contextData: $contextData,
        );
        $state->updatedAt = $updatedAt;

        return $state;
    }

    /**
     * Обновление состояния диалога.
     *
     * @param array<string, mixed> $contextData
     */
    public function update(
        string $conversationCode,
        string $stepCode,
        array $contextData,
    ): void {
        $this->conversationCode = $conversationCode;
        $this->stepCode = $stepCode;
        $this->contextData = $contextData;
        $this->updatedAt = new \DateTimeImmutable();
    }

    /**
     * Обновление только текущего шага.
     */
    public function updateStep(string $stepCode): void
    {
        $this->stepCode = $stepCode;
        $this->updatedAt = new \DateTimeImmutable();
    }

    /**
     * Обновление только данных контекста.
     *
     * @param array<string, mixed> $contextData
     */
    public function updateContextData(array $contextData): void
    {
        $this->contextData = $contextData;
        $this->updatedAt = new \DateTimeImmutable();
    }

    public function getUserId(): UserId
    {
        return $this->userId;
    }

    public function getMessenger(): Messenger
    {
        return $this->messenger;
    }

    public function getConversationCode(): string
    {
        return $this->conversationCode;
    }

    public function getStepCode(): string
    {
        return $this->stepCode;
    }

    /**
     * @return array<string, mixed>
     */
    public function getContextData(): array
    {
        return $this->contextData;
    }

    public function getUpdatedAt(): \DateTimeImmutable
    {
        return $this->updatedAt;
    }

    /**
     * Проверяет, является ли это тот же диалог.
     */
    public function isFor(string $conversationCode): bool
    {
        return $this->conversationCode === $conversationCode;
    }
}
```

## Инварианты

1. Один активный диалог на пользователя и мессенджер (составной ключ)
2. При завершении диалога state удаляется
3. updatedAt обновляется при любом изменении
4. contextData — произвольные данные диалога (JSON в БД)

## Тесты

- [x] Создание нового состояния через create()
- [x] Восстановление через restore()
- [x] update() обновляет все поля
- [x] updateStep() меняет только stepCode
- [x] updateContextData() меняет только contextData
- [x] isFor() корректно определяет диалог
- [x] updatedAt обновляется при изменениях
- [x] UserId используется корректно

## Зависимости

- `App\Context\Shared\Domain\Model\UserId` (задача 0005)
- `App\Context\Shared\Domain\Model\Messenger` (задача 0006)

## Definition of Done

- [x] Класс ConversationState создан
- [x] Используется UserId Value Object
- [x] Все методы реализованы
- [x] Инварианты соблюдены
- [x] Unit-тесты написаны и проходят
- [x] Код соответствует PSR-12
- [x] PHPStan level 9 без ошибок
