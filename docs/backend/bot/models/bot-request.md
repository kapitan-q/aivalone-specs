# BotRequest Value Object Specification

## Назначение

`BotRequest` — Value Object, представляющий унифицированный формат входящего сообщения от любого мессенджера. Создаётся адаптерами мессенджеров и передаётся в Router для маршрутизации.

## Атрибуты

| Поле              | Тип                  | Описание                                                         |
| ----------------- | -------------------- | ---------------------------------------------------------------- |
| `messenger`       | `Messenger`          | Тип мессенджера (telegram, whatsapp и т.д.)                      |
| `externalUserId`  | `string`             | ID пользователя в мессенджере                                    |
| `externalChatId`  | `string`             | ID чата в мессенджере                                            |
| `command`         | `?string`            | Код команды/диалога (если формат /{code}:{step})                 |
| `step`            | `?string`            | Код шага (если формат /{code}:{step})                            |
| `params`          | `array`              | Параметры из сообщения (param1=value:param2=value)               |
| `message`         | `?string`            | Текст сообщения (для обычных сообщений)                          |

## Формат команд

Унифицированный формат управляющих сообщений для всех мессенджеров (и типов сообщений, в том числе callback_data для telegram):

```
/{conversationCode}:{stepCode}[:{param1=value1:param2=value2}]
```

Примеры:
- `/start` — команда start, step = null
- `/help:faq` — команда help, step = faq
- `/settings:language:lang=ru` — команда settings, step = language, params = {lang: ru}

## Методы

```php
public function construct(
    private Messenger $messenger, 
    private string $externalUserId,
    private string $externalChatId,
    private ?string $command = null,
    private ?string $step = null,
    private ?string $message = null,
    private array $params = []
)
public function getMessenger(): Messenger
public function getExternalUserId(): string
public function getExternalChatId(): string
public function getCommand(): ?string
public function getStep(): ?string
public function getParams(): array
public function getParam(string $key, mixed $default = null): mixed
public function getMessage(): ?string
public function isCommand(): bool
public function hasStep(): bool
```

## Инварианты

- [x] BotRequest всегда имеет messenger
- [x] BotRequest всегда имеет externalUserId и externalChatId
- [x] BotRequest является immutable

## Связанные документы

* [BotResponse](bot-response.md)
* [Router Service](../services/router.md)
* [Messenger Adapter](../infrastructure/messenger-adapter-interface.md)

## Статус реализации

* [ ] Класс BotRequest создан
* [ ] Unit тесты написаны
