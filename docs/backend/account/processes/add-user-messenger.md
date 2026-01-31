# AddUserMessenger Process Specification

## Назначение

Процесс **добавления нового мессенджера к существующему пользователю** в системе Aivalone.
Позволяет расширить пользователя дополнительными каналами связи.

## Предусловия

* Пользователь должен существовать в системе (идентифицируется по `UserId`).
* Должны быть предоставлены данные нового мессенджера:
  * `messenger` — код мессенджера (Messenger)
  * `messengerId` — идентификатор пользователя в выбранном мессенджере
* Messenger должен быть уникальным для системы (не должно быть другого пользователя с таким же `messenger` и `messengerId`).

## Поток выполнения

1. **Обработка команды** (Command Handler)
   * Проверка наличия `userId`, `messengerCode`, и `messengerId`.
   * Конвертация `userId` в `UserId`, `messengerCode` в `Messenger`.
   * Делегация в `UserService::addMessenger()`

2. **Проверка уникальности и получение пользователя** (UserService — Application Layer)
   * **Проверка уникальности комбинации `messenger` + `messengerId` в системе** через `UserRepository::findByMessenger()`.
   * Выброс `DuplicateMessengerException` если мессенджер уже привязан к другому пользователю.
   * Получение пользователя из репозитория через `UserRepository::findById(UserId)`.
   * Выброс `UserNotFoundException` если пользователь не найден.

3. **Добавление мессенджера** (Domain Layer)
   * Вызов метода `User::addMessenger(Messenger $messenger, string $messengerId)`.
   * Метод добавляет новый объект `UserMessenger` к пользователю согласно модели.
   * Автоматически записывается событие `UserMessengersUpdated` в агрегат.

4. **Сохранение изменений пользователя** (UserService → Infrastructure Layer)
   * Сохранение пользователя и вложенных данных через `UserRepository::save(User)`.

5. **Публикация событий** (UserService)
   * Вызов `user->pullEvents()` для получения накопленных событий.
   * Публикация событий через `EventBus::publish()`.

## Исключения / Ошибки

* Пользователь не найден — выбрасывается [UserNotFoundException](../exceptions/user-not-found-exception.md).
* Нарушение уникальности мессенджера — выбрасывается [DuplicateMessengerException](../exceptions/duplicate-messenger-exception.md).
* Недостаточные данные для мессенджера — выбрасывается [ValidationException](../../shared/exceptions/validation-exception.md).

## Примечания

* Процесс является частью **Application Layer**.
* Command Handler конвертирует входные данные и делегирует работу в UserService.
* UserService выполняет полный цикл: проверка уникальности → получение пользователя → изменение модели → сохранение → публикация событий.
* Проверка уникальности мессенджера выполняется **ДО изменения модели** (в UserService).

## Связанные документы

* [Процессы](./overview.md)
* [User, UserMessenger](../model/user.md)
* [UserId](../model/user-id.md)
* [UserService](../services/user-service.md)
* [UserMessengersUpdated](../events/user-messengers-updated.md)
* [Messenger](../../shared/models/messenger.md)