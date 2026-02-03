# ConversationState Repository Specification

## Интерфейс

```php
interface ConversationStateRepositoryInterface
{
    /**
     * Поиск состояния по составному ключу (userId + messenger)
     */
    public function find(UserId $userId, Messenger $messenger): ?ConversationState;

    /**
     * Сохранение состояния (insert или update)
     */
    public function save(ConversationState $state): void;

    /**
     * Удаление состояния
     */
    public function delete(UserId $userId, Messenger $messenger): void;

    /**
     * Удаление всех состояний пользователя (например, при удалении аккаунта)
     */
    public function deleteByUserId(UserId $userId): void;
}
```

## Doctrine Entity

```php
#[ORM\Entity]
#[ORM\Table(name: 'conversation_states')]
class ConversationStateEntity
{
    #[ORM\Id]
    #[ORM\Column(name: 'user_id', type: 'string', length: 36)]
    private string $userId;

    #[ORM\Id]
    #[ORM\Column(type: 'string', length: 32)]
    private string $messenger;

    #[ORM\Column(name: 'conversation_code', type: 'string', length: 64)]
    private string $conversationCode;

    #[ORM\Column(name: 'step_code', type: 'string', length: 64)]
    private string $stepCode;

    #[ORM\Column(name: 'context_data', type: 'json')]
    private array $contextData;

    #[ORM\Column(name: 'updated_at', type: 'datetime_immutable')]
    private DateTimeImmutable $updatedAt;
}
```

## Таблица БД

```sql
CREATE TABLE bot_conversation_states (
    user_id VARCHAR(36) NOT NULL,
    messenger VARCHAR(32) NOT NULL,
    conversation_code VARCHAR(64) NOT NULL,
    step_code VARCHAR(64) NOT NULL,
    context_data JSON NOT NULL DEFAULT '{}',
    updated_at TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP,

    PRIMARY KEY (user_id, messenger),
    INDEX idx_updated_at (updated_at)
);
```

## Реализация репозитория

```php
final class DoctrineConversationStateRepository implements ConversationStateRepositoryInterface
{
    public function __construct(
        private EntityManagerInterface $em,
        private ConversationStateDataMapper $mapper,
    ) {}

    public function find(UserId $userId, Messenger $messenger): ?ConversationState
    {
        $entity = $this->em->find(ConversationStateEntity::class, [
            'userId' => $userId->getValue(),
            'messenger' => $messenger->value,
        ]);

        return $entity ? $this->mapper->toDomain($entity) : null;
    }

    public function save(ConversationState $state): void
    {
        $entity = $this->mapper->toEntity($state);
        $this->em->persist($entity);
        $this->em->flush();
    }

    public function delete(UserId $userId, Messenger $messenger): void
    {
        $this->em->createQueryBuilder()
            ->delete(ConversationStateEntity::class, 's')
            ->where('s.userId = :userId')
            ->andWhere('s.messenger = :messenger')
            ->setParameter('userId', $userId->getValue())
            ->setParameter('messenger', $messenger->value)
            ->getQuery()
            ->execute();
    }
}
```

## Очистка устаревших состояний

Периодическая очистка состояний, не обновлявшихся более 24 часов:

```php
// CLI command: bin/console bot:cleanup-states
public function execute(): int
{
    $threshold = new DateTimeImmutable('-24 hours');

    $deleted = $this->em->createQueryBuilder()
        ->delete(ConversationStateEntity::class, 's')
        ->where('s.updatedAt < :threshold')
        ->setParameter('threshold', $threshold)
        ->getQuery()
        ->execute();

    $this->logger->info('Cleaned up conversation states', ['count' => $deleted]);

    return Command::SUCCESS;
}
```

## Связанные документы

* [Infrastructure Overview](overview.md)
* [ConversationState Model](../models/conversation-state.md)

## Статус реализации

* [ ] Интерфейс репозитория создан
* [ ] Doctrine Entity создана
* [ ] Data Mapper реализован
* [ ] Doctrine Repository реализован
* [ ] Миграция создана
* [ ] CLI команда очистки создана
* [ ] Unit тесты написаны
