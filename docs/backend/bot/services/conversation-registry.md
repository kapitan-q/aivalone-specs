# ConversationRegistry Service Specification

## Назначение

`ConversationRegistry` — сервис для регистрации и получения доступных диалогов (Conversations). Реализует паттерн Service Locator для диалогов.

## Интерфейс

```php
interface ConversationRegistryInterface
{
    /**
     * Регистрирует диалог в реестре
     */
    public function register(ConversationInterface $conversation): void;

    /**
     * Получает диалог по коду
     * @throws ConversationNotFoundException
     */
    public function get(string $code): ConversationInterface;

    /**
     * Проверяет наличие диалога
     */
    public function has(string $code): bool;

    /**
     * Возвращает список всех зарегистрированных диалогов
     * @return array<string, ConversationInterface>
     */
    public function all(): array;

    /**
     * Возвращает диалоги, которые должны отображаться в меню
     * @return array<array{icon: string, name: string, link: string}>
     */
    public function getMenuItems(): array;
}
```

## Реализация

```php
final class ConversationRegistry implements ConversationRegistryInterface
{
    /** @var array<string, ConversationInterface> */
    private array $conversations = [];

    public function register(ConversationInterface $conversation): void
    {
        $code = $conversation->getCode();

        if (isset($this->conversations[$code])) {
            throw new \LogicException(
                sprintf('Conversation with code "%s" is already registered', $code)
            );
        }

        $this->conversations[$code] = $conversation;
    }

    public function get(string $code): ConversationInterface
    {
        if (!isset($this->conversations[$code])) {
            throw ConversationNotFoundException::withCode($code);
        }

        return $this->conversations[$code];
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

## Регистрация через Symfony

Использование Symfony tagged services для автоматической регистрации:

```yaml
# services.yaml
services:
    App\Context\Bot\Application\Service\ConversationRegistry:
        arguments:
            - !tagged_iterator bot.conversation

    App\Context\Bot\Domain\Conversation\:
        resource: '../src/Context/Bot/Domain/Conversation/*Conversation.php'
        tags: ['bot.conversation']
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

## Связанные документы

* [Services Overview](overview.md)
* [Router](router.md)
* [AbstractConversation](../conversations/abstract-conversation.md)

## Статус реализации

* [ ] Интерфейс ConversationRegistryInterface создан
* [ ] Класс ConversationRegistry реализован
* [ ] Symfony DI конфигурация настроена
* [ ] Tagged services работают
* [ ] Unit тесты написаны
