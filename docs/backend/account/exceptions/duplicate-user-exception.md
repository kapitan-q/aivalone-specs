# DuplicateUserException Specification

## Назначение

`DuplicateUserException` — исключение, выбрасываемое при попытке создать пользователя с уже существующим уникальным сочетанием мессенджера и идентификатора.

## Наследование

Наследник базового `DomainException` из Shared Context.

## Область применения

* Процесс регистрации нового пользователя
* Проверка уникальности пользователя по мессенджеру и messengerId

## Когда выбрасывается

* В системе уже существует пользователь с данным `messengerCode` и `messengerId`

## Инварианты

* В системе не допускается дублирование пользователей по мессенджеру и messengerId

## Связанные документы

* [Account Context Overview](../../overview.md)
* [Shared Context Exceptions](../../../shared/exceptions/overview.md)

## Статус

* [] Реализация исключения DuplicateUserException
