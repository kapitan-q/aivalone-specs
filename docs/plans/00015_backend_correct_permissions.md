# План рефакторинга системы прав пользователей

## Анализ предложенного решения

### Текущая архитектура (проблемы)

```
Account.registerUser() → UserRegisteredEvent
    ↓ (async)
Billing.UserRegisteredHandler → activateSubscription('FREE')
    ↓
PermissionAggregator.aggregateAndPublish()
    ↓
UserPermissionsUpdatedEvent
    ↓ (async)
Account.UserPermissionsUpdatedHandler → User.updatePermissions()
```

**Проблемы:**
1. Race condition: между созданием User и получением прав есть временное окно
2. Сложный event-driven flow через 4 шага и 2 события
3. Дублирование: права хранятся в Billing (TariffOption) и в Account (User.permissions)

### Замечания к предложенному решению

| Аспект | Замечание | Рекомендация |
|--------|-----------|--------------|
| **Множественные подписки** | Текущая система поддерживает несколько активных подписок (Premium+Base) с приоритетами | Хранить `effectiveTariffCodes[]` - массив тарифов, отсортированный по priority |
| **Истечение подписок** | Нужен механизм пересчета тарифов при истечении | Событие `UserEffectiveTariffsChangedEvent` от Billing при любом изменении подписок |
| **Кэш прав** | При изменении опций тарифа нужна инвалидация | `TariffPermissionsUpdatedEvent` для обновления Redis кэша |
| **Fallback** | Если Billing/Redis недоступен | Hardcoded fallback в конфиге |
| **Shared сервис** | Хорошая идея, но не в User | `AuthService` в Shared контексте - единая точка проверки прав |

### Улучшенная архитектура

**Ключевые решения:**
1. `effectiveTariffCodes` - массив доступных тарифов `['PREMIUM', 'BASE', 'FREE']` (отсортирован по priority)
2. Метод `can()` убирается из User → выносится в `AuthService`
3. `EffectiveTariffResolver` возвращает отсортированный список тарифов

```
РЕГИСТРАЦИЯ (синхронно):
Account.registerUser()
    → User создается с effectiveTariffCodes = ['FREE']
    → ГОТОВО (без событий)

ИЗМЕНЕНИЕ ПОДПИСКИ:
Billing.SubscriptionService.activateSubscription()
    → EffectiveTariffResolver.resolve() → ['PREMIUM', 'BASE', 'FREE']
    → UserEffectiveTariffsChangedEvent
        → Account обновляет effectiveTariffCodes

ПРОВЕРКА ПРАВ:
Bot.authService->allowed($user, 'CAN_USE_AI', true)
    → AuthService проверяет по первому тарифу из effectiveTariffCodes
    → TariffPermissionsCache.getPermissions('PREMIUM')
    → FeatureRegistry проверяет значение
```

---

## План изменений по контекстам

### 1. Account контекст

**Изменяемые файлы:**

| Файл | Изменение |
|------|-----------|
| [User.php](services/aivalone-backend/src/Context/Account/Domain/Model/User.php) | Заменить `$permissions` на `$effectiveTariffCodes` (массив), удалить `PermissionsTrait` |
| [UserOrmEntity.php](services/aivalone-backend/src/Context/Account/Infrastructure/Entity/UserOrmEntity.php) | Заменить поле `permissions` на `effective_tariff_codes` (JSON массив) |
| [UserService.php](services/aivalone-backend/src/Context/Account/Application/Service/UserService.php) | При `registerUser()` сразу устанавливать `effectiveTariffCodes = ['FREE']`, убрать событие |
| UserPermissionsUpdatedHandler.php | **УДАЛИТЬ** |

**Новые файлы:**
- `Application/EventHandler/UserEffectiveTariffsChangedHandler.php` - обновляет `effectiveTariffCodes`

**Изменение User.php:**
```php
class User
{
    public function __construct(
        private UserId $id,
        private int $telegramId,
        private array $effectiveTariffCodes = ['FREE'] // Массив тарифов, отсортированный по priority
    ) {}

    /**
     * Возвращает массив кодов тарифов, отсортированный по приоритету (высший первым)
     * @return string[] например ['PREMIUM', 'BASE', 'FREE']
     */
    public function getEffectiveTariffCodes(): array
    {
        return $this->effectiveTariffCodes;
    }

    /**
     * Возвращает код тарифа с наивысшим приоритетом
     */
    public function getTopTariffCode(): string
    {
        return $this->effectiveTariffCodes[0] ?? 'FREE';
    }

    public function changeEffectiveTariffCodes(array $codes): void
    {
        $this->effectiveTariffCodes = $codes;
    }
}
```

