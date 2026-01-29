# Сводка: Исправления конфигурации aivalone-backend

## Обзор

Этот документ содержит сводку всех обнаруженных проблем с конфигурацией проекта `aivalone-backend`, которые препятствуют запуску через `docker-compose up`.

## Критические проблемы (блокирующие запуск)

| # | Задача | Файл | Описание |
|---|--------|------|----------|
| 00019 | [add_rabbitmq_docker](00019_bot_backend_add_rabbitmq_docker.md) | `docker-compose.yaml` | RabbitMQ отсутствует в docker-compose, хотя используется в messenger.yaml |
| 00020 | [add_snc_redis](00020_bot_backend_add_snc_redis.md) | `composer.json`, `bundles.php` | Отсутствует SncRedisBundle, который используется в services.yaml (`@snc_redis.default`) |
| 00021 | [fix_tariff_cache_di](00021_bot_backend_fix_tariff_cache_di.md) | `services.yaml` | `TariffPermissionsCache` не получает все необходимые аргументы |

## Важные проблемы

| # | Задача | Файл | Описание |
|---|--------|------|----------|
| 00018 | [create_env_example](00018_bot_backend_create_env_example.md) | — | Отсутствует `.env.example` для документирования конфигурации |
| 00022 | [add_messenger_failed](00022_bot_backend_add_messenger_failed_transport.md) | `messenger.yaml` | Указан `failure_transport: failed`, но транспорт `failed` не определён |
| 00024 | [fix_conversation_locator](00024_bot_backend_fix_conversation_locator.md) | `services.yaml` | `AuthConversation` и `BillingConversation` не зарегистрированы в локаторе |

## Низкоприоритетные улучшения

| # | Задача | Файл | Описание |
|---|--------|------|----------|
| 00023 | [add_ext_amqp](00023_bot_backend_add_ext_amqp.md) | `composer.json` | Явно указать зависимость `ext-amqp` |

## Рекомендуемый порядок исправлений

1. **00019** — Добавить RabbitMQ в docker-compose
2. **00020** — Установить SncRedisBundle
3. **00021** — Исправить DI для TariffPermissionsCache
4. **00022** — Добавить failed транспорт
5. **00024** — Дополнить conversation.locator
6. **00018** — Создать .env.example
7. **00023** — Добавить ext-amqp в composer.json

## Команды для проверки после исправлений

```bash
# Пересборка контейнеров
docker compose build

# Запуск
docker compose up -d

# Установка зависимостей
docker compose exec php-fpm composer install

# Очистка кэша
docker compose exec php-fpm php bin/console cache:clear

# Проверка контейнера
docker compose exec php-fpm php bin/console debug:container --parameter

# Проверка Messenger
docker compose exec php-fpm php bin/console debug:messenger

# Проверка маршрутов
docker compose exec php-fpm php bin/console debug:router
```

## Связанные документы

- [README.md](../../services/aivalone-backend/README.md) — Основная документация проекта
- [Bot Context Overview](../backend/bot/overview.md) — Архитектура Bot контекста
- [Webhooks](../backend/bot/webhooks.md) — Документация по вебхукам
