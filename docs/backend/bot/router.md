# Роутинг (BotRouter)

## Обзор

`BotRouter` отвечает за маршрутизацию входящих сообщений от мессенджеров в соответствующие обработчики (диалоги или команды).

## Принципы работы

### Разделение обязанностей

1. **BotRouter** - только роутинг:
   - Определяет тип запроса (команда, callback, сообщение)
   - Направляет запрос в соответствующий обработчик
   - Работает с адаптерами мессенджеров и отправкой ответов
   - НЕ работает напрямую с диалогами и состоянием

2. **ConversationManager** - полное управление диалогами:
   - Проверка наличия активного диалога
   - Получение диалога по коду
   - Управление жизненным циклом диалогов (старт, возобновление, обработка)
   - Работа с состоянием через StateRepository

## Алгоритм роутинга

### 1. Получение и валидация пользователя

```php
$user = $this->resolveUser($update, $adapter);
```

**Логика:**
- Извлекает `telegram_id` из обновления
- Получает пользователя через `AccountPortInterface::getUserByTelegramId()`
- Если пользователь не найден - регистрирует нового

### 2. Проверка активного диалога

```php
if ($this->conversationManager->hasActiveConversation($userId)) {
    // Обработка в активном диалоге
}
```

**Логика:**
- Проверяет наличие активного диалога через `ConversationManager`
- Если диалог активен - передает сообщение в `handleConversation()`

### 3. Обработка callback query

```php
if ($adapter->isCallbackQuery($update)) {
    // Обработка callback query
}
```

**Логика:**
- Определяет тип callback query (команда или данные)
- Если команда - запускает новый диалог
- Если данные - обрабатывает в активном диалоге

### 4. Обработка команды

```php
$command = $this->extractCommand($update, $adapter);
if ($command) {
    // Запуск диалога по команде
}
```

**Логика:**
- Извлекает команду из сообщения
- Получает диалог по команде через `ConversationManager::getConversationByCode()`
- Проверяет права доступа (если требуется)
- Запускает диалог

### 5. Обработка обычного сообщения

```php
// Если нет активного диалога и нет команды - отправка сообщения об ошибке
```

## Методы BotRouter

### handleUpdate()

Главный метод для обработки обновлений.

**Параметры:**
- `array $update` - Обновление от мессенджера
- `string $messenger` - Код мессенджера (telegram, whatsapp и т.д.)

**Алгоритм:**
1. Валидация входных параметров
2. Получение адаптера через `BotFactory`
3. Получение и валидация пользователя
4. Проверка активного диалога
5. Роутинг в соответствующий обработчик

### handleConversation()

Обработка сообщения в активном диалоге.

**Параметры:**
- `UserId $userId` - ID пользователя
- `BotRequest $request` - Запрос от мессенджера
- `BotInterface $adapter` - Адаптер мессенджера

**Логика:**
- Получает активный диалог через `ConversationManager`
- Передает сообщение в диалог через `handle()`
- Отправляет ответ пользователю
- Обрабатывает ошибки

### startConversation()

Запуск нового диалога.

**Параметры:**
- `string $conversationCode` - Код диалога
- `UserId $userId` - ID пользователя
- `BotRequest $request` - Запрос от мессенджера
- `BotInterface $adapter` - Адаптер мессенджера

**Логика:**
1. Получает диалог через `ConversationManager::getConversationByCode()`
2. Проверяет права доступа (если требуется)
3. Запускает диалог через `ConversationManager::startOrResume()`
4. Отправляет ответ пользователю
5. Обрабатывает ошибки

### resolveUser()

Получение и валидация пользователя.

**Параметры:**
- `array $update` - Обновление от мессенджера
- `BotInterface $adapter` - Адаптер мессенджера

**Возвращает:**
- `User` - Пользователь

**Логика:**
- Извлекает `telegram_id` из обновления
- Получает пользователя через `AccountPortInterface`
- Если пользователь не найден - регистрирует нового
- Возвращает пользователя

### extractCommand()

Извлечение команды из обновления.

**Параметры:**
- `array $update` - Обновление от мессенджера
- `BotInterface $adapter` - Адаптер мессенджера

**Возвращает:**
- `?string` - Код команды или `null`

**Логика:**
- Извлекает команду из сообщения или callback query
- Нормализует команду (удаляет префикс '/')
- Возвращает код команды

## Интеграция с другими компонентами

### BotFactory

Фабрика для получения адаптеров мессенджеров.

**Использование:**
```php
$adapter = $this->botFactory->get($messenger);
```

### ConversationManager

Сервис для управления диалогами.

**Использование:**
```php
// Проверка активного диалога
if ($this->conversationManager->hasActiveConversation($userId)) {
    // ...
}

// Получение диалога по коду
$conversation = $this->conversationManager->getConversationByCode($code);

// Запуск или возобновление диалога
$response = $this->conversationManager->startOrResume($code, $userId, $request);
```

### AccountPortInterface

Интерфейс для взаимодействия с Account Context.

**Использование:**
```php
$user = $this->accountPort->getUserByTelegramId($telegramId);
```

### FeatureRegistry

Реестр стратегий проверки прав.

**Использование:**
```php
if ($user->can('MAX_GROUP', $currentCount, $this->featureRegistry)) {
    // ...
}
```

## Обработка ошибок

### Централизованная обработка

Все ошибки обрабатываются через единый метод `handleError()`:

```php
private function handleError(\Throwable $e, BotInterface $adapter, ?int $chatId = null): void
{
    // Логирование ошибки
    $this->logger->error('BotRouter error', [
        'exception' => $e,
        'chatId' => $chatId,
    ]);

    // Отправка сообщения об ошибке пользователю
    if ($chatId) {
        $adapter->sendMessage($chatId, $this->messageProvider->getErrorMessage());
    }
}
```

### BotMessageProvider

Централизованное управление сообщениями бота.

**Использование:**
```php
$message = $this->messageProvider->getErrorMessage();
$message = $this->messageProvider->getCommandNotFoundMessage();
$message = $this->messageProvider->getPermissionDeniedMessage();
```

## Логирование

Все операции логируются через PSR Logger:

- Успешные операции: `info`
- Ошибки: `error`
- Предупреждения: `warning`

**Контекст логов:**
- `messenger` - Код мессенджера
- `userId` - ID пользователя
- `conversationCode` - Код диалога
- `command` - Команда

## Связанные документы

- [Обзор Bot Context](overview.md)
- [Диалоги](conversations.md)
- [Вебхуки](webhooks.md)
- [Account Context](../account/overview.md)
