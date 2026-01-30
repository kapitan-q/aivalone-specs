# DuplicateMessengerException Specification

## Назначение

`DuplicateMessengerException` — исключение, выбрасываемое при попытке добавить мессенджер, который уже привязан к другому пользователю.

## Наследование

Наследник базового `DomainException` из Shared Context.

## Область применения

* Процесс добавления нового мессенджера пользователю
* Проверка уникальности мессенджера в системе

## Когда выбрасывается

* В системе уже существует пользователь с данным `messenger` и `messengerId`

## Инварианты

* В системе не допускается дублирование мессенджеров между пользователями

## Связанные документы

* [Account Context Overview](../../overview.md)
* [Shared Context Exceptions](../../../shared/exceptions/overview.md)

## Статус

* [] Реализация исключения DuplicateMessengerException
