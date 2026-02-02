# Задача 0053: BotRequest Value Object

## Контекст

BotRequest — унифицированный формат входящего сообщения от любого мессенджера. Абстрагирует специфику Telegram/WhatsApp/Discord. Адаптеры мессенджеров преобразуют нативный формат (включая callback_query для Telegram) в единый BotRequest.

## Цель

Создать Value Object `BotRequest` для представления входящего сообщения.

## Спецификация

- [BotRequest](../backend/bot/models/bot-request.md)

## Файлы для создания

```
src/Context/Bot/Domain/Model/BotRequest.php
tests/Unit/Context/Bot/Domain/Model/BotRequestTest.php
```

## Важно

**Нет понятия callback!** Callback — это telegram-специфика. Для BotRequest всё унифицировано:
- Если пришла команда (из текста или callback_data) — она парсится в command/step/params
- Если пришло обычное сообщение — оно в поле message

Формат команды: `/{conversationCode}:{stepCode}[:{param1=value1:param2=value2}]`

Примеры:
- `/start` → command=start, step=null, params=[]
- `/help:faq` → command=help, step=faq, params=[]
- `/settings:language:lang=ru` → command=settings, step=language, params={lang: ru}

## Требования

### BotRequest

```php
<?php

declare(strict_types=1);

namespace App\Context\Bot\Domain\Model;

use App\Context\Shared\Domain\Model\Messenger;

/**
 * Унифицированный формат входящего сообщения.
 *
 * Адаптеры мессенджеров преобразуют нативный формат в BotRequest:
 * - Telegram message с текстом /start → BotRequest с command=start
 * - Telegram callback_query с data=/settings:lang → BotRequest с command=settings, step=lang
 * - WhatsApp сообщение → аналогично
 */
final readonly class BotRequest
{
    /**
     * @param array<string, string> $params Параметры из команды (param1=value1:param2=value2)
     */
    private function __construct(
        private Messenger $messenger,
        private string $externalUserId,
        private string $externalChatId,
        private ?string $command,
        private ?string $step,
        private ?string $message,
        private array $params,
    ) {}

    /**
     * Создание из распарсенной команды.
     * Используется адаптерами после парсинга команды.
     *
     * @param array<string, string> $params
     */
    public static function fromCommand(
        Messenger $messenger,
        string $externalUserId,
        string $externalChatId,
        string $command,
        ?string $step = null,
        array $params = [],
    ): self {
        return new self(
            messenger: $messenger,
            externalUserId: $externalUserId,
            externalChatId: $externalChatId,
            command: $command,
            step: $step,
            message: null,
            params: $params,
        );
    }

    /**
     * Создание из обычного текстового сообщения (не команда).
     */
    public static function fromMessage(
        Messenger $messenger,
        string $externalUserId,
        string $externalChatId,
        string $message,
    ): self {
        return new self(
            messenger: $messenger,
            externalUserId: $externalUserId,
            externalChatId: $externalChatId,
            command: null,
            step: null,
            message: $message,
            params: [],
        );
    }

    public function getMessenger(): Messenger
    {
        return $this->messenger;
    }

    public function getExternalUserId(): string
    {
        return $this->externalUserId;
    }

    public function getExternalChatId(): string
    {
        return $this->externalChatId;
    }

    public function getCommand(): ?string
    {
        return $this->command;
    }

    public function getStep(): ?string
    {
        return $this->step;
    }

    /**
     * @return array<string, string>
     */
    public function getParams(): array
    {
        return $this->params;
    }

    public function getParam(string $key, mixed $default = null): mixed
    {
        return $this->params[$key] ?? $default;
    }

    public function getMessage(): ?string
    {
        return $this->message;
    }

    /**
     * Это команда? (а не обычное сообщение)
     */
    public function isCommand(): bool
    {
        return $this->command !== null;
    }

    /**
     * Есть ли конкретный шаг в команде?
     */
    public function hasStep(): bool
    {
        return $this->step !== null;
    }
}
```

## Вспомогательный класс для парсинга команд

Этот класс используется адаптерами для парсинга строки команды:

```php
<?php

declare(strict_types=1);

namespace App\Context\Bot\Domain\Model;

/**
 * Парсер команд формата /{code}:{step}:{param1=value1:param2=value2}
 */
final readonly class CommandParser
{
    /**
     * @return array{command: string, step: ?string, params: array<string, string>}|null
     */
    public static function parse(string $input): ?array
    {
        // Убираем ведущий слэш если есть
        $input = ltrim($input, '/');

        if ($input === '') {
            return null;
        }

        $parts = explode(':', $input);

        $command = array_shift($parts);
        $step = null;
        $params = [];

        // Если есть ещё части
        if (count($parts) > 0) {
            $nextPart = array_shift($parts);

            // Если это не параметр (не содержит =), это step
            if (!str_contains($nextPart, '=')) {
                $step = $nextPart;
            } else {
                // Это параметр, возвращаем обратно
                array_unshift($parts, $nextPart);
            }
        }

        // Остальное — параметры
        foreach ($parts as $part) {
            if (str_contains($part, '=')) {
                [$key, $value] = explode('=', $part, 2);
                if ($key !== '') {
                    $params[$key] = $value;
                }
            }
        }

        return [
            'command' => $command,
            'step' => $step,
            'params' => $params,
        ];
    }
}
```

## Тесты

### BotRequest
- [x] Создание BotRequest из команды без step
- [x] Создание BotRequest из команды с step
- [x] Создание BotRequest из команды с step и params
- [x] Создание BotRequest из обычного сообщения
- [x] isCommand() возвращает true для команд
- [x] isCommand() возвращает false для сообщений
- [x] hasStep() возвращает true/false корректно
- [x] getParam() возвращает значение или default

### CommandParser
- [x] Парсинг простой команды `/start` → command=start
- [x] Парсинг команды со step `/help:faq` → command=help, step=faq
- [x] Парсинг команды с params `/setup:lang:lang=ru` → params={lang: ru}
- [x] Парсинг команды с несколькими params
- [x] Парсинг пустой строки возвращает null

## Зависимости

- `App\Context\Shared\Domain\Model\Messenger` (задача 0006)

## Definition of Done

- [x] Класс BotRequest создан
- [x] Класс CommandParser создан
- [x] Все методы реализованы
- [x] Unit-тесты написаны и проходят
- [x] Код соответствует PSR-12
- [x] PHPStan level 9 без ошибок
