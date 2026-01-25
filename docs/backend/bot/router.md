# Роутинг (BotRouter)

## Обзор

`BotRouter` отвечает за маршрутизацию входящих сообщений от мессенджеров в соответствующие обработчики (диалоги).

## Принципы работы

### Разделение обязанностей

1. **BotRouter** - только роутинг:
   - Определяет тип запроса (навигационная команда или текстовое сообщение)
   - Направляет запрос в соответствующий обработчик
   - Работает с адаптерами мессенджеров и отправкой ответов
   - НЕ работает напрямую с диалогами и состоянием

2. **ConversationManager** - полное управление диалогами:
   - Проверка наличия активного диалога
   - Получение диалога по коду
   - Управление жизненным циклом диалогов (старт, возобновление, обработка)
   - Переход к конкретному шагу диалога
   - Работа с состоянием через StateRepository

## Унифицированный формат команд

Все навигационные действия используют унифицированный формат:

```
/conversationCode[:step[:param1[:param2[:...:paramN]]]]
```

**Примеры:**
- `/billing` — запуск диалога
- `/billing:stepSelect` — переход к шагу
- `/billing:stepPayment:premium` — шаг с параметром
- `/setupfilter:stepEdit:uuid-123` — шаг с ID

Этот формат используется как для команд (текстовые сообщения), так и для callback_data (inline-кнопки).

Подробнее см. [Унифицированные команды](unified-commands.md).

## Алгоритм роутинга

### Логика `handleUpdate()`

```
┌─────────────────────────────────────────┐
│        Входящее обновление              │
└───────────────────┬─────────────────────┘
                    ▼
        ┌───────────────────────┐
        │  Парсинг BotRequest   │
        │  (command, step,      │
        │   params, message)    │
        └───────────┬───────────┘
                    ▼
          ┌─────────────────┐
          │ Есть command?   │
          └────────┬────────┘
         Да        │        Нет
      ┌────────────┴────────────┐
      ▼                         ▼
┌─────────────┐        ┌─────────────────┐
│handleNavig- │        │Есть активный    │
│ation()     │        │контекст?        │
└─────────────┘        └────────┬────────┘
                       Да       │       Нет
                  ┌─────────────┴──────────┐
                  ▼                        ▼
          ┌─────────────┐          ┌─────────────┐
          │handleConver-│          │"Используйте│
          │sation()    │          │ команды"    │
          └─────────────┘          └─────────────┘
```

### 1. Навигационная команда (command !== null)

```php
private function handleNavigation(UserId $userId, BotRequest $request, $user, BotInterface $adapter): void
```

**Логика:**
- **Только command без step** → очистка контекста, запуск диалога с `stepStart`
- **command + step** → проверка контекста:
  - Если код совпадает с активным → переход к шагу через `goToStep()`
  - Если код не совпадает → очистка контекста, запуск нового диалога с указанным шагом

### 2. Текстовое сообщение в активном диалоге

```php
private function handleConversation(UserId $userId, BotRequest $request, BotInterface $adapter): void
```

**Логика:**
- Передает сообщение в текущий шаг диалога через `ConversationManager::handleMessage()`
- Диалог обрабатывает ввод и возвращает ответ

### 3. Текстовое сообщение без активного диалога

**Логика:**
- Отправляет подсказку: "Используйте команды для взаимодействия с ботом"

## Глобальные команды

Команды `/start` и `/menu` имеют особую логику:
- Всегда сбрасывают текущий активный диалог
- Запускают соответствующий диалог с чистого листа

```php
if (in_array($command, ['start', 'menu'], true)) {
    if ($this->conversationManager->hasActiveConversation($userId)) {
        $this->conversationManager->cancelActiveConversation($userId);
    }
}
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
3. Парсинг запроса в `BotRequest`
4. Получение и валидация пользователя
5. Подтверждение callback query (если есть)
6. Роутинг в соответствующий обработчик

### handleNavigation()

Обработка навигационной команды.

**Логика:**
1. Получает диалог по коду через `ConversationManager::getConversationByCode()`
2. Проверяет права доступа (если требуется)
3. Если есть step и активный диалог совпадает → `goToStep()`
4. Иначе → очистка контекста и `startConversation()`

### goToStep()

Переход к конкретному шагу в активном диалоге.

**Параметры:**
- `UserId $userId` - ID пользователя
- `BotRequest $request` - Запрос с указанием шага и параметров
- `BotInterface $adapter` - Адаптер мессенджера

**Логика:**
- Вызывает `ConversationManager::goToStep()`
- Передает request с command, step и params

### handleConversation()

Обработка сообщения в активном диалоге.

**Параметры:**
- `UserId $userId` - ID пользователя
- `BotRequest $request` - Запрос от мессенджера
- `BotInterface $adapter` - Адаптер мессенджера

### startConversation()

Запуск нового диалога.

**Параметры:**
- `UserId $userId` - ID пользователя
- `BotRequest $request` - Запрос от мессенджера
- `ConversationInterface $conversation` - Диалог для запуска
- `?string $stepName` - Имя шага для перехода (опционально)
- `BotInterface $adapter` - Адаптер мессенджера

## Интеграция с ConversationManager

```php
// Проверка активного диалога
$this->conversationManager->hasActiveConversation($userId);

// Получение кода активного диалога
$activeCode = $this->conversationManager->getActiveConversationCode($userId);

// Переход к шагу
$response = $this->conversationManager->goToStep($userId, $request);

// Запуск диалога
$response = $this->conversationManager->startOrResume($code, $userId, $request, $stepName);

// Обработка сообщения
$response = $this->conversationManager->handleMessage($userId, $request);
```

## Обработка Callback Query

Callback query (нажатие inline-кнопки) обрабатывается как обычная навигационная команда:

1. `TelegramAdapter::parseRequest()` парсит callback_data в унифицированный формат
2. `BotRouter` получает `BotRequest` с command, step, params
3. Роутинг происходит по стандартной логике

Подтверждение нажатия (ack) выполняется автоматически:

```php
private function handleCallbackQueryAck(BotRequest $request, BotInterface $adapter): void
{
    $callbackQueryId = $request->getMetadataValue('callback_query_id');
    if ($callbackQueryId !== null) {
        $adapter->answerCallbackQuery($callbackQueryId);
    }
}
```

## Обработка ошибок

### Централизованная обработка

Все ошибки обрабатываются через единый метод `handleError()`:

```php
private function handleError(
    int $telegramUserId,
    BotInterface $adapter,
    string $userMessage,
    string $logMessage,
    array $context = []
): void {
    $this->logger->error($logMessage, array_merge($context, ['telegramUserId' => $telegramUserId]));
    $adapter->sendMessage($telegramUserId, $userMessage);
}
```

### BotMessageProvider

Централизованное управление сообщениями бота.

```php
$this->messageProvider->getUseCommands();          // Подсказка использовать команды
$this->messageProvider->getUnknownCommand();       // Неизвестная команда
$this->messageProvider->getConversationError();    // Ошибка диалога
$this->messageProvider->getUserNotRegistered();    // Пользователь не зарегистрирован
$this->messageProvider->getInsufficientPermissions(); // Недостаточно прав
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
- `step` - Шаг

## Связанные документы

- [Обзор Bot Context](overview.md)
- [Диалоги](conversations.md)
- [Унифицированные команды](unified-commands.md)
- [Вебхуки](webhooks.md)
- [Account Context](../account/overview.md)
