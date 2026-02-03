# Bot Context — Сводка спецификаций

## Описание

Полный список всех спецификаций Bot Context с указанием статуса реализации.

## Структура документации

```
docs/bot/
├── overview.md                           # Обзор контекста
├── SPECIFICATIONS_SUMMARY.md             # Этот файл
│
├── models/                               # Доменные модели
│   ├── overview.md
│   ├── notification-endpoint.md          # AggregateRoot
│   ├── conversation-state.md             # Entity
│   ├── bot-request.md                    # Value Object
│   ├── bot-response.md                   # Value Object
│   ├── endpoint-id.md                    # Value Object
│   └── endpoint-status-enum.md           # Enum
│
├── events/                               # События
│   ├── overview.md
│   ├── bot-user-connected.md
│   ├── bot-user-disconnected.md
│   ├── notification-endpoint-registered.md
│   └── notification-endpoint-revoked.md
│
├── exceptions/                           # Исключения
│   ├── overview.md
│   ├── conversation-not-found.md
│   ├── invalid-command.md
│   ├── endpoint-not-found.md
│   ├── endpoint-already-revoked.md
│   ├── message-delivery.md
│   └── conversation-conflict.md
│
├── services/                             # Application сервисы
│   ├── overview.md
│   ├── router.md
│   ├── conversation-registry.md
│   └── message-sender.md
│
├── processes/                            # Бизнес-процессы
│   ├── overview.md
│   ├── handle-incoming-message.md
│   ├── send-message-to-user.md
│   ├── register-endpoint.md
│   └── start-conversation.md
│
├── infrastructure/                       # Инфраструктура
│   ├── overview.md
│   ├── notification-endpoint-repository.md
│   ├── conversation-state-repository.md
│   ├── messenger-adapter-interface.md
│   └── telegram-adapter.md
│
├── conversations/                        # Диалоги (FSM)
│   ├── overview.md
│   ├── abstract-conversation.md
│   ├── start-conversation.md
│   └── help-conversation.md
│
└── api/                                  # HTTP endpoints
    └── endpoints.md
```

## Статус реализации по слоям

### Domain Layer

| Компонент | Статус | Файл |
|-----------|--------|------|
| NotificationEndpoint | ⬜ Не реализован | models/notification-endpoint.md |
| EndpointId | ⬜ Не реализован | models/endpoint-id.md |
| EndpointStatus | ⬜ Не реализован | models/endpoint-status-enum.md |
| ConversationState | ⬜ Не реализован | models/conversation-state.md |
| BotRequest | ⬜ Не реализован | models/bot-request.md |
| BotResponse | ⬜ Не реализован | models/bot-response.md |
| BotUserConnected Event | ⬜ Не реализован | events/bot-user-connected.md |
| NotificationEndpointRegistered Event | ⬜ Не реализован | events/notification-endpoint-registered.md |
| ConversationNotFoundException | ⬜ Не реализован | exceptions/conversation-not-found.md |
| MessageDeliveryException | ⬜ Не реализован | exceptions/message-delivery.md |

### Application Layer

| Компонент | Статус | Файл |
|-----------|--------|------|
| Router | ⬜ Не реализован | services/router.md |
| ConversationRegistry | ⬜ Не реализован | services/conversation-registry.md |
| MessageSender | ⬜ Не реализован | services/message-sender.md |
| ConversationInterface | ⬜ Не реализован | conversations/overview.md |
| AbstractConversation | ⬜ Не реализован | conversations/abstract-conversation.md |
| StartConversation | ⬜ Не реализован | conversations/start-conversation.md |
| HelpConversation | ⬜ Не реализован | conversations/help-conversation.md |

### Infrastructure Layer

| Компонент | Статус | Файл |
|-----------|--------|------|
| NotificationEndpointRepository Interface | ⬜ Не реализован | infrastructure/notification-endpoint-repository.md |
| ConversationStateRepository Interface | ⬜ Не реализован | infrastructure/conversation-state-repository.md |
| MessengerAdapterInterface | ⬜ Не реализован | infrastructure/messenger-adapter-interface.md |
| TelegramAdapter | ⬜ Не реализован | infrastructure/telegram-adapter.md |
| Doctrine Entities | ⬜ Не реализован | — |
| Doctrine Repositories | ⬜ Не реализован | — |
| Database Migrations | ⬜ Не реализован | — |

### Presentation Layer

| Компонент | Статус | Файл |
|-----------|--------|------|
| WebhookController | ⬜ Не реализован | api/endpoints.md |
| CLI Commands | ⬜ Не реализован | — |

## MVP Scope

Для MVP необходимо реализовать:

1. ✅ Спецификации написаны
2. ⬜ Domain Layer
   - NotificationEndpoint, EndpointId, EndpointStatus
   - ConversationState
   - BotRequest, BotResponse
   - События и исключения
3. ⬜ Application Layer
   - Router
   - ConversationRegistry
   - MessageSender
   - StartConversation, HelpConversation
4. ⬜ Infrastructure Layer
   - Telegram Adapter (Nutgram)
   - Doctrine Repositories
   - Database Migrations
5. ⬜ Presentation Layer
   - Webhook Controller
   - CLI Commands

## Отложенные компоненты (Post-MVP)

- WhatsApp Adapter
- Discord Adapter
- Multi-endpoint support
- Fallback delivery
- Retry policies
- Advanced menu system
- Conversation analytics

## Легенда статусов

- ⬜ Не реализован
- 🔄 В процессе
- ✅ Реализован
- 🧪 Тестирование
