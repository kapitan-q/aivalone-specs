# BotResponse Value Object Specification

## Назначение

`BotResponse` — Value Object, представляющий унифицированный формат ответа бота. Создаётся Conversation и преобразуется адаптерами мессенджеров в платформо-специфичный формат.

## Атрибуты

| Поле        | Тип                  | Описание                                                     |
| ----------- | -------------------- | ------------------------------------------------------------ |
| `message`   | `string`             | Текст сообщения для публикации                               |
| `finished`  | `bool`               | Флаг завершения диалога (очистка state)                      |
| `keyboard`  | `?array`             | Универсальный формат inline-кнопок                           |

## Формат Keyboard

```php
// Структура keyboard: array<array<array{text: string, callback_data: string}>>
// Внешний массив — строки, внутренний — кнопки в строке

$keyboard = [
    [
        ['text' => 'Кнопка 1', 'action' => '/setup:lang'],
        ['text' => 'Кнопка 2', 'action' => '/setup:filter'],
    ],

    [
        ['text' => 'Отмена', 'action' => '/setup:cancel'],
    ],
];
```

## Методы

### static message(string $text): self

Создание простого текстового ответа.

**Логика**:
- message = text
- finished = false
- keyboard = null

---

### static messageWithKeyboard(string $text, array $keyboard): self

Создание ответа с inline-кнопками.

---

### static finish(string $text): self

Создание завершающего ответа (диалог будет закрыт).

**Логика**:
- finished = true
- state будет удалён после отправки

---

### static finishWithKeyboard(string $text, array $keyboard): self

Завершающий ответ с кнопками.

---

### withKeyboard(array $keyboard): self

Добавление keyboard к существующему response (immutable, возвращает копию).

---

### asFinished(): self

Пометить response как завершающий (immutable).

---

### Методы доступа

```php
public function getMessage(): string
public function isFinished(): bool
public function getKeyboard(): ?array
public function hasKeyboard(): bool
```

## Примеры использования

```php
// Простое сообщение
$response = BotResponse::message("Привет! Выберите действие:");

// С кнопками
$response = BotResponse::messageWithKeyboard(
    "Выберите язык:",
    [
        [
            ['text' => '🇷🇺 Русский', 'action' => '/setup:lang:lang=ru'],
            ['text' => '🇬🇧 English', 'action' => '/setup:lang:lang=en'],
        ]
    ]
);

// Завершение диалога
$response = BotResponse::finish("Спасибо за использование!");
```

## Инварианты

- [x] BotResponse всегда имеет message
- [x] BotResponse является immutable
- [x] Если finished = true, Router удаляет ConversationState после отправки

## Связанные документы

* [BotRequest](bot-request.md)
* [Conversation](../conversations/abstract-conversation.md)
* [Router Service](../services/router.md)

## Статус реализации

* [ ] Класс BotResponse создан
* [ ] Фабричные методы реализованы
* [ ] Методы-модификаторы реализованы
* [ ] Unit тесты написаны
