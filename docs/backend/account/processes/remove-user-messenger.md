# RemoveUserMessenger Process Specification

## Назначение

Процесс **удаления мессенджера у существующего пользователя** в системе Aivalone.
Позволяет управлять каналами связи пользователя, удаляя ненужные или устаревшие мессенджеры.

## Предусловия

* Пользователь должен существовать в системе (идентифицируется по `UserId`).
* Мессенджер должен быть привязан к пользователю:

  * `messengerCode` — код мессенджера (например, 'TELEGRAM', 'WHATSAPP')
  * `messengerId` — идентификатор пользователя в выбранном мессенджере

## Поток выполнения

1. **Обработка команды** (Command Handler)
   * Проверка наличия `userId`, `messengerCode`, и `messengerId`.
   * Конвертация `userId` в `UserId`, `messengerCode` в `Messenger`.
   * Делегация в `UserService::removeMessenger()`

2. **Получение пользователя** (UserService — Application Layer)
   * Получение пользователя из репозитория через `UserRepository::findById(UserId)`.
   * Выброс `UserNotFoundException` если пользователь не найден.

3. **Удаление мессенджера** (Domain Layer)
   * Вызов метода `User::removeMessenger(Messenger $messenger, string $messengerId)`.
   * Метод удаляет объект `UserMessenger` из коллекции пользователя согласно модели.
   * Если мессенджер не найден, выбросится `MessengerNotFoundException`.
   * Если это последний оставшийся мессенджер, выбросится `AtLeastOneMessengerRequiredException` (нарушение инварианта).
   * Автоматически записывается событие `UserMessengersUpdated` в агрегат.

4. **Сохранение изменений пользователя** (UserService → Infrastructure Layer)
   * Сохранение пользователя и вложенных данных через `UserRepository::save(User)`.

5. **Публикация событий** (UserService)
   * Вызов `user->pullEvents()` для получения накопленных событий.
   * Публикация событий через `EventBus::publish()`.

## Исключения / Ошибки

* Пользователь не найден — выбрасывается [UserNotFoundException](../exceptions/user-not-found-exception.md).
* Мессенджер не найден у пользователя — выбрасывается [MessengerNotFoundException](../exceptions/messenger-not-found-exception.md).
* Попытка удалить последний мессенджер — выбрасывается [AtLeastOneMessengerRequiredException](../exceptions/at-least-one-messenger-required-exception.md).

## Примечания

* Процесс является частью **Application Layer**.
* Command Handler конвертирует входные данные и делегирует работу в UserService.
* UserService выполняет полный цикл: получение пользователя → изменение модели → сохранение → публикация событий.
* User Model гарантирует, что при попытке удалить несуществующий мессенджер будет выброшено `MessengerNotFoundException`.

## Связанные документы

* [Процессы](./overview.md)
* [User](../model/user.md)
* [UserService](../services/user-service.md)