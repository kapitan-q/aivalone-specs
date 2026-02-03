# События Bot Context

## Описание

Bot Context генерирует события для информирования других контекстов об изменениях. Все события передаются через асинхронный Event Bus.

## Исходящие события

| Событие                                                           | Описание                                            | Получатель      |
| ----------------------------------------------------------------- | --------------------------------------------------- | --------------- |
| [BotUserConnected](bot-user-connected.md)                         | Пользователь подключился к боту                     | Account Context |
| [BotUserDisconnected](bot-user-disconnected.md)                   | Пользователь отключился от бота                     | Account Context |
| [NotificationEndpointRegistered](notification-endpoint-registered.md) | Зарегистрирован endpoint для доставки              | Account Context |
| [NotificationEndpointRevoked](notification-endpoint-revoked.md)   | Endpoint отозван                                    | Account Context |

## Входящие команды

Bot Context принимает команды от других контекстов:

| Команда                                                           | Описание                                            | Отправитель     |
| ----------------------------------------------------------------- | --------------------------------------------------- | --------------- |
| [SendMessageToUser](../processes/send-message-to-user.md)         | Отправить сообщение пользователю                    | Any Context     |
| BroadcastMessage (future)                                         | Массовая рассылка                                   | Admin Context   |

## Принципы событий

* События не содержат внешних идентификаторов (chat_id)
* События содержат только userId и messenger
* Асинхронная доставка через Symfony Messenger
* Все события реализуют `DomainEventInterface`

## Связанные документы

* [Bot Context Overview](../overview.md)
* [DomainEventInterface](../../backend/shared/events/domain-event-interface.md)
* [EventBus](../../backend/shared/events/event-bus-interface.md)
