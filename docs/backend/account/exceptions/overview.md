# Исключения Account Context

## Назначение

В данном разделе описаны исключения, используемые в Account Context для обработки ошибок бизнес-логики, нарушений инвариантов и валидации данных. Некоторые исключения наследуются от базовых shared-исключений.

## Список исключений

- [UserNotFoundException](./user-not-found-exception.md)
- [DuplicateUserException](./duplicate-user-exception.md)
- [DuplicateMessengerException](./duplicate-messenger-exception.md)
- [MessengerNotFoundException](./messenger-not-found-exception.md)
- [TariffValidationException](./tariff-validation-exception.md)
- [InvalidMessengerException](../../../shared/exceptions/invalid-messenger-exception.md) — наследуется от shared ValidationException.
- [ValidationException](../../../shared/exceptions/validation-exception.md) — базовое shared-исключение.
- [InvalidUuidException](../../../shared/exceptions/invalid-uuid-exception.md) — наследуется от shared ValidationException.

## Связанные документы

- [Account Context Overview](../../overview.md)
- [Shared Context Exceptions](../../../shared/exceptions/overview.md)
