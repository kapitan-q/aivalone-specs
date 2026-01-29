# 00023: Добавить ext-amqp в composer.json

## Категория
Конфигурация / Composer

## Приоритет
**Низкий** - Декларативная зависимость для документирования

## Описание проблемы

В Dockerfile расширение `ext-amqp` уже установлено:

```dockerfile
RUN pecl install amqp \
    && docker-php-ext-enable amqp
```

Однако в `composer.json` эта зависимость **не указана явно**:

```json
{
    "require": {
        "symfony/amqp-messenger": "^8.0"
        // Нет "ext-amqp": "*"
    }
}
```

Рекомендуется добавить явную зависимость от расширения для:
- Документирования требований проекта
- Проверки при `composer install` (предупреждение если расширение не установлено)
- Корректной работы локально (без Docker)

## Необходимые изменения

Добавить в `composer.json` секцию `require`:

```json
{
    "require": {
        "ext-amqp": "*"
    }
}
```

## Файлы для изменения

- `services/aivalone-backend/composer.json`

## Верификация

```bash
docker-compose exec php-fpm php -m | grep amqp
docker-compose exec php-fpm composer check-platform-reqs
```
