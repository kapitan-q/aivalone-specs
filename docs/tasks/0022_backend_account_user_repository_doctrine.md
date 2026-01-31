# Задача 0022: Реализация Doctrine UserRepository

## Описание

Реализовать конкретную реализацию `UserRepository` на основе Doctrine ORM в Infrastructure Layer.
Преобразует между Domain Model `User` и ORM Entity `UserEntity` используя Data Mapper pattern.

## Требования

- [x] Класс должен быть расположен в `src/Context/Account/Infrastructure/Persistence/Doctrine/Repository/UserRepository.php`
- [x] Реализовать интерфейс `UserRepositoryInterface`
- [x] Использовать Doctrine EntityManager для работы с БД
- [x] Преобразовывать UserEntity ↔ User используя Mapper

## Интерфейс

```php
class UserRepository implements UserRepositoryInterface
{
    public function __construct(
        private EntityManagerInterface $entityManager,
        private UserDataMapper $mapper
    ) {}

    public function findById(UserId $id): ?User;

    public function findByMessenger(Messenger $messenger, string $messengerId): ?User;

    public function save(User $user): void;
}
```

## Требования

### findById(UserId $id): ?User

- [x] Использовать `EntityManager::find()` или QueryBuilder
- [x] Выполнить поиск в таблице `users` по полю `id`
- [x] Если найдена UserEntity, преобразовать в User через Mapper
- [x] Вернуть null если не найдена

### findByMessenger(Messenger $messenger, string $messengerId): ?User

- [x] Выполнить поиск в таблице `user_messengers` (если отдельная таблица)
- [x] Или поиск в JSON поле `messengers` (если JSON хранится)
- [x] Найти UserEntity по мессенджеру и messengerId
- [x] Преобразовать в User через Mapper
- [x] Вернуть null если не найдена

### save(User $user): void

- [x] Преобразовать User в UserEntity через Mapper
- [x] Использовать `EntityManager::persist()` и `EntityManager::flush()`
- [x] Handle как create, так и update (flush сделает нужную операцию)
- [x] Может выбросить инфраструктурное исключение при ошибке

## Data Mapper

```php
class UserDataMapper
{
    public function mapToDomainModel(UserEntity $entity): User;

    public function mapToEntity(User $user): UserEntity;
}
```

Будет реализован в отдельной задаче.

## Критерии готовности

- [x] Класс реализован со всеми методами
- [x] Покрыт интеграционными тестами с реальной БД (минимум 3 теста)
- [x] Документация актуальна

## Зависимости

- Doctrine ORM
- Account Context: UserRepositoryInterface
- Account Context: User
- Account Context: UserId
- Account Context: UserEntity
- Account Context: UserDataMapper

## Примечания

- Транзакция управляется на уровне Application Service, не в Repository
- Repository только сохраняет изменения, не создает транзакции
- Все преобразования делегируются Mapper для чистоты кода

## Статус

done
