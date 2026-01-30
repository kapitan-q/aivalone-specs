# UserService Specification

## Назначение

Сервис **UserService** управляет всей бизнес-логикой, связанной с пользователями в системе Aivalone.
Отвечает за создание пользователей, управление тарифами и мессенджерами, а также взаимодействие с репозиторием и публикацию событий.

## Основные методы

### registerUser(Messanger messenger, messengerId)

Регистрация пользователя согласно спецификации процесса [CreateUser](../processes/create-user.md)

### addMessenger(UserId userId, Messanger messenger, messengerId)

Добавление нового мессенджера согласно процессу [AddUserMessenger](../processes/add-user-messenger.md)

### removeMessenger(UserId userId, Messanger messenger, messengerId)

Удаление указанного мессенджера у пользователя согласно процессу [RemoveUserMessenger](../processes/remove-user-messenger.md)

### updateTariffs(UserId userId, tariffCodes[])

Обновление списка тарифов пользователя согласно процессу [UpdateUserTariffs](../processes/update-user-tariffs.md)

## Особенности

* Все изменения пользователя фиксируют дату изменения (`updatedAt`) и сохраняют дату создания (`createdAt`).
* Сервис оперирует только агрегатом [User](../model/user.md) и вложенной моделью `UserMessenger`.
* Все бизнес-инварианты модели `User` гарантируются в сервисе, включая уникальность мессенджеров и корректность тарифов.
* Публикация событий осуществляется через EventBus

## Слои взаимодействия

* **Application Layer** — сервис находится в этом слое.
* **Domain Layer** — использует агрегат User и вложенные модели.
* **Infrastructure Layer** — взаимодействует с `UserRepository`.

## Связанные документы

* [Сервисы Account Context](./overview.md)
* [User](../model/user.md)
* [UserId](../model/user-id.md)
* [Messenger](../../shared/models/messenger.md)

## Статус

[] Создание сервиса `UserService`
[] Реалиализация метода `registerUser`
[] Реалиализация метода `addMessenger`
[] Реалиализация метода `removeMessenger`
[] Реалиализация метода `updateTariffs`