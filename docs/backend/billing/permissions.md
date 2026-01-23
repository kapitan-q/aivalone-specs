# Система прав (Permissions System)

## Обзор

Система проверки прав реализована на основе модели **наслаиваемых подписок**. Пользователь может иметь несколько одновременно активных подписок, права из которых агрегируются в единый набор.

## Архитектура

Система разделена на три контекста:

1. **Billing Context** - Владелец бизнес-логики тарифов и подписок
2. **Account Context** - Операционная проверка прав (хранит snapshot)
3. **Shared Domain** - Общая механика проверки прав

## Механизм работы

### 1. Активация подписки

При покупке подписки `SubscriptionService` создает новую подписку и вызывает `PermissionAggregator`.

### 2. Агрегация прав

`PermissionAggregator` выполняет следующие шаги:

1. Выбирает все активные подписки пользователя (`isActive = true` и `expiresAt > now()`)
2. Сортирует их по `TariffPlan.priority` (по убыванию)
3. Сливает `TariffOption` в единый массив:
   - Опции с высшим приоритетом перезаписывают низшие
   - Если у нескольких тарифов одна опция с одним кодом, берется значение из тарифа с высшим приоритетом
4. Генерирует итоговый JSON с правами

**Пример:**
```json
{
  "MAX_GROUP": null,
  "CAN_USE_PRIVATE_GROUPS": true,
  "CAN_USE_MORPHOLOGY": true,
  "CAN_USE_AI": true
}
```

### 3. Публикация прав

Агрегатор отправляет событие `UserPermissionsUpdatedEvent` с итоговым набором прав через Symfony Messenger.

### 4. Синхронизация

`UserPermissionsUpdatedHandler` в Account Context обновляет кэш прав пользователя в поле `permissions` (JSON).

**Важно:** Обновление идемпотентно - повторные события с теми же правами не изменяют данные.

### 5. Проверка прав

При проверке прав используется `FeatureRegistry` для получения стратегии проверки конкретной опции.

**Пример:**
```php
$user = $this->userRepository->findById($userId);
$registry = $this->featureRegistry;

// Проверка лимита групп
if ($user->can('MAX_GROUP', $currentGroupCount, $registry)) {
    // Разрешено добавить группу
}

// Проверка доступа к AI
if ($user->can('CAN_USE_AI', true, $registry)) {
    // Разрешено использовать AI фильтрацию
}
```

## Доступные коды опций (FeatureOption)

### MAX_GROUP

Максимальное количество групп.

**Стратегия:** `MaxFeatureOption`
**Тип:** Числовое значение
**Проверка:** `$currentCount < $maxValue`

### CAN_USE_PRIVATE_GROUPS

Доступ к приватным группам.

**Стратегия:** `BooleanFeatureOption`
**Тип:** Boolean
**Проверка:** `$value === true`

### CAN_USE_AI

Доступ к AI фильтрации.

**Стратегия:** `BooleanFeatureOption`
**Тип:** Boolean
**Проверка:** `$value === true`

### CAN_USE_MORPHOLOGY

Морфологическая фильтрация.

**Стратегия:** `BooleanFeatureOption`
**Тип:** Boolean
**Проверка:** `$value === true`

## Стратегии проверки (FeatureOption)

### BooleanFeatureOption

Проверяет boolean значения.

**Логика:**
- Если опция отсутствует в правах - возвращает `false`
- Если опция присутствует и `value === true` - возвращает `true`
- Иначе - возвращает `false`

### MaxFeatureOption

Проверяет числовые лимиты.

**Логика:**
- Если опция отсутствует в правах - возвращает `false`
- Если `$maxValue === null` - возвращает `true` (без ограничений)
- Если `$currentValue < $maxValue` - возвращает `true`
- Иначе - возвращает `false`

## PermissionsTrait

Трейт для унификации метода `can()` во всех контекстах.

**Метод:**
```php
public function can(string $code, $value, FeatureRegistry $registry): bool
```

**Логика:**
1. Получает значение опции из кэша прав (`$this->permissions[$code]`)
2. Получает стратегию проверки из `FeatureRegistry`
3. Вызывает метод `check()` стратегии с текущим значением и значением из прав

## FeatureRegistry

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

## Примеры использования

### Проверка лимита групп

```php
$user = $this->userRepository->findById($userId);
$currentGroupCount = $this->groupRepository->countByUserId($userId);

if (!$user->can('MAX_GROUP', $currentGroupCount, $this->featureRegistry)) {
    throw new LimitExceededException('Превышен лимит групп');
}
```

### Проверка доступа к приватным группам

```php
$user = $this->userRepository->findById($userId);

if (!$user->can('CAN_USE_PRIVATE_GROUPS', true, $this->featureRegistry)) {
    throw new AccessDeniedException('Доступ к приватным группам запрещен');
}
```

### Проверка доступа к AI

```php
$user = $this->userRepository->findById($userId);

if (!$user->can('CAN_USE_AI', true, $this->featureRegistry)) {
    throw new AccessDeniedException('AI фильтрация недоступна на вашем тарифе');
}
```

## Связанные документы

- [Обзор Billing Context](overview.md)
- [Тарифы](tariffs.md)
- [Подписки](subscriptions.md)
- [Account Context](../account/overview.md)
- [Shared Context - PermissionsTrait](../shared/permissions.md)
