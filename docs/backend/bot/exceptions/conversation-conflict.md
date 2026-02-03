# ConversationConflictException Specification

## Назначение

Исключение `ConversationConflictException` выбрасывается при конфликте активного диалога.

## Наследование

Расширяет `DomainException` из Shared Context.

## Атрибуты

| Поле                    | Тип      | Описание                              |
| ----------------------- | -------- | ------------------------------------- |
| `activeConversation`    | `string` | Код активного диалога                 |
| `requestedConversation` | `string` | Код запрошенного диалога              |

## Реализация

```php
final class ConversationConflictException extends DomainException
{
    public function __construct(
        public readonly string $activeConversation,
        public readonly string $requestedConversation,
    ) {
        parent::__construct(
            sprintf(
                'Cannot start conversation "%s" while "%s" is active',
                $requestedConversation,
                $activeConversation
            )
        );
    }
}
```

## Когда выбрасывается

* В Router когда пользователь пытается перейти к другому диалогу без завершения текущего
* Когда state существует и conversationCode не совпадает с запрошенным

## Логика обработки

Согласно архитектуре из bot-context-architecture.md:
> Если state !== null и state.conversationCode !== conversationCode -> reject

Однако для базовых команд (/start, /help) это правило может быть смягчено — они сбрасывают текущий state.

## Связанные документы

* [Exceptions Overview](overview.md)
* [Router Service](../services/router.md)
* [ConversationState](../models/conversation-state.md)

## Статус реализации

* [ ] Класс создан
* [ ] Unit тесты написаны
