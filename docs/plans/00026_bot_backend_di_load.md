# План корректировки DI загрузки классов

При запуске `cache:clear` возникает `TypeError`, так как Symfony 8 не принимает `ServiceLocatorArgument` (возвращаемый тегом `!service_locator`) напрямую в качестве значения `arguments`. Значение `arguments` всегда должно быть массивом.

## Предлагаемые изменения

### Конфигурация

#### [MODIFY] [services.yaml](file:///home/titan/workspace/aivalone/aivalone-specs/services/aivalone-backend/config/services.yaml)

Необходимо обернуть вызовы `!service_locator` в массив. Это затронет следующие сервисы:

1.  `webhook.adapters.locator`
2.  `bot.adapters.locator`
3.  `conversation.locator`

Пример исправления:
```yaml
    webhook.adapters.locator:
        class: Symfony\Component\DependencyInjection\ServiceLocator
        arguments:
            - !service_locator
                telegram: '@App\Context\Bot\Infrastructure\Messenger\TelegramWebhookAdapter'
```

## План проверки

### Автоматизированная проверка
- Выполнить очистку кэша:
  `docker-compose exec php-fpm php bin/console cache:clear`
- Проверить отсутствие ошибок `TypeError`.

### Ручная проверка
- Убедиться, что приложение запускается и корректно инициализирует сервисы, использующие эти локаторы.