### 2. Billing контекст

**Изменяемые файлы:**

| Файл | Изменение |
|------|-----------|
| [PermissionAggregator.php](services/aivalone-backend/src/Context/Billing/Application/Service/PermissionAggregator.php) | Переименовать в `EffectiveTariffResolver`, возвращать код тарифа |
| [SubscriptionService.php](services/aivalone-backend/src/Context/Billing/Application/Service/SubscriptionService.php) | Публиковать `UserEffectiveTariffChangedEvent` |
| UserRegisteredHandler.php | **УДАЛИТЬ** |
| UserPermissionsUpdatedEvent.php | **УДАЛИТЬ** |

**Новые файлы:**
- `Application/Service/TariffPermissionsService.php` - API для получения прав тарифов
- `Application/Event/UserEffectiveTariffChangedEvent.php`
- `Application/Event/TariffPermissionsUpdatedEvent.php`

**Новый EffectiveTariffResolver:**
```php
class EffectiveTariffResolver
{
    /**
     * Возвращает массив кодов тарифов пользователя, отсортированный по приоритету (высший первым)
     * Всегда включает FREE как базовый тариф
     *
     * @return string[] например ['PREMIUM', 'BASE', 'FREE']
     */
    public function resolve(UserId $userId): array
    {
        $subscriptions = $this->subscriptionRepository->findActiveByUserId($userId);

        if (empty($subscriptions)) {
            return ['FREE'];
        }

        // Собираем тарифы с приоритетами
        $tariffs = [];
        foreach ($subscriptions as $subscription) {
            $tariff = $this->tariffPlanRepository->findById($subscription->getTariffPlanId());
            if ($tariff) {
                $tariffs[] = [
                    'code' => $tariff->getCode(),
                    'priority' => $tariff->getPriority(),
                ];
            }
        }

        // Добавляем FREE если его нет
        $hasFree = array_filter($tariffs, fn($t) => $t['code'] === 'FREE');
        if (empty($hasFree)) {
            $tariffs[] = ['code' => 'FREE', 'priority' => 0];
        }

        // Сортируем по приоритету (высший первым)
        usort($tariffs, fn($a, $b) => $b['priority'] <=> $a['priority']);

        return array_column($tariffs, 'code');
    }
}
```

### 3. Shared контекст

**Новые файлы:**

| Файл | Назначение |
|------|------------|
| `Application/Service/AuthService.php` | **Главный сервис проверки прав** |
| `Domain/Permission/TariffPermissionsCache.php` | Кэш прав тарифов (Redis) |
| `Application/Port/BillingPortInterface.php` | Интерфейс порта к Billing |
| `Infrastructure/Port/BillingPort.php` | Реализация порта |
| `Application/EventHandler/TariffPermissionsUpdatedHandler.php` | Обновление кэша |

**Удаляемые файлы:**
| Файл | Причина |
|------|---------|
| `Domain/Permission/PermissionsTrait.php` | Логика переносится в `AuthService` |

**AuthService (главный сервис проверки прав):**
```php
namespace App\Context\Shared\Application\Service;

/**
 * Сервис авторизации - единая точка проверки прав пользователя
 */
class AuthService
{
    public function __construct(
        private TariffPermissionsCache $cache,
        private FeatureRegistry $registry
    ) {}

    /**
     * Проверяет, разрешено ли действие для пользователя
     *
     * @param UserInterface $user Пользователь с методом getTopTariffCode()
     * @param string $permissionCode Код права (MAX_GROUP, CAN_USE_AI, etc.)
     * @param mixed $value Значение для проверки
     */
    public function allowed(UserInterface $user, string $permissionCode, mixed $value): bool
    {
        $tariffCode = $user->getTopTariffCode();
        $permissions = $this->cache->getPermissions($tariffCode);

        if (!isset($permissions[$permissionCode])) {
            return false;
        }

        $option = $this->registry->get($permissionCode);
        return $option?->can($value, $permissions[$permissionCode]) ?? false;
    }

    /**
     * Получить значение права для пользователя
     */
    public function getPermissionValue(UserInterface $user, string $permissionCode): mixed
    {
        $tariffCode = $user->getTopTariffCode();
        $permissions = $this->cache->getPermissions($tariffCode);
        return $permissions[$permissionCode] ?? null;
    }
}
```

