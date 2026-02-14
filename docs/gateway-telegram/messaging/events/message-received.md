# Событие: MessageReceived

## Описание

Получено новое сообщение из отслеживаемой группы. Push-событие, не связанное с конкретной командой. Отправляется **одно событие на одно сообщение** вне зависимости от количества подписчиков — backend сам определяет, какие groupId привязаны к этому чату.

**Wire name (для backend)**: `MessageReceived`
**Routing key**: `telegram.message.received`

## Когда генерируется

При срабатывании Telethon event handler `events.NewMessage` для группы, зарегистрированной в MonitoringRegistry (`is_monitored(chat_id) == True`).

## Поля

| Поле | Тип | Описание |
|------|-----|----------|
| `event` | string | `"MessageReceived"` |
| `messengerType` | string | `"telegram"` |
| `chatId` | int | Числовой Telegram chat ID |
| `externalGroupId` | string | Строковый chat ID (для совместимости) |
| `messageId` | string | ID сообщения в Telegram |
| `content` | string | Текст сообщения |
| `senderName` | string | Имя отправителя |
| `sentAt` | string (ISO 8601) | Время отправки |
| `metadata.hasMedia` | boolean | Есть ли медиа |
| `metadata.replyToMessageId` | string \| null | ID сообщения-ответа |
| `timestamp` | string (ISO 8601) | Время генерации события |

> **`group_id` отсутствует!** Backend определяет группы по `chatId` / `externalGroupId`. Это позволяет gateway отправлять одно событие даже если 10 пользователей мониторят одну группу.

> **`correlationId` и `sessionId` отсутствуют** — это push-событие.

## JSON-пример

```json
{
    "event": "MessageReceived",
    "messengerType": "telegram",
    "chatId": -1001234567890,
    "externalGroupId": "-1001234567890",
    "messageId": "123456",
    "content": "Bitcoin reached new ATH today!",
    "senderName": "CryptoBot",
    "sentAt": "2024-01-15T14:30:00Z",
    "metadata": {
        "hasMedia": false,
        "replyToMessageId": null
    },
    "timestamp": "2024-01-15T14:30:01Z"
}
```

## Фильтрация

Gateway отправляет **все** текстовые сообщения. Фильтрация выполняется на стороне backend (Monitoring → Filtering Context).

Сообщения **без текста** (только медиа) — **не отправляются** (content будет пустым, backend не сможет фильтровать).

## Дедупликация

```
Telegram message (chat_id: -100ABC)
    │
    ▼
MonitoringRegistry.is_monitored(chat_id)?
    ├── YES → publish ONE MessageReceived
    └── NO  → skip (SA в группе, но никто не мониторит)
```

Даже если 10 пользователей мониторят одну группу — gateway отправляет **одно** событие. Backend сам маршрутизирует по `externalGroupId`.

## Действия backend при получении

1. Найти все MonitoredGroup по `externalGroupId` + `messengerType`
2. Для каждой найденной группы: найти привязанные фильтры
3. Отправить FilterMessageCommand в Filtering Context
4. При совпадении — отправить уведомление через Bot Context

## Связанные документы

* [Models — Events](../../models/events.md#messagereceived)
* [Message Listener](../../groups/message-listener.md)
* [Monitoring Registry](../../groups/group-registry.md)
