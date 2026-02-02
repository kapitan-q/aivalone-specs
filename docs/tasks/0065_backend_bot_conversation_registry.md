# Задача 0065: ConversationRegistry Service

## Контекст

ConversationRegistry — реестр всех доступных диалогов. Управляет регистрацией и поиском Conversation по коду и команде.

## Цель

Создать сервис `ConversationRegistry` для управления диалогами.

## Спецификация

- [ConversationRegistry](../backend/bot/services/conversation-registry.md)

## Файлы для создания

```
src/Context/Bot/Application/Service/ConversationRegistry.php
src/Context/Bot/Application/Service/ConversationRegistryInterface.php
tests/Unit/Context/Bot/Application/Service/ConversationRegistryTest.php
```

## Важно

**code != command** — код диалога может отличаться от команды вызова.
- `getCode()` — уникальный идентификатор диалога
- `getCommand()` — команда для вызова (по умолчанию `/{code}`)

Поэтому нужны два метода поиска:
- `findByCode()` — для продолжения диалога из state
- `findByCommand()` — для поиска по команде из запроса

## Требования

### ConversationRegistryInterface

```php
<?php

declare(strict_types=1);

namespace App\Context\Bot\Application\Service;

use App\Context\Bot\Domain\Conversation\ConversationInterface;

interface ConversationRegistryInterface
{
    /**
     * Регистрирует диалог в реестре.
     */
    public function register(ConversationInterface $conversation): void;

    /**
     * Находит диалог по коду.
     */
    public function get(string $code): ConversationInterface;

    /**
     * Находит диалог по коду или null.
     */
    public function findByCode(string $code): ?ConversationInterface;

    /**
     * Находит диалог по команде (например, /start).
     */
    public function findByCommand(string $command): ?ConversationInterface;

    /**
     * Проверяет наличие диалога.
     */
    public function has(string $code): bool;

    /**
     * Возвращает все зарегистрированные диалоги.
     *
     * @return array<string, ConversationInterface>
     */
    public function all(): array;

    /**
     * Возвращает диалоги для отображения в меню.
     *
     * @return array<array{icon: string, name: string, link: string}>
     */
    public function getMenuItems(): array;
}
```

### ConversationRegistry

```php
<?php

declare(strict_types=1);

namespace App\Context\Bot\Application\Service;

use App\Context\Bot\Domain\Conversation\ConversationInterface;
use App\Context\Bot\Domain\Exception\ConversationNotFoundException;

final class ConversationRegistry implements ConversationRegistryInterface
{
    /**
     * @var array<string, ConversationInterface>
     */
    private array $conversations = [];

    /**
     * Индекс command → code для быстрого поиска по команде.
     *
     * @var array<string, string>
     */
    private array $commandIndex = [];

    /**
     * @param iterable<ConversationInterface> $conversations
     */
    public function __construct(iterable $conversations = [])
    {
        foreach ($conversations as $conversation) {
            $this->register($conversation);
        }
    }

    public function register(ConversationInterface $conversation): void
    {
        $code = $conversation->getCode();

        if (isset($this->conversations[$code])) {
            throw new \LogicException(
                sprintf('Conversation with code "%s" is already registered', $code),
            );
        }

        $this->conversations[$code] = $conversation;
        $this->commandIndex[$conversation->getCommand()] = $code;
    }

    public function get(string $code): ConversationInterface
    {
        if (!isset($this->conversations[$code])) {
            throw ConversationNotFoundException::withCode($code);
        }

        return $this->conversations[$code];
    }

    public function findByCode(string $code): ?ConversationInterface
    {
        return $this->conversations[$code] ?? null;
    }

    public function findByCommand(string $command): ?ConversationInterface
    {
        $code = $this->commandIndex[$command] ?? null;

        if ($code === null) {
            return null;
        }

        return $this->conversations[$code] ?? null;
    }

    public function has(string $code): bool
    {
        return isset($this->conversations[$code]);
    }

    public function all(): array
    {
        return $this->conversations;
    }

    public function getMenuItems(): array
    {
        $items = [];

        foreach ($this->conversations as $conversation) {
            $menuInfo = $conversation->getMenuInfo();

            if ($menuInfo !== null) {
                $items[] = $menuInfo;
            }
        }

        return $items;
    }
}
```

## Symfony Integration

Для автоматической регистрации диалогов используем autoconfigure:

```yaml
# config/services.yaml
services:
    _instanceof:
        App\Context\Bot\Domain\Conversation\ConversationInterface:
            tags: ['bot.conversation']

    App\Context\Bot\Application\Service\ConversationRegistry:
        arguments:
            $conversations: !tagged_iterator bot.conversation
```

Или явная регистрация через CompilerPass:

```php
final class ConversationRegistryCompilerPass implements CompilerPassInterface
{
    public function process(ContainerBuilder $container): void
    {
        $registry = $container->getDefinition(ConversationRegistry::class);

        foreach ($container->findTaggedServiceIds('bot.conversation') as $id => $tags) {
            $registry->addMethodCall('register', [new Reference($id)]);
        }
    }
}
```

## Тесты

- [x] register() добавляет диалог в реестр
- [x] register() индексирует команду
- [x] register() выбрасывает исключение при дублировании кода
- [x] findByCode() возвращает диалог или null
- [x] findByCommand() возвращает диалог по команде
- [x] findByCommand() возвращает null для неизвестной команды
- [x] get() возвращает диалог или выбрасывает исключение
- [x] all() возвращает все диалоги
- [x] has() корректно проверяет наличие
- [x] getMenuItems() возвращает только диалоги с menuInfo
- [x] Конструктор регистрирует диалоги из iterable

## Зависимости

- `ConversationInterface` (задача 0061)
- `ConversationNotFoundException` (задача 0057)

## Definition of Done

- [x] ConversationRegistryInterface создан
- [x] Класс ConversationRegistry создан
- [x] Индексация по команде реализована
- [ ] Symfony конфигурация добавлена
- [x] Unit-тесты написаны и проходят
- [x] Код соответствует PSR-12
- [x] PHPStan level 9 без ошибок
