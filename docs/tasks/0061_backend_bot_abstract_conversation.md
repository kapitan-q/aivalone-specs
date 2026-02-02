# Задача 0061: AbstractConversation Base Class

## Контекст

AbstractConversation — базовый класс для всех диалогов. Реализует паттерн FSM (Finite State Machine) для управления шагами диалога.

## Цель

Создать абстрактный класс `AbstractConversation` для построения диалогов.

## Спецификация

- [AbstractConversation](../backend/bot/conversations/abstract-conversation.md)

## Файлы для создания

```
src/Context/Bot/Domain/Conversation/AbstractConversation.php
src/Context/Bot/Domain/Conversation/ConversationInterface.php
tests/Unit/Context/Bot/Domain/Conversation/AbstractConversationTest.php
```

## Важно

1. **Роутер управляет state** — диалог НЕ создаёт и НЕ сохраняет ConversationState
2. **Диалог получает contextData** — роутер внедряет данные через setContextData()
3. **Диалог возвращает изменённые данные** — роутер забирает через getContextData() и сохраняет
4. **code != command** — код диалога может отличаться от команды (getCommand() формирует команду)

## Требования

### ConversationInterface

```php
<?php

declare(strict_types=1);

namespace App\Context\Bot\Domain\Conversation;

use App\Context\Bot\Domain\Model\BotResponse;

interface ConversationInterface
{
    /**
     * Уникальный код диалога.
     */
    public function getCode(): string;

    /**
     * Команда для вызова диалога (по умолчанию /{code}).
     * Может отличаться от кода.
     */
    public function getCommand(): string;

    /**
     * Начальный шаг диалога.
     */
    public function getInitialStep(): string;

    /**
     * Обрабатывает шаг диалога.
     *
     * @param array<string, string> $params Параметры из команды
     */
    public function handle(
        ?string $stepCode,
        ?string $message,
        ?array $params,
    ): BotResponse;

    /**
     * Установить данные контекста (вызывается роутером перед handle).
     *
     * @param array<string, mixed> $data
     */
    public function setContextData(array $data): void;

    /**
     * Получить данные контекста (вызывается роутером после handle).
     *
     * @return array<string, mixed>
     */
    public function getContextData(): array;

    /**
     * Получить текущий шаг (после handle).
     */
    public function getCurrentStep(): string;

    /**
     * Информация для меню (если диалог должен отображаться в меню).
     *
     * @return array{icon: string, name: string, link: string}|null
     */
    public function getMenuInfo(): ?array;
}
```

### AbstractConversation

```php
<?php

declare(strict_types=1);

namespace App\Context\Bot\Domain\Conversation;

use App\Context\Bot\Domain\Model\BotResponse;

/**
 * Базовый класс для всех диалогов.
 *
 * Роутер управляет состоянием (создаёт, сохраняет, удаляет ConversationState).
 * Диалог только обрабатывает текущий шаг и может менять contextData и currentStep.
 */
abstract class AbstractConversation implements ConversationInterface
{
    protected string $currentStep;

    /** @var array<string, mixed> */
    protected array $contextData = [];

    /**
     * Код диалога (должен быть уникальным).
     */
    abstract public function getCode(): string;

    /**
     * Начальный шаг диалога.
     */
    abstract protected function getInitialStep(): string;

    /**
     * Команда для вызова. По умолчанию /{code}.
     */
    public function getCommand(): string
    {
        return '/' . $this->getCode();
    }

    public function handle(?string $stepCode, ?string $message, ?array $params): BotResponse
    {
        $step = $stepCode ?? $this->getInitialStep();
        $this->currentStep = $step;

        $method = 'step' . ucfirst($this->toCamelCase($step));

        if (method_exists($this, $method)) {
            return $this->$method($message, $params ?? []);
        }

        return $this->stepCancel();
    }

    /**
     * Переход к следующему шагу и его выполнение.
     */
    public function next(string $nextStepCode): BotResponse
    {
        $this->currentStep = $nextStepCode;

        return $this->handle($nextStepCode, null, null);
    }

    /**
     * Шаг отмены — сброс и возврат к начальному шагу.
     */
    protected function stepCancel(): BotResponse
    {
        $this->contextData = [];

        return $this->next($this->getInitialStep());
    }

    /**
     * Отмена диалога.
     */
    public function cancel(): BotResponse
    {
        return $this->stepCancel();
    }

    public function getMenuInfo(): ?array
    {
        return null;
    }

    /**
     * Генерация ссылки на шаг этого диалога.
     *
     * @param array<string, string> $params
     */
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

    /**
     * @return array<string, mixed>
     */
    public function getContextData(): array
    {
        return $this->contextData;
    }

    /**
     * @param array<string, mixed> $data
     */
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

    private function toCamelCase(string $string): string
    {
        return str_replace(' ', '', ucwords(str_replace(['_', '-'], ' ', $string)));
    }
}
```

## Паттерн использования

Каждый шаг диалога — это метод `step{StepName}`:

```php
final class SettingsConversation extends AbstractConversation
{
    public function getCode(): string
    {
        return 'settings';
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
            // Сохраняем выбор в контекст (роутер сохранит в state)
            $this->setContextParam('selectedLang', $params['lang']);

            return BotResponse::finish('Язык изменён на ' . $params['lang']);
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

## Взаимодействие с Router

```
1. Router получает BotRequest
2. Router находит диалог через Registry (по command)
3. Router загружает ConversationState (если есть)
4. Router вызывает conversation.setContextData(state.contextData)
5. Router вызывает conversation.handle(step, message, params)
6. Router получает BotResponse
7. Если response.isFinished():
   - Router удаляет state
8. Иначе:
   - Router обновляет state:
     - state.stepCode = conversation.getCurrentStep()
     - state.contextData = conversation.getContextData()
   - Router сохраняет state
9. Router отправляет BotResponse через MessageSender
```

## Тесты

- [x] getCode() возвращает уникальный код
- [x] getCommand() по умолчанию возвращает /{code}
- [x] getInitialStep() возвращает начальный шаг
- [x] handle() вызывает правильный метод step{Name}
- [x] handle() вызывает stepCancel() для несуществующего шага
- [x] next() переходит к указанному шагу
- [x] setContextData() / getContextData() работают
- [x] getLink() формирует ссылку с params
- [x] getCurrentStep() возвращает текущий шаг после handle()

## Зависимости

- `BotResponse` (задача 0054)

## Definition of Done

- [x] ConversationInterface создан
- [x] AbstractConversation создан
- [x] Unit-тесты написаны и проходят
- [x] Код соответствует PSR-12
- [x] PHPStan level 9 без ошибок
