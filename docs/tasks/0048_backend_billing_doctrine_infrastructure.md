# Задача 0048: Doctrine Infrastructure (Repositories, Entities, DataMappers)

## Описание

Создать инфраструктурный слой для Billing Context:
- Интерфейсы репозиториев
- Doctrine Entities
- Doctrine Repositories
- DataMappers (Domain ↔ Entity)
- Doctrine Migrations

## Фаза

**Phase 5: Infrastructure**

## Спецификации

- 📄 [tariff-repository.md](../backend/billing/infrastructure/tariff-repository.md)
- 📄 [user-subscription-repository.md](../backend/billing/infrastructure/user-subscription-repository.md)

## Зависимости

- ⏳ Domain Models — задачи 0034-0040
- ⏳ Exceptions — задача 0041
- ✅ Doctrine ORM

## Расположение файлов

```
src/Context/Billing/
├── Domain/
│   └── Repository/
│       ├── TariffRepositoryInterface.php
│       └── UserSubscriptionRepositoryInterface.php
└── Infrastructure/
    └── Persistence/
        ├── Entity/
        │   ├── TariffEntity.php
        │   └── UserSubscriptionEntity.php
        ├── Repository/
        │   ├── DoctrineTariffRepository.php
        │   └── DoctrineUserSubscriptionRepository.php
        └── DataMapper/
            ├── TariffDataMapper.php
            └── UserSubscriptionDataMapper.php
```

---

## 1. TariffRepositoryInterface

```php
interface TariffRepositoryInterface
{
    public function findById(TariffId $id): ?Tariff;
    public function findByCode(TariffEnum $code): ?Tariff;
    public function findAll(): array;
    public function save(Tariff $tariff): void;
    public function remove(Tariff $tariff): void;
}
```

---

## 2. UserSubscriptionRepositoryInterface

```php
interface UserSubscriptionRepositoryInterface
{
    public function findById(UserSubscriptionId $id): ?UserSubscription;
    public function findActiveByUserId(UserId $userId): array;
    public function findAllByUserId(UserId $userId): array;
    public function findActiveByUserAndTariff(UserId $userId, TariffId $tariffId): ?UserSubscription;
    public function findExpiredSubscriptions(\DateTimeImmutable $now): array;
    public function findExpiringSoon(\DateTimeImmutable $now, int $days): array;
    public function save(UserSubscription $subscription): void;
    public function remove(UserSubscription $subscription): void;
}
```

---

## 3. Doctrine Entities

### TariffEntity

```php
#[ORM\Entity]
#[ORM\Table(name: 'billing_tariffs')]
class TariffEntity
{
    #[ORM\Id]
    #[ORM\Column(type: 'string', length: 36)]
    private string $id;

    #[ORM\Column(type: 'string', length: 50, unique: true)]
    private string $code;

    #[ORM\Column(type: 'string', length: 255)]
    private string $name;

    #[ORM\Column(type: 'integer')]
    private int $priority;

    #[ORM\Column(type: 'decimal', precision: 10, scale: 2)]
    private string $price;

    #[ORM\Column(type: 'json')]
    private array $options = [];

    #[ORM\Column(type: 'datetime_immutable')]
    private \DateTimeImmutable $createdAt;

    #[ORM\Column(type: 'datetime_immutable')]
    private \DateTimeImmutable $updatedAt;
}
```

### UserSubscriptionEntity

```php
#[ORM\Entity]
#[ORM\Table(name: 'billing_user_subscriptions')]
#[ORM\Index(columns: ['user_id', 'status'])]
#[ORM\Index(columns: ['valid_until', 'status'])]
class UserSubscriptionEntity
{
    #[ORM\Id]
    #[ORM\Column(type: 'string', length: 36)]
    private string $id;

    #[ORM\Column(type: 'string', length: 36)]
    private string $userId;

    #[ORM\Column(type: 'string', length: 36)]
    private string $tariffId;

    #[ORM\Column(type: 'string', length: 20)]
    private string $status;

    #[ORM\Column(type: 'string', length: 20, nullable: true)]
    private ?string $period = null;

    #[ORM\Column(type: 'datetime_immutable', nullable: true)]
    private ?\DateTimeImmutable $validUntil = null;

    #[ORM\Column(type: 'string', length: 36, nullable: true)]
    private ?string $previousSubscriptionId = null;

    #[ORM\Column(type: 'datetime_immutable')]
    private \DateTimeImmutable $createdAt;

    #[ORM\Column(type: 'datetime_immutable')]
    private \DateTimeImmutable $updatedAt;
}
```

