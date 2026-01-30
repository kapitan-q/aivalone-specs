# UserService Specification

## Назначение

Сервис **UserService** управляет всей бизнес-логикой, связанной с пользователями в системе Aivalone.
Отвечает за создание пользователей, управление тарифами и мессенджерами, а также взаимодействие с репозиторием и публикацию событий.

## Основные методы

### registerUser(messengerCode, messengerId)

Регистрация пользователя согласно спецификации процесса `CreateUser`

### addMessenger(userId, messengerCode, messengerId)

Добавление нового мессенджера согласно процессу `AddUserMessengerProcess`

### removeMessenger(userId, messengerCode, messengerId)

Удаление указанного мессенджера у пользователя согласно процессу `RemoveUserMessengerProcess`

### updateTariffs(userId, tariffCodes[])

Обновление списка тарифов пользователя согласно процессу `UpdateUserTariffs`

## Особенности

* Все изменения пользователя фиксируют дату изменения (`updatedAt`) и сохраняют дату создания (`createdAt`).
* Сервис оперирует только агрегатом `User` и вложенной моделью `UserMessenger`.
* Все бизнес-инварианты модели User гарантируются в сервисе, включая уникальность мессенджеров и корректность тарифов.
* Публикация событий осуществляется через EventBus

## Слои взаимодействия

* **Application Layer** — сервис находится в этом слое.
* **Domain Layer** — использует агрегат User и вложенные модели.
* **Infrastructure Layer** — взаимодействует с `UserRepository`.

## Связанные документы

* [Сервисы Account Context](./overview.md)
* [User](../model/user.md)
* [CreateUserProcess](../processes/create-user.md)
* [AddMessengerProcess](../processes/add-user-messenger.md)
* [RemoveMessengerProcess](../processes/remove-user-messenger.md)
* [UpdateUserTariffs](../processes/update-user-tariffs.md)

## Статус

[] Создание сервиса `UserService`
[] Реалиализация метода `registerUser`
[] Реалиализация метода `addMessenger`
[] Реалиализация метода `removeMessenger`
[] Реалиализация метода `updateTariffs`