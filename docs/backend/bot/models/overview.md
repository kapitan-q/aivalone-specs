# Модели Bot Context

## Описание

Доменные модели Bot Context отвечают за управление взаимодействием с мессенджерами и диалогами пользователей. Все модели являются частью **Domain Layer** и не зависят от инфраструктуры.

## Доменные модели

### AggregateRoot

| Модель                                             | Описание                                                      |
| -------------------------------------------------- | ------------------------------------------------------------- |
| [NotificationEndpoint](notification-endpoint.md)   | Унифицированная точка доставки сообщений пользователю         |
| [ConversationState](conversation-state.md)         | Состояние активного диалога пользователя                      |

### Value Objects

| Модель                                           | Описание                                            |
| ------------------------------------------------ | --------------------------------------------------- |
| [EndpointId](endpoint-id.md)                     | UUID идентификатор NotificationEndpoint             |
| [BotRequest](bot-request.md)                     | Унифицированный формат входящего сообщения          |
| [BotResponse](bot-response.md)                   | Унифицированный формат исходящего ответа            |

### Enums

| Enum                                              | Описание                                            |
| ------------------------------------------------- | --------------------------------------------------- |
| [EndpointStatus](endpoint-status-enum.md)         | Статус endpoint (ACTIVE, REVOKED)                   |

## Инварианты

* NotificationEndpoint всегда связан с userId из Account Context
* externalTargetId никогда не выходит за пределы Bot Context
* Один активный диалог на пользователя и мессенджер
* ConversationState удаляется при завершении диалога

## Связанные документы

* [Bot Context Overview](../overview.md)
* [Conversations](../conversations/overview.md)
* [Shared Models](../../backend/shared/models/overview.md)
