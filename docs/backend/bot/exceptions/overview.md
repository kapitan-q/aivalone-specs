# Исключения Bot Context

## Описание

Исключения Bot Context обрабатывают ошибочные ситуации при работе с мессенджерами и диалогами. Все исключения наследуют от базового `DomainException` из Shared Context.

## Список исключений

| Исключение                                                         | Описание                                          |
| ------------------------------------------------------------------ | ------------------------------------------------- |
| [ConversationNotFoundException](conversation-not-found.md)         | Диалог не найден по коду                          |
| [InvalidCommandException](invalid-command.md)                      | Невалидный формат команды                         |
| [EndpointNotFoundException](endpoint-not-found.md)                 | NotificationEndpoint не найден                    |
| [EndpointAlreadyRevokedException](endpoint-already-revoked.md)     | Попытка отозвать уже отозванный endpoint          |
| [MessageDeliveryException](message-delivery.md)                    | Ошибка доставки сообщения                         |
| [ConversationConflictException](conversation-conflict.md)          | Конфликт активного диалога                        |

## Иерархия

```
DomainException (Shared)
├── ConversationNotFoundException
├── InvalidCommandException
├── EndpointNotFoundException
├── EndpointAlreadyRevokedException
├── MessageDeliveryException
└── ConversationConflictException
```

## Обработка исключений

* Все исключения логируются
* Пользователю отправляется дружественное сообщение об ошибке
* Внутренние ошибки не раскрываются пользователю

## Связанные документы

* [Bot Context Overview](../overview.md)
* [DomainException](../../backend/shared/exceptions/domain-exception.md)
