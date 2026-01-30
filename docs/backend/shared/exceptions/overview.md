# Исключения Shared Context

## Назначение

В данном разделе описываются исключения, используемые в Shared Context для обеспечения типобезопасности и валидации на границах домена.

## Основные исключения

- [DomainException](./domain-exception.md) — базовое исключение для ошибок бизнес-логики и нарушений инвариантов.
- [ValidationException](./validation-exception.md) — базовое исключение для ошибок валидации.
- [InvalidMessengerException](./invalid-messenger-exception.md) — выбрасывается при попытке создать Messenger с неподдерживаемым кодом.
- [InvalidUuidException](./invalid-uuid-exception.md) — выбрасывается при передаче невалидного UUID.

## Связанные документы

- [Shared Context Overview](../overview.md)
