# Задача 0025: Реализация UserService

## Описание

Реализовать сервис `UserService` в Application Layer для управления бизнес-логикой работы с пользователями.

## Требования

- [x] Класс должен быть расположен в `src/Context/Account/Application/Service/UserService.php`
- [x] Быть частью Application Layer
- [x] Реализовать 4 основных метода согласно спецификации
- [x] Использовать UserRepository для получения/сохранения
- [x] Использовать EventBus для публикации событий

## Основные методы

### registerUser(Messenger $messenger, string $messengerId): User

Регистрация нового пользователя.

**Ответственность**:
- Валидация входных параметров (не пусто, etc)
- **Проверка уникальности** через `repository->findByMessenger($messenger, $messengerId)`
- Выброс `DuplicateUserException` если существует
- Вызов `User::registerUser($messenger, $messengerId)`
- **Сохранение** через `repository->save($user)`
- **Публикация событий** через `eventBus->publish(...$events)`

**Возвращает**: User объект (сохранённый)

### addMessenger(UserId $userId, Messenger $messenger, string $messengerId): User

Добавление мессенджера.

**Ответственность**:
- Валидация входных параметров
- **Проверка уникальности** через `repository->findByMessenger($messenger, $messengerId)`
- Выброс `DuplicateMessengerException` если существует
- Получение User из репозитория: `repository->findById($userId)`
- Выброс `UserNotFoundException` если не найден
- Вызов `user->addMessenger($messenger, $messengerId)` на полученном объекте
- **Сохранение** через `repository->save($user)`
- **Публикация событий** через `eventBus->publish(...$events)`

**Возвращает**: User объект (сохранённый)

### removeMessenger(UserId $userId, Messenger $messenger, string $messengerId): User

Удаление мессенджера.

**Ответственность**:
- Валидация входных параметров
- Получение User из репозитория: `repository->findById($userId)`
- Выброс `UserNotFoundException` если не найден
- Вызов `user->removeMessenger($messenger, $messengerId)`
- Может выбросить `MessengerNotFoundException` или `AtLeastOneMessengerRequiredException` из User Model
- **Сохранение** через `repository->save($user)`
- **Публикация событий** через `eventBus->publish(...$events)`

**Возвращает**: User объект (сохранённый)

### updateTariffs(UserId $userId, array<Tariff> $tariffs): User

Обновление тарифов.

**Ответственность**:
- Валидация входных параметров (не пусто)
- Получение User из репозитория: `repository->findById($userId)`
- Выброс `UserNotFoundException` если не найден
- Вызов `user->replaceTariffs($tariffs)` с массивом объектов Tariff
- **Сохранение** через `repository->save($user)`
- **Публикация событий** через `eventBus->publish(...$events)`

**Возвращает**: User объект (сохранённый)

**Примечание**: Валидация тарифных кодов выполняется на уровне Command Handler перед вызовом этого метода. Метод ожидает уже сконвертированные объекты Tariff.

## Конструктор

```php
public function __construct(
    private UserRepositoryInterface $repository,
    private EventBusInterface $eventBus
)
```

## Важные особенности

- [x] **Все методы сохраняют User в репозиторий**
- [x] **Все методы публикуют события через EventBus**
- [x] Все проверки уникальности выполняются перед изменением модели
- [x] Валидация тарифов выполняется с использованием Tariff enum

## Критерии готовности

- [x] Класс реализован со всеми методами
- [x] Сохранение выполняется в UserService
- [x] Публикация событий выполняется в UserService
- [ ] Покрыт unit-тестами с мок-репозиторием (минимум 8 тестов)
- [ ] Документация актуальна

## Зависимости

- Account Context: UserRepositoryInterface
- Account Context: User
- Account Context: UserId
- Account Context: Исключения (UserNotFoundException, DuplicateUserException, DuplicateMessengerException)
- Shared Context: EventBusInterface
- Shared Context: Tariff
- Shared Context: Messenger

## Примечания

- Сервис выполняет полный цикл операции: проверка → изменение → сохранение → публикация событий
- Все бизнес-логика в User Model, UserService координирует и оркеструет процесс

## Статус

done
