# Shared Context - Обзор

## Назначение

Контекст **Shared** содержит общие компоненты, используемые всеми другими контекстами. Это обеспечивает единообразие и переиспользование кода.

## Основные функции

1. **Система проверки прав**
   - `PermissionsTrait` - Трейт для унификации метода `can()`
   - `FeatureOptionInterface` - Контракт для стратегий проверки опций
   - `FeatureRegistry` - Реестр всех зарегистрированных стратегий

2. **Общие Value Objects**
   - `UserId` - Идентификатор пользователя
   - Другие общие типы данных

3. **Системные компоненты**
   - `HealthCheckController` - Проверка состояния системы

## Компоненты

### PermissionsTrait

Трейт для унификации метода `can()` во всех контекстах.

**Использование:**
```php
class User {
    use PermissionsTrait;
    
    private array $permissions; // JSON с правами
}
```

**Метод:**
```php
public function can(string $code, $value, FeatureRegistry $registry): bool
```

**Логика:**
1. Получает значение опции из кэша прав (`$this->permissions[$code]`)
2. Получает стратегию проверки из `FeatureRegistry`
3. Вызывает метод `check()` стратегии с текущим значением и значением из прав

### FeatureOptionInterface

Контракт для стратегий проверки опций.

**Методы:**
- `check($currentValue, $optionValue): bool` - Проверка значения

**Реализации:**
- `BooleanFeatureOption` - Проверка boolean значений
- `MaxFeatureOption` - Проверка числовых лимитов

### FeatureRegistry

Реестр всех зарегистрированных стратегий проверки опций.

**Регистрация:**
- Через Symfony tagged services
- Тег: `feature.option`

**Пример:**
```yaml
App\Context\Shared\Domain\FeatureOption\MaxFeatureOption:
    tags:
        - { name: 'feature.option', code: 'MAX_GROUP' }
```

**Использование:**
```php
$strategy = $this->featureRegistry->getStrategy('MAX_GROUP');
$result = $strategy->check($currentValue, $optionValue);
```

## Value Objects

### UserId

Идентификатор пользователя.

**Использование:**
- Типизация вместо примитивных типов
- Обеспечение типобезопасности

**Пример:**
```php
$userId = new UserId(123);
$user = $this->userRepository->findById($userId);
```

## Health Check

### HealthCheckController

Контроллер для проверки состояния системы.

**Эндпоинт:**
- `GET /api/health`

**Ответ:**
```json
{
  "status": "ok",
  "timestamp": "2024-01-01T00:00:00+00:00",
  "services": {
    "database": "ok"
  }
}
```

**Проверки:**
- Подключение к базе данных
- Доступность Redis
- Доступность RabbitMQ (опционально)

## Интеграция с другими контекстами

### Account Context

**Использование:**
- `User` использует `PermissionsTrait`
- `UserId` используется для типизации

### Billing Context

**Использование:**
- `FeatureRegistry` используется для проверки прав
- `FeatureOptionInterface` реализуется стратегиями проверки

### Bot Context

**Использование:**
- `FeatureRegistry` используется для проверки прав пользователя
- `PermissionsTrait` используется в диалогах

### Monitoring Context

**Использование:**
- `FeatureRegistry` используется для проверки лимитов
- `UserId` используется для типизации

## Связанные документы

- [Система прав](permissions.md)
- [Billing Context - Система прав](../billing/permissions.md)
- [Account Context](../account/overview.md)
