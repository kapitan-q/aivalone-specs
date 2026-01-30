# UserId Value Object Specification

## Назначение

`UserId` — доменный **Value Object**, представляющий уникальный идентификатор пользователя в системе Aivalone.

Используется как **единственный способ идентификации пользователя** внутри домена и между контекстами.
Основан на [UUID](./../../shared/models/uuid.md) и обеспечивает неизменяемость, типобезопасность и корректность идентификаторов.

## Роль в архитектуре

* Используется:
  * в агрегате [User](./user.md)
  * в процессах ([CreateUser](../processes/create-user.md), [AddUserMessenger](../processes/add-user-messenger.md), [RemoveUserMessenger](../processes/remove-user-messenger.md), [UpdateUserTariffs](../processes/update-user-tariffs.md))
  * в доменных событиях ([UserRegistered](../events/user-registered.md), [UserMessengersUpdated](../events/user-messengers-updated.md), [UserTariffsUpdated](../events/user-tariffs-updated.md) и др.)

## Основа реализации

* Расширяет базовый [UUID](./../../shared/models/uuid.md)
* Поддерживает UUID v4

## Инварианты

* `UserId` **всегда валиден**
* Значение `UserId`:
  * неизменяемо
  * не может быть `null`
  * не может быть пустой строкой
* В домене **не существует пользователя без `UserId`**

## Ограничения и правила

* Запрещено:
  * передавать `string` вместо `UserId` внутри домена
  * изменять значение после создания
* Допускается:
  * сериализация / десериализация через `toString()`

## Пример использования (conceptual)

* Создание нового пользователя:
  * `UserId::generate()`

* Поиск пользователя:
  * `UserRepository::findById(UserId $id)`

* Сравнение:
  * `$user->getId()->equals($otherUserId)`

* [InvalidUuidException](./../../shared/exceptions/InvalidUuidException.md) — при попытке создать `UserId` из некорректной строки

## Связанные документы

* [Account Context Overview](../overview.md)

## Статус

* [] Определён `UserId` Value Object
* [] Используется во всех процессах Account Context