**UserInterface (для типизации):**
```php
namespace App\Context\Shared\Domain\Model;

interface UserInterface
{
    public function getId(): UserId;
    public function getTopTariffCode(): string;
    public function getEffectiveTariffCodes(): array;
}
```

**TariffPermissionsCache (Redis):**
```php
class TariffPermissionsCache
{
    private const CACHE_KEY = 'tariff_permissions';
    private const CACHE_TTL = 3600; // 1 час

    public function __construct(
        private \Redis $redis,
        private BillingPortInterface $billingPort,
        private array $fallbackPermissions = []
    ) {}

    public function getPermissions(string $tariffCode): array
    {
        $cached = $this->redis->hGet(self::CACHE_KEY, $tariffCode);

        if ($cached !== false) {
            return json_decode($cached, true);
        }

        // Cache miss - загрузить из Billing
        $this->warmUp();
        $cached = $this->redis->hGet(self::CACHE_KEY, $tariffCode);

        return $cached !== false
            ? json_decode($cached, true)
            : $this->fallbackPermissions[$tariffCode] ?? [];
    }

    public function warmUp(): void
    {
        try {
            $permissions = $this->billingPort->getAllTariffPermissions();
            foreach ($permissions as $tariffCode => $perms) {
                $this->redis->hSet(self::CACHE_KEY, $tariffCode, json_encode($perms));
            }
            $this->redis->expire(self::CACHE_KEY, self::CACHE_TTL);
        } catch (\Exception $e) {
            // Fallback в конфиг
            foreach ($this->fallbackPermissions as $tariffCode => $perms) {
                $this->redis->hSet(self::CACHE_KEY, $tariffCode, json_encode($perms));
            }
        }
    }

    public function invalidate(): void
    {
        $this->redis->del(self::CACHE_KEY);
    }
}
```

**PermissionsTrait - УДАЛЯЕТСЯ**

Логика проверки прав переносится в `AuthService`. Вместо:
```php
// БЫЛО:
$user->can('CAN_USE_AI', true, $featureRegistry)

// СТАЛО:
$authService->allowed($user, 'CAN_USE_AI', true)
```

### 4. Bot контекст

**Изменяемые файлы:**

| Файл | Изменение |
|------|-----------|
| [BotUser.php](services/aivalone-backend/src/Context/Bot/Domain/Model/BotUser.php) | Заменить `$permissions` на `$effectiveTariffCodes`, имплементировать `UserInterface`, удалить `PermissionsTrait` |
| [AccountPort.php](services/aivalone-backend/src/Context/Bot/Infrastructure/Account/AccountPort.php) | Обновить маппинг |
| [SetupFilterConversation.php](services/aivalone-backend/src/Context/Bot/Application/Conversation/SetupFilterConversation.php) | Инжектировать `AuthService`, заменить `$user->can()` на `$authService->allowed()` |
| [BotRouter.php](services/aivalone-backend/src/Context/Bot/Application/Service/BotRouter.php) | Инжектировать `AuthService`, заменить вызовы `can()` |
| [AddGroupConversation.php](services/aivalone-backend/src/Context/Bot/Application/Conversation/AddGroupConversation.php) | Обновить проверку `MAX_GROUP` |

**Изменение BotUser.php:**
```php
class BotUser implements UserInterface
{
    // Убираем: use PermissionsTrait;

    public function __construct(
        private readonly UserId $id,
        private readonly int $telegramId,
        private array $effectiveTariffCodes = ['FREE']
    ) {}

    public function getTopTariffCode(): string
    {
        return $this->effectiveTariffCodes[0] ?? 'FREE';
    }

    public function getEffectiveTariffCodes(): array
    {
        return $this->effectiveTariffCodes;
    }
}
```

