# InvalidMessengerException Specification

## Назначение


`InvalidMessengerException` — специализированное исключение, наследник [ValidationException](./validation-exception.md). Выбрасывается, если указанный код мессенджера **не поддерживается системой** или не может быть сопоставлен с `Messenger enum`.

## Область применения

* `Messenger::fromCode(string $code)`
* Валидация входных данных на границе системы

## Когда выбрасывается

* Передан неизвестный `messengerCode`
* Передана пустая или некорректная строка
* Код не входит в список поддерживаемых `Messenger`

## Инварианты

* В домене используются **только валидные мессенджеры**
* Расширение списка мессенджеров происходит через enum
* Является частным случаем `ValidationException`

## Связанные документы

* [Shared Context Overview](../overview.md)
* [Исключения](./overview.md)

## Статус

* [] Реализация исключения InvalidMessengerException