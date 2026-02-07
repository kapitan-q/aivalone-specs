# Задача 0089: Doctrine Infrastructure (Entities, Repositories, DataMapper)

## Контекст

Инфраструктурный слой Monitoring Context. Включает Doctrine Entities для ORM маппинга, реализацию репозитория и DataMapper для преобразования между доменными моделями и entity.

## Цель

Создать полный инфраструктурный слой для persistence.

## Спецификация

- [MonitoringProfileRepository](../backend/monitoring/infrastructure/monitoring-profile-repository.md)
- [Infrastructure Overview](../backend/monitoring/infrastructure/overview.md)

## Файлы для создания

```
src/Context/Monitoring/Infrastructure/Persistence/Doctrine
├── Entity/
│   ├── MonitoringProfileEntity.php
│   ├── FilterEntity.php
│   ├── MonitoringSessionEntity.php
│   ├── MonitoredGroupEntity.php
│   └── FilterGroupBindingEntity.php
├── Repository/
│   └── MonitoringProfileRepository.php
└── DataMapper/
    └── MonitoringProfileDataMapper.php

tests/Integration/Context/Monitoring/Infrastructure/Persistence/Doctrine
└── MonitoringProfileRepositoryTest.php
```

## Требования

### Doctrine Entities

#### MonitoringProfileEntity

```php
#[ORM\Entity]
#[ORM\Table(name: 'monitoring_profiles')]
class MonitoringProfileEntity
{
    #[ORM\Id]
    #[ORM\Column(type: 'string', length: 36)]
    private string $id;

    #[ORM\Column(type: 'string', length: 36, unique: true)]
    private string $userId;

    #[ORM\Column(type: 'boolean')]
    private bool $isActive;

    #[ORM\OneToMany(targetEntity: FilterEntity::class, mappedBy: 'profile', cascade: ['all'], orphanRemoval: true)]
    private Collection $filters;

    #[ORM\OneToMany(targetEntity: MonitoringSessionEntity::class, mappedBy: 'profile', cascade: ['all'], orphanRemoval: true)]
    private Collection $sessions;

    #[ORM\OneToMany(targetEntity: MonitoredGroupEntity::class, mappedBy: 'profile', cascade: ['all'], orphanRemoval: true)]
    private Collection $groups;

    #[ORM\OneToMany(targetEntity: FilterGroupBindingEntity::class, mappedBy: 'profile', cascade: ['all'], orphanRemoval: true)]
    private Collection $filterGroupBindings;

    #[ORM\Column(type: 'datetime_immutable')]
    private \DateTimeImmutable $createdAt;

    #[ORM\Column(type: 'datetime_immutable')]
    private \DateTimeImmutable $updatedAt;
}
```

#### FilterEntity

```php
#[ORM\Entity]
#[ORM\Table(name: 'monitoring_filters')]
class FilterEntity
{
    #[ORM\Id]
    #[ORM\Column(type: 'string', length: 36)]
    private string $id;

    #[ORM\ManyToOne(targetEntity: MonitoringProfileEntity::class, inversedBy: 'filters')]
    #[ORM\JoinColumn(nullable: false)]
    private MonitoringProfileEntity $profile;

    #[ORM\Column(type: 'string', length: 500)]
    private string $value;

    #[ORM\Column(type: 'string', length: 20)]
    private string $filterType;

    #[ORM\Column(type: 'string', length: 255, nullable: true)]
    private ?string $name = null;

    #[ORM\Column(type: 'boolean')]
    private bool $isActive;

    #[ORM\Column(type: 'datetime_immutable')]
    private \DateTimeImmutable $createdAt;

    #[ORM\Column(type: 'datetime_immutable')]
    private \DateTimeImmutable $updatedAt;
}
```

#### MonitoringSessionEntity, MonitoredGroupEntity, FilterGroupBindingEntity

Аналогичная структура с полями согласно спецификации.

### DataMapper

```php
class MonitoringProfileDataMapper
{
    public function toDomain(MonitoringProfileEntity $entity): MonitoringProfile;
    public function toEntity(MonitoringProfile $domain): MonitoringProfileEntity;
    public function updateEntity(MonitoringProfileEntity $entity, MonitoringProfile $domain): void;
}
```

### Doctrine Repository

```php
class DoctrineMonitoringProfileRepository implements MonitoringProfileRepositoryInterface
{
    public function __construct(
        private EntityManagerInterface $em,
        private MonitoringProfileDataMapper $mapper,
    ) {}

    // Реализация всех методов интерфейса
}
```

## Индексы

```sql
CREATE UNIQUE INDEX idx_profiles_user_id ON monitoring_profiles(user_id);
CREATE INDEX idx_filters_profile_id ON monitoring_filters(profile_id);
CREATE INDEX idx_sessions_profile_id ON monitoring_sessions(profile_id);
CREATE UNIQUE INDEX idx_sessions_unique ON monitoring_sessions(profile_id, messenger_type);
CREATE INDEX idx_groups_profile_id ON monitoring_groups(profile_id);
CREATE UNIQUE INDEX idx_groups_unique ON monitoring_groups(profile_id, external_group_id, messenger_type);
CREATE UNIQUE INDEX idx_bindings_unique ON monitoring_filter_group_bindings(filter_id, group_id);
```

## Тесты

- [ ] save() и findById() — round trip (интеграционный, требует DB)
- [ ] findByUserId() находит по userId (интеграционный, требует DB)
- [ ] existsByUserId() возвращает корректный результат (интеграционный, требует DB)
- [ ] findBySessionId() находит по sessionId (интеграционный, требует DB)
- [ ] findByGroupId() находит по groupId (интеграционный, требует DB)
- [ ] findByExternalGroupId() находит по внешнему ID группы (интеграционный, требует DB)
- [ ] delete() удаляет каскадно (интеграционный, требует DB)
- [x] DataMapper корректно маппит domain ↔ entity

## Зависимости

- MonitoringProfile AggregateRoot (задача 0087)
- Repository Interface (задача 0088)
- Все доменные модели (задачи 0083-0086)
- Doctrine ORM

## Definition of Done

- [x] Все Doctrine Entities созданы
- [x] DataMapper реализован
- [x] Doctrine Repository реализован
- [ ] Интеграционные тесты написаны и проходят (требует DB)
- [x] Код соответствует PSR-12
- [x] PHPStan level 9 без ошибок
