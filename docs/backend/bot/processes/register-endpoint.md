# Процесс: Register Endpoint

## Описание

Процесс регистрации NotificationEndpoint при первом подключении пользователя к боту.

## Диаграмма взаимодействия

```mermaid
sequenceDiagram
    participant SC as Start Conversation
    participant ER as Endpoint Repository
    participant E as NotificationEndpoint
    participant EB as Event Bus
    participant AC as Account Context

    SC->>ER: findByUserIdAndMessenger(userId, messenger)
    ER-->>SC: null (not exists)

    SC->>E: create(userId, messenger, chatId)
    E->>E: Generate EndpointId
    E->>E: Set status = ACTIVE
    E->>E: Record NotificationEndpointRegistered

    SC->>ER: save(endpoint)

    SC->>E: pullEvents()
    E-->>SC: [NotificationEndpointRegistered]

    SC->>EB: publish(NotificationEndpointRegistered)
    EB-->>AC: Handle event
```

## Пошаговое описание

### Шаг 1: Проверка существующего endpoint

```
StartConversation::stepStart()
    ↓
Проверка наличия endpoint:
    - endpointRepository->findByUserIdAndMessenger(userId, messenger)
    ↓
Если существует и активен:
    → Пропуск регистрации
    → Продолжение диалога

Если существует и отозван:
    → Создание нового endpoint

Если не существует:
    → Создание нового endpoint
```

### Шаг 2: Создание endpoint

```
NotificationEndpoint::create(userId, messenger, externalChatId)
    ↓
1. Генерация EndpointId (UUID)
2. Установка status = ACTIVE
3. createdAt = now()
4. Регистрация события NotificationEndpointRegistered
```

### Шаг 3: Сохранение и публикация события

```
endpointRepository->save(endpoint)
    ↓
endpoint->pullEvents()
    ↓
eventBus->publish(NotificationEndpointRegistered)
    ↓
Account Context получает событие
```

## Безопасность

**ВАЖНО**: externalChatId (chat_id) не включается в событие!

Событие содержит только:
- endpointId
- userId
- messenger
- occurredAt

## Связанные документы

* [Processes Overview](overview.md)
* [NotificationEndpoint](../models/notification-endpoint.md)
* [NotificationEndpointRegistered Event](../events/notification-endpoint-registered.md)
* [Start Conversation](../conversations/start-conversation.md)

## Статус реализации

* [ ] Логика в StartConversation реализована
* [ ] Интеграция с Endpoint Repository
* [ ] Событие публикуется
* [ ] Unit тесты написаны
