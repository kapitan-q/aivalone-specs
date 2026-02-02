# Задача 0054: BotResponse Value Object

## Контекст

BotResponse — унифицированный формат исходящего сообщения для любого мессенджера. Адаптеры преобразуют его в нативный формат (Telegram InlineKeyboard, WhatsApp buttons и т.д.).

## Цель

Создать Value Object `BotResponse` для представления исходящего сообщения.

## Спецификация

- [BotResponse](../backend/bot/models/bot-response.md)

## Файлы для создания

```
src/Context/Bot/Domain/Model/BotResponse.php
tests/Unit/Context/Bot/Domain/Model/BotResponseTest.php
```

## Важно

**Нет понятия callback!** Используем `action`:
- Если action начинается с `/` — это команда (Telegram конвертирует в callback_data)
- Если action начинается с `url:` — это внешняя ссылка
- Если action начинается с `web_app:` — это Telegram WebApp

Формат keyboard в спецификации:
```php
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

## Требования

### BotResponse

```php
<?php

declare(strict_types=1);

namespace App\Context\Bot\Domain\Model;

/**
 * Унифицированный формат ответа бота.
 *
 * Создаётся Conversation и преобразуется адаптерами
 * мессенджеров в платформо-специфичный формат.
 */
final readonly class BotResponse
{
    /**
     * @param array<array<array{text: string, action: string}>>|null $keyboard
     */
    private function __construct(
        private string $message,
        private bool $finished,
        private ?array $keyboard,
    ) {}

    /**
     * Создание простого текстового ответа.
     */
    public static function message(string $text): self
    {
        return new self(
            message: $text,
            finished: false,
            keyboard: null,
        );
    }

    /**
     * Создание ответа с inline-кнопками.
     *
     * @param array<array<array{text: string, action: string}>> $keyboard
     */
    public static function messageWithKeyboard(string $text, array $keyboard): self
    {
        return new self(
            message: $text,
            finished: false,
            keyboard: $keyboard,
        );
    }

    /**
     * Создание завершающего ответа (диалог будет закрыт, state удалён).
     */
    public static function finish(string $text): self
    {
        return new self(
            message: $text,
            finished: true,
            keyboard: null,
        );
    }

    /**
     * Завершающий ответ с кнопками.
     *
     * @param array<array<array{text: string, action: string}>> $keyboard
     */
    public static function finishWithKeyboard(string $text, array $keyboard): self
    {
        return new self(
            message: $text,
            finished: true,
            keyboard: $keyboard,
        );
    }

    /**
     * Добавление keyboard к существующему response (immutable).
     *
     * @param array<array<array{text: string, action: string}>> $keyboard
     */
    public function withKeyboard(array $keyboard): self
    {
        return new self(
            message: $this->message,
            finished: $this->finished,
            keyboard: $keyboard,
        );
    }

    /**
     * Пометить response как завершающий (immutable).
     */
    public function asFinished(): self
    {
        return new self(
            message: $this->message,
            finished: true,
            keyboard: $this->keyboard,
        );
    }

    public function getMessage(): string
    {
        return $this->message;
    }

    public function isFinished(): bool
    {
        return $this->finished;
    }

    /**
     * @return array<array<array{text: string, action: string}>>|null
     */
    public function getKeyboard(): ?array
    {
        return $this->keyboard;
    }

    public function hasKeyboard(): bool
    {
        return $this->keyboard !== null && count($this->keyboard) > 0;
    }
}
```

## Формат Action

Action определяет поведение кнопки:

| Формат | Описание | Пример |
|--------|----------|--------|
| `/command` | Команда бота | `/start`, `/help:faq` |
| `url:https://...` | Внешняя ссылка | `url:https://example.com` |
| `web_app:https://...` | Telegram WebApp | `web_app:https://app.aivalone.com` |

Адаптер мессенджера конвертирует action в нативный формат:
- Telegram: `/start` → `callback_data: "/start"`, `url:...` → `url: ...`, `web_app:...` → `web_app: {url: ...}`
- WhatsApp: может использовать другой формат кнопок

## Примеры использования

```php
// Простое сообщение
$response = BotResponse::message("Привет! Выберите действие:");

// С кнопками
$response = BotResponse::messageWithKeyboard(
    "Выберите язык:",
    [
        [
            ['text' => '🇷🇺 Русский', 'action' => '/settings:lang:lang=ru'],
            ['text' => '🇬🇧 English', 'action' => '/settings:lang:lang=en'],
        ],
    ]
);

// С WebApp кнопкой
$response = BotResponse::messageWithKeyboard(
    "Откройте приложение:",
    [
        [
            ['text' => '🚀 Открыть', 'action' => 'web_app:https://app.aivalone.com'],
        ],
    ]
);

// Завершение диалога
$response = BotResponse::finish("Спасибо за использование!");

// Fluent API
$response = BotResponse::message("Текст")
    ->withKeyboard([...])
    ->asFinished();
```

## Тесты

### BotResponse

- [x] Создание простого текстового ответа (message)
- [x] Создание ответа с клавиатурой (messageWithKeyboard)
- [x] Создание завершающего ответа (finish)
- [x] Создание завершающего ответа с клавиатурой (finishWithKeyboard)
- [x] isFinished() возвращает корректное значение
- [x] hasKeyboard() возвращает true/false корректно
- [x] withKeyboard() создаёт новый объект (immutable)
- [x] asFinished() создаёт новый объект (immutable)

## Зависимости

Нет внешних зависимостей.

## Definition of Done

- [x] Класс BotResponse создан
- [x] Все фабричные методы реализованы
- [x] Методы-модификаторы реализованы (immutable)
- [x] Unit-тесты написаны и проходят
- [x] Код соответствует PSR-12
- [x] PHPStan level 9 без ошибок
