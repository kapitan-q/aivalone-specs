# AbstractConversation Specification

## Назначение

`AbstractConversation` — базовый класс для всех диалогов. Реализует общую логику FSM и предоставляет удобные методы для создания конкретных диалогов.

## Реализация

```php
abstract class AbstractConversation implements ConversationInterface
{
    protected string $currentStep;
    protected array $contextData = [];

    /**
     * Код диалога (должен быть уникальным)
     */
    abstract public function getCode(): string;

    /**
     * Начальный шаг диалога
     */
    abstract protected function getInitialStep(): string;

    public function getCommand(): string
    {
        return '/' . $this->getCode();
    }

    public function handle(?string $stepCode, ?string $message, ?array $params): BotResponse
    {
        $step = $stepCode ?? $this->getInitialStep();
        $this->currentStep = $step;

        $method = 'step' . ucfirst($step);

        if (method_exists($this, $method)) {
            return $this->$method($message, $params ?? []);
        }

        return $this->stepCancel();
    }

    public function next(string $nextStepCode): BotResponse
    {
        $this->currentStep = $nextStepCode;

        return $this->handle($nextStepCode, null, null);
    }

    protected function stepCancel(): BotResponse
    {
        $this->contextData = [];

        return $this->next($this->getInitialStep());
    }

    public function cancel(): BotResponse
    {
        return $this->stepCancel();
    }

    public function getMenuInfo(): ?array
    {
        return null;
    }

    public function getLink(?string $step = null, array $params = []): string
    {
        $link = $this->getCommand();

        if ($step !== null) {
            $link .= ':' . $step;
        }

        if (!empty($params)) {
            $paramStrings = [];
            foreach ($params as $key => $value) {
                $paramStrings[] = $key . '=' . $value;
            }
            $link .= ':' . implode(':', $paramStrings);
        }

        return $link;
    }

    public function getCancelLink(): string
    {
        return $this->getLink('cancel');
    }

    public function getCurrentStep(): string
    {
        return $this->currentStep;
    }

    public function getContextData(): array
    {
        return $this->contextData;
    }

    public function setContextData(array $data): void
    {
        $this->contextData = $data;
    }

    protected function setContextParam(string $key, mixed $value): void
    {
        $this->contextData[$key] = $value;
    }

    protected function getContextParam(string $key, mixed $default = null): mixed
    {
        return $this->contextData[$key] ?? $default;
    }
}
```

## Создание нового диалога

```php
final class SettingsConversation extends AbstractConversation
{
    public function getCode(): string
    {
        return 'settings';
    }

    public function getDescription(): string
    {
        return 'Настройки аккаунта';
    }

    protected function getInitialStep(): string
    {
        return 'menu';
    }

    public function getMenuInfo(): ?array
    {
        return [
            'icon' => '⚙️',
            'name' => 'Настройки',
            'link' => $this->getCommand(),
        ];
    }

    protected function stepMenu(?string $message, array $params): BotResponse
    {
        return BotResponse::messageWithKeyboard(
            '⚙️ Настройки\n\nВыберите раздел:',
            [
                [['text' => '🌐 Язык', 'action' => $this->getLink('language')]],
                [['text' => '🔔 Уведомления', 'action' => $this->getLink('notifications')]],
                [['text' => '◀️ Назад', 'action' => '/start']],
            ]
        );
    }

    protected function stepLanguage(?string $message, array $params): BotResponse
    {
        if (isset($params['lang'])) {
            // Сохранение языка...
            return BotResponse::finish('Язык изменён!');
        }

        return BotResponse::messageWithKeyboard(
            'Выберите язык:',
            [
                [
                    ['text' => '🇷🇺 Русский', 'action' => $this->getLink('language', ['lang' => 'ru'])],
                    ['text' => '🇬🇧 English', 'action' => $this->getLink('language', ['lang' => 'en'])],
                ],
                [['text' => '◀️ Назад', 'action' => $this->getLink('menu')]],
            ]
        );
    }
}
```

## Связанные документы

* [Conversations Overview](overview.md)
* [ConversationInterface](#)
* [BotResponse](../models/bot-response.md)

## Статус реализации

* [ ] Интерфейс ConversationInterface создан
* [ ] Класс AbstractConversation реализован
* [ ] Helper методы добавлены
* [ ] Unit тесты написаны
