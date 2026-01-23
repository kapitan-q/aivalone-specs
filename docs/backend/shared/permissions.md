# Система прав (Shared)

## Обзор

Система проверки прав реализована в Shared Context для обеспечения единообразия во всех контекстах проекта.

## Компоненты

### PermissionsTrait

Трейт для унификации метода `can()` во всех контекстах.

**Использование:**
```php
class User {
    use PermissionsTrait;
    
    private array $permissions; // JSON с правами от Billing Context
}
```

**Метод:**
```php
public function can(string $code, $value, FeatureRegistry $registry): bool
```

**Параметры:**
- `$code` - Код опции (например, 'MAX_GROUP')
- `$value` - Текущее значение для проверки
- `$registry` - Реестр стратегий проверки

**Возвращает:**
- `bool` - Результат проверки

**Логика:**
1. Получает значение опции из кэша прав (`$this->permissions[$code]`)
2. Если опция отсутствует - возвращает `false`
3. Получает стратегию проверки из `FeatureRegistry`
4. Вызывает метод `check()` стратегии с текущим значением и значением из прав

### FeatureOptionInterface

Контракт для стратегий проверки опций.

**Методы:**
- `check($currentValue, $optionValue): bool` - Проверка значения

**Параметры:**
- `$currentValue` - Текущее значение (например, текущее количество групп)
- `$optionValue` - Значение из прав (например, максимальное количество групп)

**Возвращает:**
- `bool` - Результат проверки

### FeatureRegistry

Реестр всех зарегистрированных стратегий проверки опций.

**Регистрация:**
- Через Symfony tagged services
- Тег: `feature.option`

**Пример конфигурации:**
```yaml
App\Context\Shared\Domain\FeatureOption\MaxFeatureOption:
    tags:
        - { name: 'feature.option', code: 'MAX_GROUP' }

App\Context\Shared\Domain\FeatureOption\BooleanFeatureOption:
    tags:
        - { name: 'feature.option', code: 'CAN_USE_PRIVATE_GROUPS' }
        - { name: 'feature.option', code: 'CAN_USE_AI' }
        - { name: 'feature.option', code: 'CAN_USE_MORPHOLOGY' }
```

**Методы:**
- `getStrategy(string $code): FeatureOptionInterface` - Получение стратегии по коду

## Реализации стратегий

### BooleanFeatureOption

Проверка boolean значений.

**Логика:**
- Если опция отсутствует в правах - возвращает `false`
- Если опция присутствует и `value === true` - возвращает `true`
- Иначе - возвращает `false`

**Пример:**
```php
// Права: { "CAN_USE_AI": true }
$strategy = new BooleanFeatureOption();
$result = $strategy->check(true, true); // true
$result = $strategy->check(false, true); // false (не используется)
```

**Использование:**
```php
if ($user->can('CAN_USE_AI', true, $this->featureRegistry)) {
    // Разрешено использовать AI
}
```

### MaxFeatureOption

Проверка числовых лимитов.

**Логика:**
- Если опция отсутствует в правах - возвращает `false`
- Если `$maxValue === null` - возвращает `true` (без ограничений)
- Если `$currentValue < $maxValue` - возвращает `true`
- Иначе - возвращает `false`

**Пример:**
```php
// Права: { "MAX_GROUP": 5 }
$strategy = new MaxFeatureOption();
$result = $strategy->check(3, 5); // true (3 < 5)
$result = $strategy->check(5, 5); // false (5 >= 5)
$result = $strategy->check(6, 5); // false (6 >= 5)

// Права: { "MAX_GROUP": null } (без ограничений)
$result = $strategy->check(100, null); // true
```

**Использование:**
```php
$currentGroupCount = $this->groupRepository->countByUserId($userId);
if ($user->can('MAX_GROUP', $currentGroupCount, $this->featureRegistry)) {
    // Разрешено добавить группу
}
```

## Примеры использования

### Проверка лимита групп

```php
$user = $this->userRepository->findById($userId);
$currentGroupCount = $this->groupRepository->countByUserId($userId);

if (!$user->can('MAX_GROUP', $currentGroupCount, $this->featureRegistry)) {
    throw new LimitExceededException('Превышен лимит групп');
}

// Разрешено добавить группу
$this->groupRepository->save($group);
```

### Проверка доступа к приватным группам

```php
$user = $this->userRepository->findById($userId);

if (!$user->can('CAN_USE_PRIVATE_GROUPS', true, $this->featureRegistry)) {
    throw new AccessDeniedException('Доступ к приватным группам запрещен');
}

// Разрешено добавить приватную группу
```

### Проверка доступа к AI

```php
$user = $this->userRepository->findById($userId);

if (!$user->can('CAN_USE_AI', true, $this->featureRegistry)) {
    throw new AccessDeniedException('AI фильтрация недоступна на вашем тарифе');
}

// Разрешено использовать AI фильтрацию
```

## Добавление новой стратегии

### 1. Создание класса стратегии

```php
namespace App\Context\Shared\Domain\FeatureOption;

class CustomFeatureOption implements FeatureOptionInterface
{
    public function check($currentValue, $optionValue): bool
    {
        // Логика проверки
        return $currentValue === $optionValue;
    }
}
```

### 2. Регистрация в services.yaml

```yaml
App\Context\Shared\Domain\FeatureOption\CustomFeatureOption:
    tags:
        - { name: 'feature.option', code: 'CUSTOM_OPTION' }
```

### 3. Использование

```php
if ($user->can('CUSTOM_OPTION', $value, $this->featureRegistry)) {
    // Логика
}
```

## Интеграция с Billing Context

Права пользователя формируются в Billing Context через `PermissionAggregator` и сохраняются в Account Context в поле `permissions` (JSON).

**Формат прав:**
```json
{
  "MAX_GROUP": 5,
  "CAN_USE_PRIVATE_GROUPS": true,
  "CAN_USE_AI": false,
  "CAN_USE_MORPHOLOGY": true
}
```

**Обновление прав:**
- При изменении подписки Billing Context публикует событие `UserPermissionsUpdatedEvent`
- Account Context обновляет кэш прав в поле `permissions`

## Связанные документы

- [Обзор Shared Context](overview.md)
- [Billing Context - Система прав](../billing/permissions.md)
- [Account Context](../account/overview.md)
