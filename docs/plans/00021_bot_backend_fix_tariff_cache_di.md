# 00021: Исправить DI для TariffPermissionsCache

## Категория
Конфигурация / Dependency Injection

## Приоритет
**Критический** - Без этого не работает проверка прав пользователей

## Описание проблемы

В `services.yaml` конфигурация для `TariffPermissionsCache` **неполная**:

```yaml
# services.yaml, строки 134-138
App\Context\Shared\Domain\Permission\TariffPermissionsCache:
    arguments:
        $redis: '@snc_redis.default'
        $fallbackPermissions: '%tariff_permissions.fallback%'
```

Однако конструктор `TariffPermissionsCache` требует **4 аргумента**:

```php
// TariffPermissionsCache.php, строки 25-31
public function __construct(
    private \Redis|\Predis\Client $redis,
    private BillingPortInterface $billingPort,      // НЕ УКАЗАН в services.yaml!
    private array $fallbackPermissions = [],
    private ?LoggerInterface $logger = null
) {
}
```

**Проблема:** Аргумент `$billingPort` не указан в конфигурации, и Symfony не может автоматически разрешить его, так как привязка `BillingPortInterface` находится в другом месте `services.yaml`.

## Необходимые изменения

Обновить конфигурацию в `services.yaml`:

```yaml
# TariffPermissionsCache с Redis и fallback
App\Context\Shared\Domain\Permission\TariffPermissionsCache:
    arguments:
        $redis: '@snc_redis.default'
        $billingPort: '@App\Context\Shared\Application\Port\BillingPortInterface'
        $fallbackPermissions: '%tariff_permissions.fallback%'
        $logger: '@logger'
```

## Или использовать autowire

Если autowire включен (что так и есть по умолчанию), достаточно добавить alias для `BillingPortInterface`:

```yaml
# Shared -> Billing port (для получения прав тарифов)
App\Context\Shared\Application\Port\BillingPortInterface:
    alias: App\Context\Shared\Infrastructure\Port\BillingPort
```

**Этот alias уже есть в services.yaml (строки 131-132)**, поэтому проблема может быть в порядке определений или в том, что Symfony требует явного указания аргумента.

## Рекомендуемое решение

Явно указать все аргументы для надёжности:

```yaml
# TariffPermissionsCache с Redis и fallback
App\Context\Shared\Domain\Permission\TariffPermissionsCache:
    arguments:
        $redis: '@snc_redis.default'
        $billingPort: '@App\Context\Shared\Infrastructure\Port\BillingPort'
        $fallbackPermissions: '%tariff_permissions.fallback%'
        # $logger будет автоматически внедрён через autowire
```

## Связанные задачи

- [00020_bot_backend_add_snc_redis.md](00020_bot_backend_add_snc_redis.md) - Установка SncRedisBundle (для `@snc_redis.default`)

## Файлы для изменения

- `services/aivalone-backend/config/services.yaml`

## Верификация

После внесения изменений:
```bash
docker-compose exec php-fpm php bin/console debug:autowiring TariffPermissionsCache
docker-compose exec php-fpm php bin/console debug:container App\\Context\\Shared\\Domain\\Permission\\TariffPermissionsCache
docker-compose exec php-fpm php bin/console cache:clear
```
