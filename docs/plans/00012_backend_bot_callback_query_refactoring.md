# План: Унификация обработки команд бота для мультимессенджерной платформы

## Цель
Унифицировать механизм обработки команд (включая callback_query от Telegram) для поддержки любых мессенджеров без модификации диалогов. Устранить привязку к Telegram-специфичному понятию callback_query.

## Проблема
1. В диалогах много условий типа `if ($callbackData === 'action:cancel')`
2. Нет единого формата callback_data: `action:cancel`, `plan:premium`, `strategy:morpho`, `menu`
3. BotRouter парсит callback_query специфичным для Telegram способом
4. Callback_query — понятие только Telegram, в WhatsApp/Viber может не быть

## Решение

### Унифицированный формат команд
```
/conversationCode[:step[:param1[:param2[:...:paramN]]]]
```
**Примеры:**
- `/billing` — запуск диалога
- `/billing:stepSelect` — переход к шагу
- `/billing:stepPayment:premium` — шаг с параметром
- `/setupfilter:stepEdit:uuid-123` — шаг с ID

---

## Фазы реализации

### Фаза 1: Расширение BotRequest

**Файл:** [BotRequest.php](services/aivalone-backend/src/Context/Bot/Application/DTO/BotRequest.php)

Обновить структуру:
```php
class BotRequest
{
    public function __construct(
        private int $userId,
        private string $message,
        private ?string $command = null,      // conversationCode (без слеша)
        private ?string $step = null,         // имя шага (stepSelect, stepPayment и т.д.)
        private array $params = [],           // массив параметров из команды
        private array $metadata = [],         // мета-данные мессенджера
    ) {}

    public function getStep(): ?string;
    public function getParams(): array;
    public function getParam(int $index, mixed $default = null): mixed;
    public function isNavigation(): bool; // command !== null || step !== null
}
```

**Удалить:** `callbackQuery`, `getCallbackQuery()`, `isCallbackQuery()`

---

### Фаза 2: Метод getLink() в AbstractConversation

**Файл:** [AbstractConversation.php](services/aivalone-backend/src/Context/Bot/Domain/Model/AbstractConversation.php)

Добавить методы:
```php
abstract class AbstractConversation
{
    /**
     * Формирует команду для перехода к шагу диалога
     * $this->getLink('stepSelect') → '/billing:stepSelect'
     * $this->getLink('stepPayment', ['premium']) → '/billing:stepPayment:premium'
     */
    public function getLink(string $step, array $params = []): string
    {
        $normalizedStep = str_starts_with($step, 'step') ? $step : 'step' . ucfirst($step);
        $parts = [$this->getCommand(), $normalizedStep];
        foreach ($params as $param) {
            $parts[] = (string) $param;
        }
        return implode(':', $parts);
    }

    /**
     * Команда для перехода в главное меню
     */
    public function getMenuLink(): string
    {
        return '/menu';
    }

    /**
     * Команда для отмены/возврата к началу диалога
     * Просто запускает диалог заново (очищает контекст)
     */
    public function getCancelLink(): string
    {
        return $this->getCommand(); // → '/billing'
    }
}
```

Обновить **ConversationInterface** — добавить `getLink()` в интерфейс.

---

### Фаза 3: Обновление TelegramAdapter

**Файл:** [TelegramAdapter.php](services/aivalone-backend/src/Context/Bot/Infrastructure/Messenger/TelegramAdapter.php)

Метод `parseCallbackQuery()` должен парсить унифицированный формат:
```php
private function parseCommandFormat(string $data): array
{
    $data = ltrim($data, '/');
    $parts = explode(':', $data);
    return [
        'command' => $parts[0] ?? null,
        'step' => $parts[1] ?? null,
        'params' => array_slice($parts, 2),
    ];
}
```

---

### Фаза 4: Обновление BotRouter

**Файл:** [BotRouter.php](services/aivalone-backend/src/Context/Bot/Application/Service/BotRouter.php)

Новая логика `handleUpdate()`:
1. **Только command** → очистка контекста, запуск диалога с `stepStart`
2. **command + step** → проверка контекста (код должен совпадать), вызов метода step
3. **Только message + активный контекст** → вызов сохранённого метода
4. **Message без контекста** → подсказка использовать команды
5. Проверить текущие методы, возможно они устарели и с новой логикой их можно упрастить/удалить

Добавить в **ConversationManager**:
```php
public function getActiveConversationCode(UserId $userId): ?string;
public function goToStep(UserId $userId, BotRequest $request): BotResponse;
```

---

### Фаза 5: Рефакторинг диалогов

