# NotificationEndpoint Repository Specification

## Интерфейс

```php
interface NotificationEndpointRepositoryInterface
{
    /**
     * Поиск по ID
     */
    public function findById(EndpointId $id): ?NotificationEndpoint;

    /**
     * Поиск всех endpoints пользователя
     * @return array<NotificationEndpoint>
     */
    public function findByUserId(UserId $userId): array;

    /**
     * Поиск endpoint пользователя для конкретного мессенджера
     */
    public function findByUserIdAndMessenger(
        UserId $userId,
        Messenger $messenger
    ): ?NotificationEndpoint;

    /**
     * Поиск по внешнему идентификатору
     * (используется при обработке webhook)
     */
    public function findByExternalTarget(
        Messenger $messenger,
        string $externalTargetId
    ): ?NotificationEndpoint;

    /**
     * Сохранение endpoint
     */
    public function save(NotificationEndpoint $endpoint): void;

    /**
     * Удаление endpoint
     */
    public function delete(NotificationEndpoint $endpoint): void;
}
```

## Doctrine Entity

```php
#[ORM\Entity]
#[ORM\Table(name: 'notification_endpoints')]
#[ORM\UniqueConstraint(
    name: 'unique_user_messenger',
    columns: ['user_id', 'messenger']
)]
#[ORM\Index(
    name: 'idx_external_target',
    columns: ['messenger', 'external_target_id']
)]
class NotificationEndpointEntity
{
    #[ORM\Id]
    #[ORM\Column(type: 'string', length: 36)]
    private string $id;

    #[ORM\Column(name: 'user_id', type: 'string', length: 36)]
    private string $userId;

    #[ORM\Column(type: 'string', length: 32)]
    private string $messenger;

    #[ORM\Column(name: 'external_target_id', type: 'string', length: 255)]
    private string $externalTargetId;

    #[ORM\Column(type: 'string', length: 16)]
    private string $status;

    #[ORM\Column(name: 'created_at', type: 'datetime_immutable')]
    private DateTimeImmutable $createdAt;

    #[ORM\Column(name: 'revoked_at', type: 'datetime_immutable', nullable: true)]
    private ?DateTimeImmutable $revokedAt;
}
```

## Таблица БД

```sql
CREATE TABLE bot_notification_endpoints (
    id VARCHAR(36) PRIMARY KEY,
    user_id VARCHAR(36) NOT NULL,
    messenger VARCHAR(32) NOT NULL,
    external_target_id VARCHAR(255) NOT NULL,
    status VARCHAR(16) NOT NULL DEFAULT 'active',
    created_at TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP,
    revoked_at TIMESTAMP NULL,

    CONSTRAINT unique_user_messenger UNIQUE (user_id, messenger),
    INDEX idx_external_target (messenger, external_target_id),
    INDEX idx_user_id (user_id)
);
```

## Data Mapper

```php
final class NotificationEndpointDataMapper
{
    public function toDomain(NotificationEndpointEntity $entity): NotificationEndpoint
    {
        return NotificationEndpoint::restore(
            id: EndpointId::fromString($entity->getId()),
            userId: UserId::fromString($entity->getUserId()),
            messenger: Messenger::from($entity->getMessenger()),
            externalTargetId: $entity->getExternalTargetId(),
            status: EndpointStatus::from($entity->getStatus()),
            createdAt: $entity->getCreatedAt(),
            revokedAt: $entity->getRevokedAt(),
        );
    }

    public function toEntity(NotificationEndpoint $domain): NotificationEndpointEntity
    {
        // ... mapping logic
    }
}
```

## Безопасность хранения

**ВАЖНО**: `external_target_id` хранится в БД, но:
- Не логируется в application logs
- Не включается в события
- Не передаётся за пределы Bot Context

## Связанные документы

* [Infrastructure Overview](overview.md)
* [NotificationEndpoint Model](../models/notification-endpoint.md)

## Статус реализации

* [ ] Интерфейс репозитория создан
* [ ] Doctrine Entity создана
* [ ] Data Mapper реализован
* [ ] Doctrine Repository реализован
* [ ] Миграция создана
* [ ] Unit тесты написаны
