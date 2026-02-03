# Инфраструктура Bot Context

## Описание

Инфраструктурный слой Bot Context содержит реализации репозиториев, адаптеры мессенджеров и другие внешние интеграции.

## Компоненты

### Репозитории

| Компонент                                                              | Описание                                  |
| ---------------------------------------------------------------------- | ----------------------------------------- |
| [NotificationEndpoint Repository](notification-endpoint-repository.md) | Хранение и поиск NotificationEndpoint     |
| [ConversationState Repository](conversation-state-repository.md)       | Хранение состояний диалогов               |

### Messenger Adapters

| Компонент                                                    | Описание                                  |
| ------------------------------------------------------------ | ----------------------------------------- |
| [MessengerAdapter Interface](messenger-adapter-interface.md) | Интерфейс адаптера мессенджера            |
| [Telegram Adapter](telegram-adapter.md)                      | Реализация для Telegram (Nutgram)         |

## Связанные документы

* [Bot Context Overview](../overview.md)
* [Doctrine Entities](#)
