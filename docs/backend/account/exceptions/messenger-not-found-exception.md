# MessengerNotFoundException Specification

## Назначение

`MessengerNotFoundException` — исключение, выбрасываемое при попытке удалить или получить мессенджер, который не привязан к пользователю.

## Наследование

Наследник базового `DomainException` из Shared Context.

## Область применения

* Процесс удаления мессенджера у пользователя
* Получение информации о мессенджере пользователя

## Когда выбрасывается

* У пользователя не найден указанный мессенджер по `messengerCode` и `messengerId`

## Инварианты

* Операции над несуществующими мессенджерами невозможны

## Связанные документы

* [Account Context Overview](../../overview.md)
* [Shared Context Exceptions](../../../shared/exceptions/overview.md)

## Статус

* [] Реализация исключения MessengerNotFoundException