**Пример изменения в SetupFilterConversation:**
```php
// БЫЛО:
if ($user->can('CAN_USE_MORPHOLOGY', 0, $this->featureRegistry)) {

// СТАЛО:
if ($this->authService->allowed($user, 'CAN_USE_MORPHOLOGY', true)) {
```

### 5. Monitoring контекст

**Изменяемые файлы:**

| Файл | Изменение |
|------|-----------|
| [AccountPort.php](services/aivalone-backend/src/Context/Monitoring/Infrastructure/Account/AccountPort.php) | Инжектировать `AuthService`, использовать `$authService->allowed()` |

**Пример изменения:**
```php
// БЫЛО:
return match ($strategyEnum) {
    FilterStrategy::Morpho => $user->can('CAN_USE_MORPHOLOGY', true, $this->featureRegistry),
    FilterStrategy::Ai => $user->can('CAN_USE_AI', true, $this->featureRegistry),
};

// СТАЛО:
return match ($strategyEnum) {
    FilterStrategy::Morpho => $this->authService->allowed($user, 'CAN_USE_MORPHOLOGY', true),
    FilterStrategy::Ai => $this->authService->allowed($user, 'CAN_USE_AI', true),
};
```

---

## Миграция данных

> **Примечание:** Production отсутствует, миграция существующих данных не требуется.
> Достаточно создать новую структуру таблицы с `effective_tariff_codes` вместо `permissions`.

### Миграция БД

```php
public function up(Schema $schema): void
{
    // Удалить старую колонку permissions
    $this->addSql('ALTER TABLE users DROP COLUMN IF EXISTS permissions');

    // Добавить новую колонку (JSON массив тарифов)
    $this->addSql("ALTER TABLE users ADD effective_tariff_codes JSON NOT NULL DEFAULT '[\"FREE\"]'");
}
```

---

## Конфигурация fallback

```yaml
# config/packages/tariff_permissions.yaml
tariff_permissions:
    fallback:
        FREE:
            MAX_GROUP: 5
            CAN_USE_PRIVATE_GROUPS: false
            CAN_USE_AI: false
            CAN_USE_MORPHOLOGY: false
        BASE:
            MAX_GROUP: 999999
            CAN_USE_PRIVATE_GROUPS: true
            CAN_USE_AI: false
            CAN_USE_MORPHOLOGY: true
        PREMIUM:
            MAX_GROUP: 999999
            CAN_USE_PRIVATE_GROUPS: true
            CAN_USE_AI: true
            CAN_USE_MORPHOLOGY: true
```

---

## Этапы реализации

### Этап 1: Инфраструктура Shared
1. Создать `UserInterface` (интерфейс для User с `getTopTariffCode()`)
2. Создать `TariffPermissionsCache` (Redis)
3. Создать `AuthService` (главный сервис проверки прав)
4. Создать `BillingPortInterface` и реализацию
5. Добавить конфиг fallback permissions
6. **Удалить** `PermissionsTrait`

### Этап 2: Billing контекст
1. Создать `TariffPermissionsService` (API для кэша)
2. Переименовать `PermissionAggregator` → `EffectiveTariffResolver` (возвращает массив тарифов)
3. Создать `UserEffectiveTariffsChangedEvent`
4. Создать `TariffPermissionsUpdatedEvent`
5. Обновить `SubscriptionService` для публикации новых событий
6. **Удалить** `UserRegisteredHandler`
7. **Удалить** `UserPermissionsUpdatedEvent`

### Этап 3: Account контекст
1. Заменить `permissions` на `effectiveTariffCodes` (JSON массив) в `User`
2. Имплементировать `UserInterface` в `User`
3. Обновить `UserOrmEntity`
4. Создать миграцию БД
5. Обновить `UserService.registerUser()` - убрать событие, ставить `['FREE']` сразу
6. Создать `UserEffectiveTariffsChangedHandler`
7. **Удалить** `UserPermissionsUpdatedHandler`

### Этап 4: Bot контекст
1. Обновить `BotUser` - заменить `permissions` на `effectiveTariffCodes`, имплементировать `UserInterface`
2. Убрать `use PermissionsTrait` из `BotUser`
3. Обновить `AccountPort` для маппинга
4. Инжектировать `AuthService` в Conversations и Router
5. Заменить `$user->can()` на `$authService->allowed()` во всех Conversations

