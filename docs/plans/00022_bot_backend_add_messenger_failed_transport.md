# 00022: Добавить failed транспорт для Messenger

## Категория
Конфигурация / Messenger

## Приоритет
**Средний** - Важно для отладки и повторной обработки неудачных сообщений

## Описание проблемы

В `config/packages/messenger.yaml` указан `failure_transport: failed`, но сам транспорт `failed` **не определён**:

```yaml
framework:
    messenger:
        failure_transport: failed  # Указан, но не определён!

        transports:
            async:
                dsn: '%env(MESSENGER_TRANSPORT_DSN)%'
                # ...
            python_bridge:
                # ...
            python_bridge_events:
                # ...
            # НЕТ транспорта 'failed'!
```

При неудачной обработке сообщения Symfony Messenger попытается отправить его в транспорт `failed`, который не существует, что приведёт к ошибке.

## Необходимые изменения

Добавить транспорт `failed` в `config/packages/messenger.yaml`:

```yaml
framework:
    messenger:
        failure_transport: failed

        transports:
            async:
                dsn: '%env(MESSENGER_TRANSPORT_DSN)%'
                options:
                    queue_name: messages

            # Транспорт для неудачных сообщений
            failed:
                dsn: '%env(MESSENGER_TRANSPORT_DSN)%'
                options:
                    queue_name: failed

            # Транспорт для команд в Python-bridge
            python_bridge:
                # ...

            # Транспорт для событий от Python-bridge
            python_bridge_events:
                # ...
```

## Альтернативный вариант (использовать Doctrine)

Для более надёжного хранения неудачных сообщений можно использовать Doctrine:

```yaml
failed:
    dsn: 'doctrine://default?queue_name=failed_messages'
```

Это требует добавления пакета `symfony/doctrine-messenger`:

```bash
composer require symfony/doctrine-messenger
```

## Рекомендация

**Рекомендуется использовать Redis** для простоты, так как он уже используется для основного транспорта.

## Полезные команды

После настройки можно использовать:

```bash
# Просмотр неудачных сообщений
php bin/console messenger:failed:show

# Повторная обработка неудачного сообщения
php bin/console messenger:failed:retry

# Удаление неудачного сообщения
php bin/console messenger:failed:remove {id}
```

## Файлы для изменения

- `services/aivalone-backend/config/packages/messenger.yaml`

## Верификация

После внесения изменений:
```bash
docker-compose exec php-fpm php bin/console debug:messenger
docker-compose exec php-fpm php bin/console messenger:failed:show
```
