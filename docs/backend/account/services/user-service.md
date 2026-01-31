# UserService Specification

## Назначение

Сервис **UserService** управляет бизнес-логикой работы с пользователями в Application Layer.
Отвечает за взаимодействие между Application Layer и Domain Model, включая получение агрегата из репозитория, применение изменений доменной модели и сохранение результатов.

## Важное замечание о слоях

**Важно**: UserService работает в Application Layer, но основная бизнес-логика находится в Domain Model (`User` класс).

* **Application Layer (UserService)**:
  - Принимает команды/запросы
  - Валидирует данные, проверяет бизнес-инварианты (например, уникальность)
  - Получает агреват из репозитория
  - Вызывает методы доменной модели
  - Сохраняет результаты в репозиторий
  - Опубликовывает события

* **Domain Layer (User Model)**:
  - Содержит всю бизнес-логику (создание, добавление/удаление мессенджеров, обновление тарифов)
  - Публикует события через `recordEvent()`
  - Гарантирует инварианты

## Основные методы

### registerUser(Messenger $messenger, string $messengerId): User

Регистрация нового пользователя согласно спецификации процесса [CreateUser](../processes/create-user.md).

**Ответственность Application Layer (UserService)**:
- Валидация входных параметров
- **Проверка уникальности** `messenger` + `messengerId` через `UserRepository::findByMessenger()`
- Выброс `DuplicateUserException` если пользователь уже существует

**Ответственность Domain Layer (User)**:
- Создание пользователя: `User::registerUser($messenger, $messengerId)`
- Генерация `UserId`
- Инициализация начального тарифа `FREE`
- Запись события `UserRegistered`

**Ответственность UserService после изменения модели**:
- Сохранение пользователя: `UserRepository::save($user)`
- Публикация событий: `EventBus::publish(...$user->pullEvents())`

**Возвращает**: Созданный и сохранённый объект `User`

### addMessenger(UserId $userId, Messenger $messenger, string $messengerId): User

Добавление мессенджера пользователю согласно процессу [AddUserMessenger](../processes/add-user-messenger.md).

**Ответственность Application Layer (UserService)**:
- Валидация входных параметров
- **Проверка уникальности** `messenger` + `messengerId` через `UserRepository::findByMessenger()`
- Выброс `DuplicateMessengerException` если мессенджер уже привязан к другому пользователю
- Получение пользователя из репозитория: `UserRepository::findById($userId)`
- Выброс `UserNotFoundException` если пользователь не найден

**Ответственность Domain Layer (User)**:
- Добавление мессенджера: `user->addMessenger($messenger, $messengerId)`
- Запись события `UserMessengersUpdated`
- Обновление `updatedAt`

**Ответственность UserService после изменения модели**:
- Сохранение пользователя: `UserRepository::save($user)`
- Публикация событий: `EventBus::publish(...$user->pullEvents())`

**Возвращает**: Обновленный и сохранённый объект `User`

### removeMessenger(UserId $userId, Messenger $messenger, string $messengerId): User

Удаление мессенджера у пользователя согласно процессу [RemoveUserMessenger](../processes/remove-user-messenger.md).

**Ответственность Application Layer (UserService)**:
- Валидация входных параметров
- Получение пользователя из репозитория: `UserRepository::findById($userId)`
- Выброс `UserNotFoundException` если пользователь не найден

**Ответственность Domain Layer (User)**:
- Удаление мессенджера: `user->removeMessenger($messenger, $messengerId)`
- Может выбросить `MessengerNotFoundException` если мессенджер не найден
- Может выбросить `AtLeastOneMessengerRequiredException` если это последний мессенджер
- Запись события `UserMessengersUpdated`
- Обновление `updatedAt`

**Ответственность UserService после изменения модели**:
- Сохранение пользователя: `UserRepository::save($user)`
- Публикация событий: `EventBus::publish(...$user->pullEvents())`

**Возвращает**: Обновленный и сохранённый объект `User`

### updateTariffs(UserId $userId, array<Tariff> $tariffs): User

Обновление тарифов пользователя согласно процессу [UpdateUserTariffs](../processes/update-user-tariffs.md).

**Ответственность Application Layer (UserService)**:
- Валидация входных параметров (не пусто)
- Получение пользователя из репозитория: `UserRepository::findById($userId)`
- Выброс `UserNotFoundException` если пользователь не найден

**Ответственность Domain Layer (User)**:
- Обновление тарифов: `user->replaceTariffs($tariffs)`
- Запись события `UserTariffsUpdated`
- Обновление `updatedAt`

**Ответственность UserService после изменения модели**:
- Сохранение пользователя: `UserRepository::save($user)`
- Публикация событий: `EventBus::publish(...$user->pullEvents())`

**Примечание**: Валидация тарифных кодов и конвертация в объекты Tariff выполняется в Command Handler перед вызовом этого метода.

**Возвращает**: Обновленный и сохранённый объект `User`

## Особенности реализации

* Все методы UserService выполняют **полный цикл операции**: проверка → изменение модели → сохранение → публикация событий
* Сохранение пользователя в репозиторий происходит **в UserService**
* Публикация событий через EventBus происходит **в UserService** после сохранения
* Все проверки уникальности выполняются **до попытки изменить модель**
* Command Handlers только конвертируют входные данные и делегируют работу UserService

## Зависимости

```php
public function __construct(
    private UserRepositoryInterface $repository,
    private EventBusInterface $eventBus
)
```

## Слои взаимодействия

```
HTTP Request / Bot Command
        ↓
Command Handler (Application Layer)
        ├─ Конвертация входных данных (string → Value Objects)
        └─ Делегация в UserService
        ↓
UserService (Application Layer)
        ├─ Проверка бизнес-инвариантов (уникальность, существование)
        ├─ Получение данных из репозитория
        ├─ Вызов методов доменной модели
        ├─ Сохранение результата в UserRepository
        └─ Публикация событий через EventBus
        ↓
User Model (Domain Layer)
        ├─ Вся бизнес-логика
        ├─ Проверка инвариантов
        └─ Запись событий (recordEvent)
        ↓
UserRepository (Infrastructure Layer)
        └─ Сохранение/чтение из БД
```

## Связанные документы

* [Сервисы Account Context](./overview.md)
* [User](../model/user.md)
* [UserId](../model/user-id.md)
* [Messenger](../../shared/models/messenger.md)
* [Tariff Enum](../../shared/models/tariff.md)

## Статус

[x] Создание сервиса `UserService`
[x] Реализация метода `registerUser`
[x] Реализация метода `addMessenger`
[x] Реализация метода `removeMessenger`
[x] Реализация метода `updateTariffs`
[x] Интеграция с UserRepository (save)
[x] Интеграция с EventBus (publish)