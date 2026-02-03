# Conversations (Диалоги) Bot Context

## Описание

Conversations — это диалоги с пользователем, реализованные как Finite State Machine (FSM). Каждый диалог состоит из шагов и управляет переходами между ними.

## Интерфейс Conversation

Каждый диалог реализует `ConversationInterface`:

```php
interface ConversationInterface
{
    /**
     * Обработка текущего шага
     */
    public function handle(?string $stepCode, ?string $message, ?array $params): BotResponse;

    /**
     * Переход к следующему шагу
     */
    public function next(string $nextStepCode): BotResponse;

    /**
     * Отмена диалога
     */
    public function cancel(): BotResponse;

    /**
     * Код диалога (уникальный идентификатор)
     */
    public function getCode(): string;

    /**
     * Команда для вызова (/start, /help)
     */
    public function getCommand(): string;

    /**
     * Краткое описание
     */
    public function getDescription(): string;

    /**
     * Текущий шаг (для сохранения в state)
     */
    public function getCurrentStep(): string;

    /**
     * Данные контекста (для сохранения в state)
     */
    public function getContextData(): array;

    /**
     * Установка данных контекста (при загрузке из state)
     */
    public function setContextData(array $data): void;

    /**
     * Информация для меню (null если не отображается)
     * @return array{icon: string, name: string, link: string}|null
     */
    public function getMenuInfo(): ?array;

    /**
     * Генерация ссылки на шаг
     */
    public function getLink(?string $step = null, array $params = []): string;

    /**
     * Ссылка для отмены
     */
    public function getCancelLink(): string;
}
```

## Базовый класс AbstractConversation

Все диалоги наследуют от `AbstractConversation`:

| Диалог                                           | Описание                                  |
| ------------------------------------------------ | ----------------------------------------- |
| [AbstractConversation](abstract-conversation.md) | Базовый класс с общей логикой             |
| [StartConversation](start-conversation.md)       | Диалог /start                             |
| [HelpConversation](help-conversation.md)         | Диалог /help                              |

## Формат ссылок

Унифицированный формат команд:

```
/{conversationCode}:{stepCode}[:{param1=value1:param2=value2}]
```

Примеры:
- `/start` — вызов диалога start
- `/help:faq` — вызов шага faq диалога help
- `/settings:lang:lang=ru` — вызов шага lang с параметром

## Связанные документы

* [Bot Context Overview](../overview.md)
* [ConversationState](../models/conversation-state.md)
* [Router](../services/router.md)