#### 5.1 BillingConversation
**Файл:** [BillingConversation.php](services/aivalone-backend/src/Context/Bot/Application/Conversation/BillingConversation.php)

**Было:**
```php
['text' => '🔄 Сменить тариф', 'callback_data' => 'action:change']
['text' => '📜 История платежей', 'callback_data' => 'action:history']
```

**Стало:**
```php
['text' => '🔄 Сменить тариф', 'callback_data' => $this->getLink('stepSelect')]
['text' => '📜 История платежей', 'callback_data' => $this->getLink('stepHistory')]
```

Добавить: `stepHistory()`, `stepConfirmPayment()`

#### 5.2 SetupFilterConversation
**Файл:** [SetupFilterConversation.php](services/aivalone-backend/src/Context/Bot/Application/Conversation/SetupFilterConversation.php)

**Было:**
```php
['callback_data' => 'action:new']
['callback_data' => 'action:edit:' . $filter->id]
['callback_data' => 'action:cancel']
```

**Стало:**
```php
['callback_data' => $this->getLink('stepName')]
['callback_data' => $this->getLink('stepEdit', [$filter->id])]
['callback_data' => $this->getCancelLink()]  // → '/setupfilter' (начать заново)
```

Добавить: `stepEdit(BotRequest $request)` — получает ID из `$request->getParam(0)`

#### 5.3 Остальные диалоги
- **MenuConversation** — изменить callback_data на `/код_диалога`
- **AddGroupConversation** — аналогичный рефакторинг
- **HelpConversation**, **StartConversation**, **AuthConversation** — проверить и обновить

---

### Фаза 6: Тестирование

**Новые тесты:**
1. `BotRequestTest` — парсинг команд с параметрами
2. `AbstractConversationTest` — метод `getLink()`
3. `TelegramAdapterTest` — парсинг унифицированного формата
4. `BotRouterTest` — роутинг с новой логикой
5. Integration-тесты для каждого диалога

---

### Фаза 7: Документация

**Обновить:**
- [router.md](docs/backend/bot/router.md) — новый формат команд
- [conversations.md](docs/backend/bot/conversations.md) — метод `getLink()`

**Создать:**
- `docs/backend/bot/unified-commands.md` — спецификация формата команд

---

## Критические файлы для изменения

| Файл | Изменения |
|------|-----------|
| [BotRequest.php](services/aivalone-backend/src/Context/Bot/Application/DTO/BotRequest.php) | + step, params, методы |
| [AbstractConversation.php](services/aivalone-backend/src/Context/Bot/Domain/Model/AbstractConversation.php) | + getLink(), getMenuLink(), getCancelLink() |
| [ConversationInterface.php](services/aivalone-backend/src/Context/Bot/Domain/Model/ConversationInterface.php) | + getLink() |
| [TelegramAdapter.php](services/aivalone-backend/src/Context/Bot/Infrastructure/Messenger/TelegramAdapter.php) | Парсинг формата |
| [BotRouter.php](services/aivalone-backend/src/Context/Bot/Application/Service/BotRouter.php) | Новая логика роутинга |
| [ConversationManager.php](services/aivalone-backend/src/Context/Bot/Application/Service/ConversationManager.php) | + getActiveConversationCode(), goToStep() |
| [BillingConversation.php](services/aivalone-backend/src/Context/Bot/Application/Conversation/BillingConversation.php) | Рефакторинг callback_data |
| [SetupFilterConversation.php](services/aivalone-backend/src/Context/Bot/Application/Conversation/SetupFilterConversation.php) | Рефакторинг callback_data |
| Все остальные диалоги | Рефакторинг callback_data |

---

## Верификация

1. **Unit-тесты:**
   - `composer run test` — все тесты проходят

2. **Ручное тестирование в Telegram:**
   - `/billing` → показывает текущий тариф
   - Кнопка "Сменить тариф" → переход к stepSelect
   - Кнопка "Отмена" → возврат к началу диалога (stepStart)
   - `/setupfilter` → создание/редактирование фильтров
   - Редактирование фильтра с параметром ID работает корректно
   - Inline-кнопки работают корректно

3. **Проверка документации:**
   - Документация в `/docs/backend/bot/` обновлена и соответствует коду

---

## Риски и митигация

| Риск | Митигация |
|------|-----------|
| Сломается навигация | Поэтапная миграция, тесты на каждый шаг |
| Спецсимволы в параметрах | URL-encoding для сложных параметров |

---

## Примечание

Обратная совместимость не требуется — production версии ещё нет. Можно сразу удалять `getCallbackQuery()` и использовать только новый формат.