---

## 4. DataMappers

### TariffDataMapper

```php
class TariffDataMapper
{
    public function toDomain(TariffEntity $entity): Tariff;
    public function toEntity(Tariff $domain): TariffEntity;
    public function updateEntity(TariffEntity $entity, Tariff $domain): void;
}
```

### UserSubscriptionDataMapper

```php
class UserSubscriptionDataMapper
{
    public function toDomain(UserSubscriptionEntity $entity): UserSubscription;
    public function toEntity(UserSubscription $domain): UserSubscriptionEntity;
    public function updateEntity(UserSubscriptionEntity $entity, UserSubscription $domain): void;
}
```

---

## 5. Doctrine Repositories

### DoctrineTariffRepository

```php
class DoctrineTariffRepository implements TariffRepositoryInterface
{
    public function __construct(
        private EntityManagerInterface $em,
        private TariffDataMapper $mapper
    ) {}

    public function findById(TariffId $id): ?Tariff
    {
        $entity = $this->em->find(TariffEntity::class, $id->toString());
        return $entity ? $this->mapper->toDomain($entity) : null;
    }

    public function findByCode(TariffEnum $code): ?Tariff
    {
        $entity = $this->em->getRepository(TariffEntity::class)
            ->findOneBy(['code' => $code->code()]);
        return $entity ? $this->mapper->toDomain($entity) : null;
    }

    // ... остальные методы
}
```

### DoctrineUserSubscriptionRepository

```php
class DoctrineUserSubscriptionRepository implements UserSubscriptionRepositoryInterface
{
    public function findExpiredSubscriptions(\DateTimeImmutable $now): array
    {
        $qb = $this->em->createQueryBuilder();
        $qb->select('e')
            ->from(UserSubscriptionEntity::class, 'e')
            ->where('e.status = :status')
            ->andWhere('e.validUntil IS NOT NULL')
            ->andWhere('e.validUntil < :now')
            ->setParameter('status', SubscriptionStatus::ACTIVE->code())
            ->setParameter('now', $now);

        return array_map(
            fn($e) => $this->mapper->toDomain($e),
            $qb->getQuery()->getResult()
        );
    }

    public function findExpiringSoon(\DateTimeImmutable $now, int $days): array
    {
        $targetDate = $now->modify("+{$days} days");
        $targetStart = $targetDate->setTime(0, 0, 0);
        $targetEnd = $targetDate->setTime(23, 59, 59);

        // ... query для диапазона дат
    }

    // ... остальные методы
}
```

---

## 6. Migration

```php
final class VersionXXXX_CreateBillingTables extends AbstractMigration
{
    public function up(Schema $schema): void
    {
        $this->addSql('CREATE TABLE billing_tariffs (...)');
        $this->addSql('CREATE TABLE billing_user_subscriptions (...)');
        $this->addSql('CREATE INDEX ...');
    }
}
```

---

## Критерии готовности

- [x] Созданы интерфейсы репозиториев в Domain слое
- [x] Созданы Doctrine Entities
- [x] Созданы DataMappers
- [x] Созданы Doctrine Repositories
- [x] Создана миграция для таблиц
- [x] Настроены индексы для оптимизации запросов
- [ ] Написаны интеграционные тесты
- [x] Код соответствует PSR-12

## Статус: ✅ ВЫПОЛНЕНО

## Связанные задачи

- 0034-0040: Domain Models
- 0046: SubscriptionService
- 0047: CLI Command
