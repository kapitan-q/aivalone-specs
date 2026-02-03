# Router Service Specification

## Назначение

`Router` — ключевой сервис Bot Context, отвечающий за маршрутизацию входящих сообщений к соответствующим диалогам (Conversations).

## Зависимости

```php
final class Router
{
    public function __construct(
        private ConversationRegistry $registry,
        private ConversationStateRepositoryInterface $stateRepository,
        private UserServiceInterface $userService,  // из Account Context
        private EventBusInterface $eventBus,
    ) {}
}
```

## Основной метод

### handle(BotRequest $request): BotResponse

Обработка входящего запроса.

**Алгоритм**:

```
1. Получить или создать пользователя
   → AccountContext::getOrCreateUser(messenger, externalUserId)
   → Если новый пользователь, генерируется UserRegistered

2. Получить текущий state пользователя
   → stateRepository->find(userId, messenger)

3. Определить диалог и шаг:

   3.1 Если команда (/start, /help):
       - conversationCode = request.command
       - stepCode = null
       - state = null (сброс состояния)

   3.2 Если команда с конкретным шагом (/code:step):
       - conversationCode = request.command
       - stepCode = request.step

   3.3 Если обычное сообщение:
       - conversationCode = state.conversationCode
       - stepCode = state.stepCode

   3.4 Если нет state и нет команды:
       - fallback → /help

4. Найти диалог в реестре
   → registry->get(conversationCode)
   → Если не найден: fallback → /help

5. Проверка конфликта диалогов
   → Если state !== null && state.conversationCode !== conversationCode
   → Выбросить ConversationConflictException
   → (или сбросить state для базовых команд)

6. Установить контекст диалога
   → conversation->setContextData(state.contextData ?? [])

7. Вызвать обработчик диалога
   → response = conversation->handle(stepCode, request.message, request.params)

8. Обработать ответ:
   8.1 Если response.finished:
       - stateRepository->delete(userId, messenger)

   8.2 Иначе:
       - state.conversationCode = conversation->getCode()
       - state.stepCode = conversation->getCurrentStep()
       - state.contextData = conversation->getContextData()
       - stateRepository->save(state)

9. Вернуть BotResponse для отправки
```

## Fallback Logic

```php
private function handleFallback(BotRequest $request): BotResponse
{
    $helpConversation = $this->registry->get('help');
    return $helpConversation->handle(null, null, []);
}
```

## Обработка базовых команд

Команды `/start` и `/help` являются "reset" командами — они сбрасывают текущий state:

```php
private function isResetCommand(string $command): bool
{
    return in_array($command, ['start', 'help'], true);
}
```

## Диаграмма потока

```mermaid
flowchart TD
    A[BotRequest] --> B[Get/Create User]
    B --> C[Get State]
    C --> D{Command?}

    D -->|/start, /help| E[Reset State]
    D -->|/code:step| F[Use from Command]
    D -->|Message| G{Has State?}

    G -->|Yes| H[Use from State]
    G -->|No| I[Fallback /help]

    E --> J[Find Conversation]
    F --> J
    H --> J
    I --> J

    J --> K{Found?}
    K -->|No| I
    K -->|Yes| L[handle()]

    L --> M[BotResponse]
    M --> N{Finished?}

    N -->|Yes| O[Delete State]
    N -->|No| P[Save State]

    O --> Q[Return Response]
    P --> Q
```

## Связанные документы

* [Services Overview](overview.md)
* [ConversationRegistry](conversation-registry.md)
* [ConversationState](../models/conversation-state.md)
* [BotRequest](../models/bot-request.md)
* [Handle Incoming Message Process](../processes/handle-incoming-message.md)

## Статус реализации

* [ ] Класс Router создан
* [ ] Метод handle() реализован
* [ ] Интеграция с ConversationRegistry
* [ ] Интеграция с StateRepository
* [ ] Интеграция с Account Context (getOrCreateUser)
* [ ] Fallback логика реализована
* [ ] Reset команды обрабатываются
* [ ] Unit тесты написаны
* [ ] Integration тесты написаны
