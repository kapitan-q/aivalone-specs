# Задача 0069: ConversationState Doctrine Repository

## Контекст

Doctrine реализация репозитория для ConversationState. Хранит состояние активных диалогов пользователей.

## Цель

Создать Doctrine Entity и Repository для ConversationState.

## Спецификация

- [ConversationState Repository](../backend/bot/infrastructure/conversation-state-repository.md)

## Файлы для создания

```
src/Context/Bot/Infrastructure/Persistence/Entity/ConversationStateEntity.php
src/Context/Bot/Infrastructure/Persistence/Repository/DoctrineConversationStateRepository.php
src/Context/Bot/Infrastructure/Persistence/DataMapper/ConversationStateDataMapper.php
tests/Integration/Context/Bot/Infrastructure/Persistence/DoctrineConversationStateRepositoryTest.php
```

## Важно

1. **Составной первичный ключ** — (user_id, messenger), без отдельного id
2. **Используем restore метод** — не рефлексию
3. **Без flush в repository** — flush управляется на уровне UnitOfWork/Transaction
4. **UserId Value Object** — не общий Uuid

## Требования

### ConversationStateEntity

```php
<?php

declare(strict_types=1);

namespace App\Context\Bot\Infrastructure\Persistence\Entity;

use Doctrine\ORM\Mapping as ORM;

#[ORM\Entity]
#[ORM\Table(name: 'bot_conversation_states')]
class ConversationStateEntity
{
    #[ORM\Id]
    #[ORM\Column(name: 'user_id', type: 'string', length: 36)]
    public string $userId;

    #[ORM\Id]
    #[ORM\Column(type: 'string', length: 32)]
    public string $messenger;

    #[ORM\Column(name: 'conversation_code', type: 'string', length: 64)]
    public string $conversationCode;

    #[ORM\Column(name: 'step_code', type: 'string', length: 64)]
    public string $stepCode;

    #[ORM\Column(name: 'context_data', type: 'json')]
    public array $contextData = [];

    #[ORM\Column(name: 'updated_at', type: 'datetime_immutable')]
    public \DateTimeImmutable $updatedAt;
}
```

### ConversationStateDataMapper

```php
<?php

declare(strict_types=1);

namespace App\Context\Bot\Infrastructure\Persistence\DataMapper;

use App\Context\Bot\Domain\Model\ConversationState;
use App\Context\Bot\Infrastructure\Persistence\Entity\ConversationStateEntity;
use App\Context\Shared\Domain\Enum\Messenger;
use App\Context\Shared\Domain\ValueObject\UserId;

final class ConversationStateDataMapper
{
    public function toEntity(ConversationState $domain): ConversationStateEntity
    {
        $entity = new ConversationStateEntity();
        $entity->userId = $domain->getUserId()->toString();
        $entity->messenger = $domain->getMessenger()->value;
        $entity->conversationCode = $domain->getConversationCode();
        $entity->stepCode = $domain->getStepCode();
        $entity->contextData = $domain->getContextData();
        $entity->updatedAt = $domain->getUpdatedAt();

        return $entity;
    }

    public function toDomain(ConversationStateEntity $entity): ConversationState
    {
        // Используем статический метод restore для восстановления
        return ConversationState::restore(
            UserId::fromString($entity->userId),
            Messenger::from($entity->messenger),
            $entity->conversationCode,
            $entity->stepCode,
            $entity->contextData,
            $entity->updatedAt,
        );
    }
}
```

### DoctrineConversationStateRepository

```php
<?php

declare(strict_types=1);

namespace App\Context\Bot\Infrastructure\Persistence\Repository;

use App\Context\Bot\Domain\Model\ConversationState;
use App\Context\Bot\Domain\Repository\ConversationStateRepositoryInterface;
use App\Context\Bot\Infrastructure\Persistence\DataMapper\ConversationStateDataMapper;
use App\Context\Bot\Infrastructure\Persistence\Entity\ConversationStateEntity;
use App\Context\Shared\Domain\Enum\Messenger;
use App\Context\Shared\Domain\ValueObject\UserId;
use Doctrine\ORM\EntityManagerInterface;

final class DoctrineConversationStateRepository implements ConversationStateRepositoryInterface
{
    public function __construct(
        private readonly EntityManagerInterface $em,
        private readonly ConversationStateDataMapper $mapper,
    ) {}

    public function find(UserId $userId, Messenger $messenger): ?ConversationState
    {
        $entity = $this->em->find(ConversationStateEntity::class, [
            'userId' => $userId->toString(),
            'messenger' => $messenger->value,
        ]);

        return $entity !== null ? $this->mapper->toDomain($entity) : null;
    }

    public function save(ConversationState $state): void
    {
        $entity = $this->mapper->toEntity($state);
        $this->em->persist($entity);
        // Без flush — управляется на уровне транзакции
    }

    public function delete(UserId $userId, Messenger $messenger): void
    {
        $entity = $this->em->find(ConversationStateEntity::class, [
            'userId' => $userId->toString(),
            'messenger' => $messenger->value,
        ]);

        if ($entity !== null) {
            $this->em->remove($entity);
            // Без flush — управляется на уровне транзакции
        }
    }

    public function removeAllByUser(UserId $userId): void
    {
        $this->em->createQueryBuilder()
            ->delete(ConversationStateEntity::class, 's')
            ->where('s.userId = :userId')
            ->setParameter('userId', $userId->toString())
            ->getQuery()
            ->execute();
    }
}
```

## Обновление интерфейса репозитория (задача 0060)

```php
<?php

declare(strict_types=1);

namespace App\Context\Bot\Domain\Repository;

use App\Context\Bot\Domain\Model\ConversationState;
use App\Context\Shared\Domain\Enum\Messenger;
use App\Context\Shared\Domain\ValueObject\UserId;

interface ConversationStateRepositoryInterface
{
    public function find(UserId $userId, Messenger $messenger): ?ConversationState;

    public function save(ConversationState $state): void;

    public function delete(UserId $userId, Messenger $messenger): void;

    public function removeAllByUser(UserId $userId): void;
}
```

## Тесты

- [ ] save() создаёт новую запись (после flush)
- [ ] save() обновляет существующую запись (upsert)
- [ ] find() находит состояние по userId + messenger
- [ ] find() возвращает null если не найден
- [ ] delete() удаляет состояние
- [ ] removeAllByUser() удаляет все состояния пользователя
- [ ] DataMapper корректно преобразует через restore()
- [ ] contextData корректно сохраняется и восстанавливается (JSON)

## Зависимости

- `ConversationStateRepositoryInterface` (задача 0060)
- `ConversationState` (задача 0056)
- `UserId` (Shared)

## Definition of Done

- [x] Entity класс создан с составным ключом
- [x] DataMapper создан (с использованием restore)
- [x] Repository реализован (без flush)
- [ ] Integration-тесты написаны и проходят
- [x] Код соответствует PSR-12
- [x] PHPStan level 9 без ошибок
