# AddUserMessenger Process Specification

## Назначение

Процесс **добавления нового мессенджера к существующему пользователю** в системе Aivalone.
Позволяет расширить пользователя дополнительными каналами связи.

## Предусловия

* Пользователь должен существовать в системе (идентифицируется по `UserId`).
* Должны быть предоставлены данные нового мессенджера:

  * `messengerCode` — код мессенджера (например, 'TELEGRAM', 'WHATSAPP')
  * `messengerId` — идентификатор пользователя в выбранном мессенджере
* Messenger должен быть уникальным для системы (не должно быть другого пользователя с таким же `messengerCode` и `messengerId`).

## Поток выполнения

1. **Валидация данных**
   * Проверка наличия `messengerCode` и `messengerId`.
   * Проверка существования пользователя по `UserId`.
   * Проверка уникальности комбинации `messengerCode` + `messengerId`.

2. **Добавление мессенджера**
   * Вызов сервиса `UserService::addMessenger(userId, messengerCode, messengerId)`.
   * `UserService` создает новый объект `UserMessenger` и добавляет его к пользователю согласно модели.

3. **Сохранение изменений пользователя**
   * Сохранение пользователя и вложенных данных через `UserRepository::save(User)`.

4. **Публикация событий**
   * Публикация накопленных событий доменной модели

## Исключения / Ошибки

* Пользователь не найден — выбрасывается `UserNotFoundException`.
* Нарушение уникальности мессенджера — выбрасывается `DuplicateMessengerException`.
* Недостаточные данные для мессенджера — выбрасывается `ValidationException`.

## Примечания

* Процесс является частью **Application Layer**.
* `UserService` отвечает за все бизнес-инварианты модели `User`.

## Связанные документы

* [Процессы](./overview.md)
* [User, UserMessenger](../model/user.md)
* [UserService](../services/user-service.md)
* [UserMessengersUpdated](../events/user-messengers-updated.md)