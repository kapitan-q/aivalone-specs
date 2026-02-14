# Message Listener

## Описание

`MessageListener` создаёт Telethon event handlers для получения новых сообщений из групп. Ключевое отличие от предыдущей версии: **один handler на один chat_id**, и перед отправкой события проверяется, мониторится ли группа.

## Файл

`src/groups/listener.py`

## Интерфейс

```python
class MessageListener:
    def __init__(
        self,
        publisher: Publisher,
        monitoring_registry: MonitoringRegistry,
    ): ...

    def register(self, client: TelegramClient, chat_id: int) -> object:
        """Регистрирует event handler для chat_id. Возвращает handle."""

    def unregister(self, client: TelegramClient, handle: object) -> None:
        """Снимает event handler."""
```

## Pseudocode

```python
class MessageListener:
    def __init__(self, publisher: Publisher, monitoring_registry: MonitoringRegistry):
        self._publisher = publisher
        self._registry = monitoring_registry

    def register(self, client: TelegramClient, chat_id: int) -> object:
        async def on_new_message(event: events.NewMessage.Event):
            await self._handle_message(event)

        handler = client.add_event_handler(
            on_new_message,
            events.NewMessage(chats=[chat_id]),
        )
        return handler

    async def _handle_message(self, event: events.NewMessage.Event):
        message = event.message
        chat_id = event.chat_id

        # 1. Проверяем: мониторится ли группа?
        if not self._registry.is_monitored(chat_id):
            return  # Пропускаем — никто не мониторит

        # 2. Игнорируем сообщения без текста
        if not message.text:
            return

        # 3. Извлечение данных отправителя
        sender_name = await self._get_sender_name(event)

        # 4. Публикация ОДНОГО события (без group_id!)
        # Backend сам определит, какие groupId привязаны к этому чату
        await self._publisher.publish(
            MessageReceivedEvent(
                chat_id=chat_id,
                external_group_id=str(chat_id),
                message_id=str(message.id),
                content=message.text,
                sender_name=sender_name,
                sent_at=message.date.isoformat(),
                metadata=MessageMetadata(
                    has_media=message.media is not None,
                    reply_to_message_id=(
                        str(message.reply_to_msg_id)
                        if message.reply_to_msg_id
                        else None
                    ),
                ),
            ),
            routing_key="telegram.message.received",
        )

    async def _get_sender_name(self, event) -> str:
        try:
            sender = await event.get_sender()
            if sender is None:
                return "Unknown"
            if hasattr(sender, "first_name"):
                name = sender.first_name or ""
                if sender.last_name:
                    name += f" {sender.last_name}"
                return name.strip() or "Unknown"
            if hasattr(sender, "title"):
                return sender.title  # Channel post
            return "Unknown"
        except Exception:
            return "Unknown"
```

## Проверка мониторинга

```
Telethon event → on_new_message(event)
    │
    ├─ monitoring_registry.is_monitored(chat_id)?
    │  ├── ДА → извлекаем данные → publish MessageReceived
    │  └── НЕТ → return (пропускаем)
    │
    └─ Отправляем ОДНО событие с chat_id
       Backend сам найдёт все groupId по externalGroupId
```

> **Важно**: Для публичных групп SA может быть в группе, но никто не мониторит (все MonitoringEntry удалены). В этом случае `is_monitored()` вернёт `False` и сообщение будет пропущено.

## MessageReceived — что отправляем

### Поля события

| Поле | Тип | Описание |
|------|-----|----------|
| `chat_id` | int | Числовой Telegram chat ID |
| `external_group_id` | string | Строковый chat ID (для backend) |
| `message_id` | string | ID сообщения в Telegram |
| `content` | string | Текст сообщения |
| `sender_name` | string | Имя отправителя |
| `sent_at` | string (ISO) | Время отправки |
| `metadata` | object | Доп. информация |

> **Без `group_id`!** Backend получает `external_group_id` (chatId) и сам определяет, какие группы (groupId) подписаны на этот чат.

## Фильтрация сообщений

### Что отправляется

| Тип | Отправляется | Причина |
|-----|-------------|---------|
| Текстовое сообщение | ✅ | Основной контент для фильтрации |
| Сообщение с текстом + медиа | ✅ | Текст извлекается, медиа игнорируется |
| Только медиа (без текста) | ❌ | Backend не может фильтровать |
| Service message (join/leave) | ❌ | Не имеет `message.text` |
| Edited message | ❌ (MVP) | Post-MVP: `events.MessageEdited` |

### Почему не фильтруем на gateway

Gateway — тонкий адаптер. Вся логика фильтрации на backend:

1. Gateway не знает какие фильтры активны
2. Gateway не знает привязки фильтров к группам
3. Фильтрация может быть сложной (regex, NLP — post-MVP)

## Производительность

### Высоконагруженные группы

Для групп с большим потоком сообщений (>100/мин):

* Telethon обрабатывает каждое сообщение в event loop
* Публикация в RabbitMQ — async, не блокирует
* При пиковой нагрузке — RabbitMQ буферизирует

### Backpressure

Если RabbitMQ перегружен:

* `aio_pika` Publisher confirm — ждёт подтверждения
* При timeout — логируем ошибку, сообщение теряется (acceptable для MVP)
* Post-MVP: локальный буфер + retry

## Связанные документы

* [Groups Overview](overview.md)
* [Group Handler](group-handler.md)
* [Monitoring Registry](group-registry.md)
* [Событие MessageReceived](../messaging/events/message-received.md)
