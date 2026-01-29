# 00024: Дополнить Conversation Locator

## Категория
Конфигурация / Dependency Injection

## Приоритет
**Важный** - Без регистрации диалогов они недоступны через `ConversationFactory`

## Описание проблемы

В `services.yaml` в `conversation.locator` зарегистрированы не все диалоги:

```yaml
# services.yaml, строки 96-109
conversation.locator:
    class: Symfony\Component\DependencyInjection\ServiceLocator
    arguments: !service_locator
        start: '@App\Context\Bot\Application\Conversation\StartConversation'
        help: '@App\Context\Bot\Application\Conversation\HelpConversation'
        menu: '@App\Context\Bot\Application\Conversation\MenuConversation'
        setupfilter: '@App\Context\Bot\Application\Conversation\SetupFilterConversation'
        addgroup: '@App\Context\Bot\Application\Conversation\AddGroupConversation'
        # Отсутствуют: BillingConversation, AuthConversation
```

Согласно документации (`docs/backend/bot/conversations.md`), в проекте реализованы следующие диалоги:

1. ✅ `StartConversation` (/start) - зарегистрирован
2. ✅ `MenuConversation` (/menu) - зарегистрирован
3. ✅ `SetupFilterConversation` (/setupfilter) - зарегистрирован
4. ✅ `AddGroupConversation` (/addgroup) - зарегистрирован
5. ❌ **`AuthConversation`** (/auth) - **НЕ зарегистрирован**
6. ❌ **`BillingConversation`** (/billing) - **НЕ зарегистрирован**
7. ✅ `HelpConversation` (/help) - зарегистрирован

## Файлы диалогов

Проверка наличия файлов:
- `AuthConversation.php` - **существует** в `src/Context/Bot/Application/Conversation/`
- `BillingConversation.php` - **существует** в `src/Context/Bot/Application/Conversation/`

## Необходимые изменения

Обновить `services.yaml`:

```yaml
# Conversation Locator (для получения диалогов по коду)
conversation.locator:
    class: Symfony\Component\DependencyInjection\ServiceLocator
    arguments: !service_locator
        start: '@App\Context\Bot\Application\Conversation\StartConversation'
        help: '@App\Context\Bot\Application\Conversation\HelpConversation'
        menu: '@App\Context\Bot\Application\Conversation\MenuConversation'
        setupfilter: '@App\Context\Bot\Application\Conversation\SetupFilterConversation'
        addgroup: '@App\Context\Bot\Application\Conversation\AddGroupConversation'
        auth: '@App\Context\Bot\Application\Conversation\AuthConversation'        # Добавить
        billing: '@App\Context\Bot\Application\Conversation\BillingConversation'  # Добавить
```

## Альтернативный подход (автоматическая регистрация)

Для автоматической регистрации диалогов можно использовать tagged services:

1. Добавить в `services.yaml`:

```yaml
# Автоматическая регистрация всех Conversation
services:
    _instanceof:
        App\Context\Bot\Domain\Model\ConversationInterface:
            tags: ['bot.conversation']
```

2. Создать Compiler Pass или использовать `ServiceLocatorTagPass`.

Однако это требует дополнительной работы, поэтому **рекомендуется просто добавить недостающие диалоги вручную**.

## Файлы для изменения

- `services/aivalone-backend/config/services.yaml`

## Верификация

После внесения изменений:
```bash
docker-compose exec php-fpm php bin/console debug:container conversation.locator
docker-compose exec php-fpm php bin/console cache:clear
```

Или протестировать через бота:
- Отправить команду `/auth`
- Отправить команду `/billing`
