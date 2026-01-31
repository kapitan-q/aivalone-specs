# Задача 0021: Реализация UserRepositoryInterface

## Описание

Реализовать интерфейс `UserRepositoryInterface` в Domain Layer для определения контракта по управлению хранилищем пользователей.

## Требования

- [x] Интерфейс должен быть расположен в `src/Context/Account/Domain/Repository/UserRepositoryInterface.php`
- [x] Быть частью Domain Layer (независимо от реализации)
- [x] Определить все методы для работы с хранилищем

## Интерфейс

```php
interface UserRepositoryInterface
{
    public function findById(UserId $id): ?User;

    public function findByMessenger(Messenger $messenger, string $messengerId): ?User;

    public function save(User $user): void;
}
```

## Методы

### findById(UserId $id): ?User

Получить пользователя по идентификатору.

**Параметры**:
- `$id` - идентификатор пользователя

**Возвращает**:
- `User` если найден
- `null` если не найден

**Исключения**: Нет (возвращает null)

### findByMessenger(Messenger $messenger, string $messengerId): ?User

Получить пользователя по мессенджеру и его идентификатору в этом мессенджере.

**Параметры**:
- `$messenger` - код мессенджера
- `$messengerId` - идентификатор пользователя в этом мессенджере

**Возвращает**:
- `User` если найден
- `null` если не найден

**Использование**: Проверка уникальности мессенджера перед добавлением или регистрацией

### save(User $user): void

Сохранить пользователя (create или update).

**Параметры**:
- `$user` - объект пользователя для сохранения

**Логика**:
- Преобразует Domain Model User в ORM Entity UserEntity (Data Mapper)
- Сохраняет в БД
- Может быть как insert, так и update (БД определит по наличию id)

**Исключения**:
- Может выбросить инфраструктурное исключение при ошибке БД

## Критерии готовности

- [x] Интерфейс реализован со всеми методами
- [x] Документация актуальна

## Зависимости

- Account Context: User
- Account Context: UserId
- Shared Context: Messenger

## Примечания

- Интерфейс не должен содержать implementation деталей (такие как EntityManager)
- Реализация будет в Infrastructure Layer (UserRepository)
- Методы фиксированы и не должны изменяться после публикации

## Статус

done
