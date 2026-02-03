# ConversationNotFoundException Specification

## Назначение

Исключение `ConversationNotFoundException` выбрасывается когда запрошенный диалог не найден в реестре.

## Наследование

Расширяет `DomainException` из Shared Context.

## Атрибуты

| Поле               | Тип      | Описание                              |
| ------------------ | -------- | ------------------------------------- |
| `conversationCode` | `string` | Код диалога, который не найден        |

## Реализация

```php
final class ConversationNotFoundException extends DomainException
{
    public function __construct(
        public readonly string $conversationCode,
    ) {
        parent::__construct(
            sprintf('Conversation with code "%s" not found', $conversationCode)
        );
    }

    public static function withCode(string $code): self
    {
        return new self($code);
    }
}
```

## Когда выбрасывается

* В `ConversationRegistry::get()` когда диалог с указанным кодом не зарегистрирован
* В `Router` при попытке обработать неизвестную команду

## Обработка

Router перенаправляет на fallback диалог (/help) при получении этого исключения.

## Связанные документы

* [Exceptions Overview](overview.md)
* [Router Service](../services/router.md)
* [ConversationRegistry](../services/conversation-registry.md)

## Статус реализации

* [ ] Класс создан
* [ ] Unit тесты написаны
