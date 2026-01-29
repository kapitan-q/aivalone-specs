# 00020: Добавить SncRedisBundle в зависимости

## Категория
Зависимости / Composer

## Приоритет
**Критический** - Без этого бандла не работает DI для Redis

## Описание проблемы

В `services.yaml` используется сервис `@snc_redis.default`:

```yaml
# services.yaml, строка 137
App\Context\Shared\Domain\Permission\TariffPermissionsCache:
    arguments:
        $redis: '@snc_redis.default'
        $fallbackPermissions: '%tariff_permissions.fallback%'
```

Однако в `composer.json` **отсутствует зависимость** `snc/redis-bundle`:

```json
{
    "require": {
        // ... нет snc/redis-bundle
    }
}
```

Также в `config/bundles.php` **отсутствует регистрация бандла**:

```php
return [
    Symfony\Bundle\FrameworkBundle\FrameworkBundle::class => ['all' => true],
    Doctrine\Bundle\DoctrineBundle\DoctrineBundle::class => ['all' => true],
    Doctrine\Bundle\MigrationsBundle\DoctrineMigrationsBundle::class => ['all' => true],
    Symfony\Bundle\MakerBundle\MakerBundle::class => ['dev' => true],
    // Нет Snc\RedisBundle\SncRedisBundle
];
```

## Необходимые действия

### 1. Установить пакет

```bash
composer require snc/redis-bundle predis/predis
```

Или добавить в `composer.json`:

```json
{
    "require": {
        "snc/redis-bundle": "^4.0",
        "predis/predis": "^2.0"
    }
}
```

### 2. Зарегистрировать бандл

Добавить в `config/bundles.php`:

```php
<?php

return [
    Symfony\Bundle\FrameworkBundle\FrameworkBundle::class => ['all' => true],
    Doctrine\Bundle\DoctrineBundle\DoctrineBundle::class => ['all' => true],
    Doctrine\Bundle\MigrationsBundle\DoctrineMigrationsBundle::class => ['all' => true],
    Symfony\Bundle\MakerBundle\MakerBundle::class => ['dev' => true],
    Snc\RedisBundle\SncRedisBundle::class => ['all' => true],  // Добавить
];
```

### 3. Создать конфигурацию

Создать файл `config/packages/snc_redis.yaml`:

```yaml
snc_redis:
    clients:
        default:
            type: predis
            alias: default
            dsn: '%env(REDIS_URL)%'
            logging: '%kernel.debug%'
```

## Альтернативный подход

Если SncRedisBundle не требуется, можно использовать прямую инъекцию Redis через factory. Но это потребует рефакторинга `TariffPermissionsCache` и всех связанных сервисов.

### Альтернативная конфигурация (без SncRedisBundle)

1. Создать factory для Redis:

```php
// src/Context/Shared/Infrastructure/Service/RedisFactory.php
namespace App\Context\Shared\Infrastructure\Service;

class RedisFactory
{
    public function __construct(private string $redisUrl) {}

    public function create(): \Redis
    {
        $redis = new \Redis();
        $urlParts = parse_url($this->redisUrl);
        $redis->connect($urlParts['host'], $urlParts['port'] ?? 6379);
        return $redis;
    }
}
```

2. Обновить `services.yaml`:

```yaml
App\Context\Shared\Infrastructure\Service\RedisFactory:
    arguments:
        $redisUrl: '%env(REDIS_URL)%'

redis.client:
    class: \Redis
    factory: ['@App\Context\Shared\Infrastructure\Service\RedisFactory', 'create']

App\Context\Shared\Domain\Permission\TariffPermissionsCache:
    arguments:
        $redis: '@redis.client'
        $fallbackPermissions: '%tariff_permissions.fallback%'
```

## Рекомендация

**Рекомендуется использовать SncRedisBundle**, так как:
- Стандартный подход для Symfony-проектов
- Поддержка логирования и профилирования
- Интеграция с Symfony Profiler
- Проверенное решение с хорошей документацией

## Файлы для создания/изменения

- `services/aivalone-backend/composer.json` - добавить зависимости
- `services/aivalone-backend/config/bundles.php` - зарегистрировать бандл
- `services/aivalone-backend/config/packages/snc_redis.yaml` - **создать**

## Верификация

После внесения изменений:
```bash
docker-compose exec php-fpm composer install
docker-compose exec php-fpm php bin/console debug:container snc_redis
docker-compose exec php-fpm php bin/console cache:clear
```
