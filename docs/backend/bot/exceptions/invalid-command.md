# InvalidCommandException Specification

## Назначение

Исключение `InvalidCommandException` выбрасывается при невалидном формате команды.

## Наследование

Расширяет `DomainException` из Shared Context.

## Атрибуты

| Поле      | Тип      | Описание                              |
| --------- | -------- | ------------------------------------- |
| `command` | `string` | Невалидная команда                    |
| `reason`  | `string` | Причина невалидности                  |

## Реализация

```php
final class InvalidCommandException extends DomainException
{
    public function __construct(
        public readonly string $command,
        public readonly string $reason,
    ) {
        parent::__construct(
            sprintf('Invalid command "%s": %s', $command, $reason)
        );
    }

    public static function emptyCommand(): self
    {
        return new self('', 'command cannot be empty');
    }

    public static function invalidFormat(string $command): self
    {
        return new self($command, 'invalid command format');
    }

    public static function invalidStep(string $command, string $step): self
    {
        return new self($command, sprintf('step "%s" is invalid', $step));
    }
}
```

## Когда выбрасывается

* При парсинге команды с неверным форматом
* При указании несуществующего шага в команде

## Связанные документы

* [Exceptions Overview](overview.md)
* [BotRequest](../models/bot-request.md)

## Статус реализации

* [ ] Класс создан
* [ ] Unit тесты написаны
