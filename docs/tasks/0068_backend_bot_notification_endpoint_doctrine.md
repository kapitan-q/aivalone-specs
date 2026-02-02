# Задача 0068: NotificationEndpoint Doctrine Repository

## Контекст

Doctrine реализация репозитория для NotificationEndpoint. Включает Entity mapping и реализацию интерфейса репозитория.

## Цель

Создать Doctrine Entity и Repository для NotificationEndpoint.

## Спецификация

- [NotificationEndpoint Repository](../backend/bot/infrastructure/notification-endpoint-repository.md)

## Файлы для создания

```
src/Context/Bot/Infrastructure/Persistence/Entity/NotificationEndpointEntity.php
src/Context/Bot/Infrastructure/Persistence/Repository/DoctrineNotificationEndpointRepository.php
src/Context/Bot/Infrastructure/Persistence/DataMapper/NotificationEndpointDataMapper.php
tests/Integration/Context/Bot/Infrastructure/Persistence/DoctrineNotificationEndpointRepositoryTest.php
```

## Важно

1. **Используем restore метод** — не рефлексию для восстановления доменной модели
2. **Без flush в repository** — flush управляется на уровне UnitOfWork/Transaction
3. **UserId Value Object** — не общий Uuid

## Требования

### NotificationEndpointEntity

```php
<?php

declare(strict_types=1);

namespace App\Context\Bot\Infrastructure\Persistence\Entity;

use Doctrine\ORM\Mapping as ORM;

#[ORM\Entity]
#[ORM\Table(name: 'bot_notification_endpoints')]
#[ORM\Index(columns: ['user_id', 'messenger', 'status'], name: 'idx_user_messenger_status')]
#[ORM\Index(columns: ['messenger', 'external_target_id'], name: 'idx_messenger_external')]
#[ORM\UniqueConstraint(columns: ['user_id', 'messenger'], name: 'uniq_user_messenger')]
class NotificationEndpointEntity
{
    #[ORM\Id]
    #[ORM\Column(type: 'string', length: 36)]
    public string $id;

    #[ORM\Column(name: 'user_id', type: 'string', length: 36)]
    public string $userId;

    #[ORM\Column(type: 'string', length: 32)]
    public string $messenger;

    #[ORM\Column(name: 'external_target_id', type: 'string', length: 255)]
    public string $externalTargetId;

    #[ORM\Column(type: 'string', length: 32)]
    public string $status;

    #[ORM\Column(name: 'created_at', type: 'datetime_immutable')]
    public \DateTimeImmutable $createdAt;

    #[ORM\Column(name: 'revoked_at', type: 'datetime_immutable', nullable: true)]
    public ?\DateTimeImmutable $revokedAt = null;
}
```

### NotificationEndpointDataMapper

```php
<?php

declare(strict_types=1);

namespace App\Context\Bot\Infrastructure\Persistence\DataMapper;

use App\Context\Bot\Domain\Model\EndpointId;
use App\Context\Bot\Domain\Model\EndpointStatus;
use App\Context\Bot\Domain\Model\NotificationEndpoint;
use App\Context\Bot\Infrastructure\Persistence\Entity\NotificationEndpointEntity;
use App\Context\Shared\Domain\Enum\Messenger;
use App\Context\Shared\Domain\ValueObject\UserId;

final class NotificationEndpointDataMapper
{
    public function toEntity(NotificationEndpoint $domain): NotificationEndpointEntity
    {
        $entity = new NotificationEndpointEntity();
        $entity->id = $domain->getId()->toString();
        $entity->userId = $domain->getUserId()->toString();
        $entity->messenger = $domain->getMessenger()->value;
        $entity->externalTargetId = $domain->getExternalTargetId();
        $entity->status = $domain->getStatus()->value;
        $entity->createdAt = $domain->getCreatedAt();
        $entity->revokedAt = $domain->getRevokedAt();

        return $entity;
    }

    public function toDomain(NotificationEndpointEntity $entity): NotificationEndpoint
    {
        // Используем статический метод restore для восстановления
        return NotificationEndpoint::restore(
            EndpointId::fromString($entity->id),
            UserId::fromString($entity->userId),
            Messenger::from($entity->messenger),
            $entity->externalTargetId,
            EndpointStatus::from($entity->status),
            $entity->createdAt,
            $entity->revokedAt,
        );
    }
}
```

### Добавить restore метод в NotificationEndpoint (задача 0055)

```php
// В NotificationEndpoint добавить:
public static function restore(
    EndpointId $id,
    UserId $userId,
    Messenger $messenger,
    string $externalTargetId,
    EndpointStatus $status,
    \DateTimeImmutable $createdAt,
    ?\DateTimeImmutable $revokedAt,
): self {
    $endpoint = new self($id, $userId, $messenger, $externalTargetId);
    $endpoint->status = $status;
    $endpoint->createdAt = $createdAt;
    $endpoint->revokedAt = $revokedAt;

    return $endpoint;
}
```

### DoctrineNotificationEndpointRepository