### Этап 5: Monitoring контекст
1. Инжектировать `AuthService` в `AccountPort`
2. Заменить `$user->can()` на `$authService->allowed()`

### Этап 6: Финализация
1. Запустить миграцию БД
2. Warmup Redis кэша при деплое

### Этап 7: Документация
1. Обновить README с описанием новой системы прав
2. Добавить документацию по `AuthService` и его использованию
3. Описать flow событий `UserEffectiveTariffsChangedEvent` и `TariffPermissionsUpdatedEvent`
4. Добавить примеры использования в контекстах Bot и Monitoring

---

## Верификация

### Тесты

1. **Unit тесты:**
   - `AuthService::allowed()` с разными тарифами
   - `TariffPermissionsCache::getPermissions()` с fallback
   - `EffectiveTariffResolver::resolve()` с множественными подписками (проверить сортировку)

2. **Integration тесты:**
   - Регистрация пользователя → сразу доступен с `['FREE']`
   - Активация подписки → обновление `effectiveTariffCodes`
   - Проверка прав через `AuthService` в Bot контексте

3. **E2E тесты:**
   - Полный flow регистрации в Telegram
   - Изменение тарифа и проверка доступа к функциям

### Команды проверки

```bash
# Проверка миграции (через docker-compose)
docker-compose exec php php bin/console doctrine:migrations:migrate --dry-run

# Запуск миграции
docker-compose exec php php bin/console doctrine:migrations:migrate

# Warmup Redis кэша
docker-compose exec php php bin/console app:tariff-cache:warmup

# Проверка кэша
docker-compose exec redis redis-cli HGETALL tariff_permissions

# Запуск тестов (отложено)
# docker-compose exec php ./vendor/bin/phpunit --testsuite=unit
# docker-compose exec php ./vendor/bin/phpunit --testsuite=integration
```

---

## Диаграмма новой архитектуры

```
                    ┌─────────────────────────────────────────────┐
                    │                  Shared                      │
                    │                                              │
                    │  ┌──────────────────────────────────────┐   │
                    │  │ AuthService                          │   │
                    │  │  - allowed($user, $code, $value)     │   │
                    │  │  - getPermissionValue($user, $code)  │   │
                    │  └──────────────┬───────────────────────┘   │
                    │                 │                           │
                    │  ┌──────────────▼───────────────────────┐   │
                    │  │ TariffPermissionsCache (Redis)       │   │
                    │  │  - getPermissions($tariffCode)       │   │
                    │  └──────────────┬───────────────────────┘   │
                    │                 │                           │
                    │  ┌──────────────▼───────────────────────┐   │
                    │  │ BillingPort                          │   │
                    │  │  - getAllTariffPermissions()         │   │
                    │  └──────────────────────────────────────┘   │
                    └────────────────┬────────────────────────────┘
                                     │
            ┌────────────────────────┼────────────────────────┐
            │                        │                        │
            ▼                        ▼                        ▼
    ┌───────────────┐       ┌───────────────┐       ┌───────────────┐
    │     Bot       │       │   Monitoring  │       │    Account    │
    │               │       │               │       │               │
    │ BotUser       │       │ AccountPort   │       │ User          │
    │ (UserInterface)       │               │       │ (UserInterface)
    │               │       │               │       │               │
    │ authService   │       │ authService   │       │ effectiveTariff
    │  .allowed()   │       │  .allowed()   │       │ Codes[]       │
    └───────────────┘       └───────────────┘       └───────┬───────┘
                                                            │
                                                            │ Event
                            ┌───────────────────────────────┘
                            ▼
                    ┌───────────────────────────────────────────────┐
                    │                  Billing                       │
                    │                                                │
                    │  SubscriptionService                          │
                    │       ↓                                       │
                    │  EffectiveTariffResolver                      │
                    │    - resolve($userId) → ['PREMIUM','BASE','FREE']
                    │       ↓                                       │
                    │  UserEffectiveTariffsChangedEvent            │
                    │       → Account обновляет effectiveTariffCodes│
                    │                                                │
                    │  TariffPermissionsService                     │
                    │    - getAllTariffPermissions() → кэш Shared   │
                    └───────────────────────────────────────────────┘
```
