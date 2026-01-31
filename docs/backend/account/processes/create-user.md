# CreateUserProcess  Specification

## Назначение

Процесс **регистрации нового пользователя** в системе Aivalone. 
Отвечает за создание пользователя с минимальной информацией (messenger) и присвоение начального тарифа.

## Предусловия

* Пользователь должен предоставить:
  * `messengerCode` — код мессенджера (например, 'TELEGRAM')
  * `messengerId` — идентификатор пользователя в выбранном мессенджере

* Messenger должен быть уникальным в системе (не должно быть другого пользователя с таким же `messengerCode` и `messengerId`).

## Поток выполнения

1. **Обработка команды** (Command Handler)
   * Проверка наличия `messengerCode` и `messengerId`.
   * Конвертация `messengerCode` в объект `Messenger` (через `Messenger::fromCode()`)
   * Делегация в `UserService::registerUser()`

2. **Проверка уникальности** (UserService — Application Layer)
   * **Проверка уникальности комбинации `messenger` + `messengerId`** через `UserRepository::findByMessenger()`.
   * Выброс `DuplicateUserException` если пользователь уже существует.

3. **Создание пользователя** (Domain Layer)
   * Вызов `User::registerUser(Messenger $messenger, string $messengerId)`.
   * Метод генерирует новый `UserId` через `UserId::generate()`.
   * Создается новый объект `User` с инициализацией внутренней структуры согласно спецификации модели.
   * Автоматически записывается событие `UserRegistered` в агрегат.

4. **Сохранение пользователя** (UserService → Infrastructure Layer)
   * Сохранение пользователя через `UserRepository::save(User)`.

5. **Публикация событий** (UserService)
   * Вызов `user->pullEvents()` для получения накопленных событий.
   * Публикация событий через `EventBus::publish()`.

## Исключения / Ошибки

* Если `messengerCode` или `messengerId` отсутствуют — генерация [ValidationException](../../shared/exceptions/validation-exception.md).
* Если пользователь с данным `messengerCode` и `messengerId` уже существует — генерация [DuplicateUserException](../exceptions/duplicate-user-exception.md).

## Примечания

* Регистрация возможна только через один мессенджер (по умолчанию) — дальнейшие мессенджеры добавляются отдельным процессом `AddUserMessenger`.
* `UserId` генерируется **внутри** метода `User::registerUser()`, а не передается как параметр.
* При регистрации пользователь автоматически получает тариф `FREE` (из Shared Context enum).
* Только одно событие `UserRegistered` публикуется при регистрации (содержит информацию о мессенджерах).

## Связанные документы

* [Процессы](./overview.md)
* [User](../model/user.md)
* [UserService](../services/user-service.md)
* [AddUserMessenger](./add-user-messenger.md)