```php
<?php

declare(strict_types=1);

namespace App\Context\Bot\Infrastructure\Persistence\Repository;

use App\Context\Bot\Domain\Exception\EndpointNotFoundException;
use App\Context\Bot\Domain\Model\EndpointId;
use App\Context\Bot\Domain\Model\EndpointStatus;
use App\Context\Bot\Domain\Model\NotificationEndpoint;
use App\Context\Bot\Domain\Repository\NotificationEndpointRepositoryInterface;
use App\Context\Bot\Infrastructure\Persistence\DataMapper\NotificationEndpointDataMapper;
use App\Context\Bot\Infrastructure\Persistence\Entity\NotificationEndpointEntity;
use App\Context\Shared\Domain\Enum\Messenger;
use App\Context\Shared\Domain\ValueObject\UserId;
use Doctrine\ORM\EntityManagerInterface;

final class DoctrineNotificationEndpointRepository implements NotificationEndpointRepositoryInterface
{
    public function __construct(
        private readonly EntityManagerInterface $em,
        private readonly NotificationEndpointDataMapper $mapper,
    ) {}

    public function save(NotificationEndpoint $endpoint): void
    {
        $entity = $this->mapper->toEntity($endpoint);
        $this->em->persist($entity);
        // Без flush — управляется на уровне транзакции
    }

    public function findById(EndpointId $id): ?NotificationEndpoint
    {
        $entity = $this->em->find(NotificationEndpointEntity::class, $id->toString());

        return $entity !== null ? $this->mapper->toDomain($entity) : null;
    }

    public function getById(EndpointId $id): NotificationEndpoint
    {
        $endpoint = $this->findById($id);

        if ($endpoint === null) {
            throw EndpointNotFoundException::byId($id);
        }

        return $endpoint;
    }

    public function findByUserIdAndMessenger(
        UserId $userId,
        Messenger $messenger,
    ): ?NotificationEndpoint {
        $entity = $this->em->createQueryBuilder()
            ->select('e')
            ->from(NotificationEndpointEntity::class, 'e')
            ->where('e.userId = :userId')
            ->andWhere('e.messenger = :messenger')
            ->setParameter('userId', $userId->toString())
            ->setParameter('messenger', $messenger->value)
            ->getQuery()
            ->getOneOrNullResult();

        return $entity !== null ? $this->mapper->toDomain($entity) : null;
    }

    public function findActiveByUserAndMessenger(
        UserId $userId,
        Messenger $messenger,
    ): ?NotificationEndpoint {
        $entity = $this->em->createQueryBuilder()
            ->select('e')
            ->from(NotificationEndpointEntity::class, 'e')
            ->where('e.userId = :userId')
            ->andWhere('e.messenger = :messenger')
            ->andWhere('e.status = :status')
            ->setParameter('userId', $userId->toString())
            ->setParameter('messenger', $messenger->value)
            ->setParameter('status', EndpointStatus::ACTIVE->value)
            ->getQuery()
            ->getOneOrNullResult();

        return $entity !== null ? $this->mapper->toDomain($entity) : null;
    }

    public function findByExternalTargetId(
        Messenger $messenger,
        string $externalTargetId,
    ): ?NotificationEndpoint {
        $entity = $this->em->createQueryBuilder()
            ->select('e')
            ->from(NotificationEndpointEntity::class, 'e')
            ->where('e.messenger = :messenger')
            ->andWhere('e.externalTargetId = :externalTargetId')
            ->setParameter('messenger', $messenger->value)
            ->setParameter('externalTargetId', $externalTargetId)
            ->getQuery()
            ->getOneOrNullResult();

        return $entity !== null ? $this->mapper->toDomain($entity) : null;
    }

    /**
     * @return NotificationEndpoint[]
     */
    public function findActiveByUserId(UserId $userId): array
    {
        $entities = $this->em->createQueryBuilder()
            ->select('e')
            ->from(NotificationEndpointEntity::class, 'e')
            ->where('e.userId = :userId')
            ->andWhere('e.status = :status')
            ->setParameter('userId', $userId->toString())
            ->setParameter('status', EndpointStatus::ACTIVE->value)
            ->getQuery()
            ->getResult();

        return array_map(
            fn (NotificationEndpointEntity $e) => $this->mapper->toDomain($e),
            $entities,
        );
    }

    public function remove(NotificationEndpoint $endpoint): void
    {
        $entity = $this->em->find(NotificationEndpointEntity::class, $endpoint->getId()->toString());

        if ($entity !== null) {
            $this->em->remove($entity);
            // Без flush — управляется на уровне транзакции
        }
    }
}
```

## Тесты

- [ ] save() создаёт новую запись (после flush)
- [ ] save() обновляет существующую запись
- [ ] findById() находит endpoint
- [ ] findById() возвращает null если не найден
- [ ] getById() выбрасывает исключение если не найден
- [ ] findByUserIdAndMessenger() находит по userId и messenger
- [ ] findActiveByUserAndMessenger() находит только активные
- [ ] findByExternalTargetId() находит по chat_id
- [ ] findActiveByUserId() возвращает все активные endpoints
- [ ] remove() удаляет endpoint (после flush)
- [ ] DataMapper корректно преобразует через restore()

## Зависимости

- `NotificationEndpointRepositoryInterface` (задача 0059)
- `NotificationEndpoint` (задача 0055)
- `EndpointId` (задача 0051)
- `EndpointStatus` (задача 0052)
- `EndpointNotFoundException` (задача 0057)
- `UserId` (Shared)

## Definition of Done

- [x] Entity класс создан с ORM маппингом
- [x] DataMapper создан (с использованием restore)
- [x] Repository реализован (без flush)
- [ ] Integration-тесты написаны и проходят
- [x] Код соответствует PSR-12
- [x] PHPStan level 9 без ошибок
