# Внутренние модели Gateway Telegram

## Описание

Внутренние модели хранят состояние gateway в оперативной памяти. Они не сериализуются в MQ и не передаются за пределы сервиса. При перезапуске gateway все внутреннее состояние теряется (кроме `.session` файлов на volume).

---

## MessageEnvelope

Базовый формат для всех MQ-сообщений (входящих и исходящих). Используется consumer и publisher для единообразной обработки.

### Поля

| Поле | Тип | Описание |
|------|-----|----------|
| `type` | string | Тип сообщения (`command` name или `event` name) |
| `correlation_id` | string \| None | UUID корреляции |
| `timestamp` | datetime | Время создания сообщения |
| `payload` | dict | Тело сообщения (десериализуется в конкретную модель) |

### Использование

```python
@dataclass
class MessageEnvelope:
    type: str
    correlation_id: str | None
    timestamp: datetime
    payload: dict[str, Any]
```

Consumer получает raw JSON из RabbitMQ → создаёт `MessageEnvelope` → по `type` определяет Pydantic-модель → валидирует `payload`.

---

## AuthState

Состояние незавершённой авторизации. Хранится в памяти на время auth flow (от `RequestAuthCode` до `SessionAuthorized` или `SessionFailed`).

### Поля

| Поле | Тип | Описание |
|------|-----|----------|
| `session_id` | string (UUID) | ID сессии в backend |
| `external_session_id` | string | Сгенерированный ID сессии в gateway |
| `phone_number` | string | Номер телефона (хранится до завершения auth) |
| `phone_code_hash` | string | Hash от Telethon `send_code_request` (нужен для `sign_in`) |
| `state` | AuthFlowState | Текущее состояние flow |
| `created_at` | datetime | Время создания |
| `correlation_id` | string | ID корреляции текущей команды |

### AuthFlowState (enum)

```python
class AuthFlowState(str, Enum):
    AWAITING_CODE = "awaiting_code"       # Код отправлен, ждём SubmitAuthCode
    AWAITING_2FA = "awaiting_2fa"         # 2FA запрошен, ждём Submit2FA
    COMPLETED = "completed"               # Авторизация завершена (success или fail)
```

### Жизненный цикл

```
RequestAuthCode → AuthState создан (AWAITING_CODE)
                      │
                SubmitAuthCode
                      │
              ┌───────┴────────┐
              │                │
         Success          2FA Required
         (COMPLETED)      (AWAITING_2FA)
                               │
                          Submit2FA
                               │
                        ┌──────┴──────┐
                        │             │
                   Success         Failed
                   (COMPLETED)     (COMPLETED)
```

### TTL

AuthState имеет TTL **15 минут**. Если за это время auth flow не завершён, состояние удаляется и клиент disconnected. Backend получит timeout (отсутствие ответного события).

### Хранение

```python
# In-memory storage в AuthHandler
_auth_states: dict[str, AuthState] = {}  # key = sessionId (backend UUID)
```

### Важно

* `phone_number` и `phone_code_hash` **никогда не логируются**
* После завершения auth flow (success или fail) `AuthState` удаляется из памяти
* При перезапуске gateway все незавершённые auth flow теряются

---

## ActiveClient

Обёртка над `TelegramClient` в пуле клиентов. Хранит метаданные для управления жизненным циклом.

### Поля

| Поле | Тип | Описание |
|------|-----|----------|
| `external_session_id` | string | ID сессии в gateway (имя `.session` файла) |
| `session_id` | string (UUID) \| None | ID сессии в backend (None для сервисных аккаунтов) |
| `client` | TelegramClient | Экземпляр Telethon-клиента |
| `is_service_account` | bool | Является ли клиент сервисным аккаунтом |
| `user_groups` | UserGroupCache \| None | Кэш групп пользователя (None для SA) |
| `connected_at` | datetime | Время подключения |
| `last_activity_at` | datetime | Время последней активности |

### Хранение

```python
# In-memory storage в ClientPool
_clients: dict[str, ActiveClient] = {}  # key = externalSessionId
```

### Операции

| Метод | Описание |
|-------|----------|
| `get(external_session_id)` | Получить клиент по ID |
| `add(active_client)` | Добавить клиент в пул |
| `remove(external_session_id)` | Отключить и удалить клиент |
| `get_all()` | Получить все активные клиенты |
| `get_user_clients()` | Все пользовательские клиенты (не SA) |
| `has(external_session_id)` | Проверить наличие клиента |

---

## ServiceAccountInfo

Описание одного сервисного аккаунта в пуле. Сервисные аккаунты используются для мониторинга публичных групп.

### Поля

| Поле | Тип | Описание |
|------|-----|----------|
| `session_name` | string | Имя `.session` файла (например `sa_1`) |
| `external_session_id` | string | Уникальный ID в пуле (совпадает с `session_name`) |
| `client` | TelegramClient | Экземпляр Telethon-клиента |
| `joined_groups` | set[int] | chat_id групп, в которых SA реально состоит в TG |

### Хранение

```python
# In-memory storage в ServiceAccountPool
_accounts: list[ServiceAccountInfo] = []
_group_to_account: dict[int, str] = {}  # chat_id → external_session_id
```

### Операции

| Метод | Описание |
|-------|----------|
| `get_for_group(chat_id)` | SA, который уже в этой группе (или None) |
| `get_least_loaded()` | SA с наименьшим числом групп |
| `add_group(external_session_id, chat_id)` | Зафиксировать что SA вступил в группу |
| `get_all()` | Все сервисные аккаунты |
| `count()` | Количество SA |

### Важно

* Все SA подключаются при старте gateway, FATAL если хоть один не авторизован
* `joined_groups` обновляется при `JoinChannelRequest` (добавление) и `LeaveChannelRequest` (удаление)
* Два SA **никогда** не вступают в одну и ту же группу (гарантируется через `_group_to_account`)

---

## UserGroupCache

Кэш групп и каналов, в которых состоит пользователь. Получается через `get_dialogs()` после авторизации и обновляется при MonitoringStart.

### Поля

| Поле | Тип | Описание |
|------|-----|----------|
| `session_id` | string | `external_session_id` пользовательской сессии |
| `chat_ids` | set[int] | Числовые Telegram chat ID групп/каналов |
| `last_updated` | datetime | Время последнего обновления |

### Использование

```python
@dataclass
class UserGroupCache:
    session_id: str
    chat_ids: set[int]
    last_updated: datetime
```

### Хранение

Хранится внутри `ActiveClient.user_groups`.

### Обновление

| Когда | Как |
|-------|-----|
| После авторизации (`_complete_auth`) | `get_dialogs()` → фильтр групп/каналов |
| При MonitoringStart для приватной группы | `get_dialogs()` → обновить кэш |
| Best-effort | Если `get_dialogs()` не удался — используем старый кэш + `GetParticipantRequest` |

### Важно

* Кэш может быть устаревшим (пользователь вступил/вышел из группы после последнего обновления)
* При MonitoringStart всегда дополнительно проверяем `GetParticipantRequest` если chat_id нет в кэше
* Кэш теряется при перезапуске gateway (пересоздаётся при `StartSession`)

---

## ActiveListener

Активная подписка на получение сообщений из одного Telegram-чата. **Один ActiveListener на один chat_id** — дедупликация: несколько пользователей (MonitoringEntry) используют один Telethon event handler.

### Поля

| Поле | Тип | Описание |
|------|-----|----------|
| `chat_id` | int | Числовой Telegram chat ID |
| `external_group_id` | string | `@username` или строковый chat ID |
| `listener_session_id` | string | `external_session_id` сессии, через которую слушаем |
| `listener_handle` | object | Handle от `client.add_event_handler()` для отписки |
| `is_public` | bool | Публичная группа (через SA) или приватная |
| `monitoring_entries` | list[MonitoringEntry] | Все подписки на мониторинг этой группы |
| `created_at` | datetime | Время создания listener |

### Пример

```
ActiveListener:
    chat_id: -1001234567890
    external_group_id: "@cryptonews"
    listener_session_id: "sa_1"          # слушаем через SA #1
    listener_handle: <handler>
    is_public: True
    monitoring_entries:
      - MonitoringEntry(group_id="uuid-1", requesting_session_id=null)   # пользователь A
      - MonitoringEntry(group_id="uuid-2", requesting_session_id=null)   # пользователь B
```

```
ActiveListener:
    chat_id: -1009876543210
    external_group_id: "-1009876543210"
    listener_session_id: "user_session_a"   # слушаем через сессию пользователя A
    listener_handle: <handler>
    is_public: False
    monitoring_entries:
      - MonitoringEntry(group_id="uuid-3", requesting_session_id="user_session_a")
      - MonitoringEntry(group_id="uuid-4", requesting_session_id="user_session_b")  # fallback
```

### Инварианты

1. **Один listener на chat_id** — не может быть двух ActiveListener с одинаковым chat_id
2. **listener_session_id** должна быть активной сессией в ClientPool (или SA в ServiceAccountPool)
3. **monitoring_entries** не может быть пустым — если удалён последний entry, удаляется и listener
4. Для **публичных** групп `listener_session_id` всегда указывает на SA
5. Для **приватных** групп `listener_session_id` указывает на одну из пользовательских сессий

---

## MonitoringEntry

Запись о том, что конкретная группа (backend UUID) включена в мониторинг. Привязана к `ActiveListener`.

### Поля

| Поле | Тип | Описание |
|------|-----|----------|
| `group_id` | string (UUID) | ID группы в backend |
| `requesting_session_id` | string \| None | `external_session_id` сессии, запросившей мониторинг. `null` для публичных |
| `is_public` | bool | Публичная группа |
| `added_at` | datetime | Время добавления |

### Важно

* `group_id` уникален глобально — одна группа не может мониториться дважды
* `requesting_session_id` для приватных групп — это сессия пользователя, а **не** сессия-listener
* Для публичных `requesting_session_id = null` (SA выбирается gateway автоматически)
* При удалении MonitoringEntry проверяем: остались ли другие entries в ActiveListener

---

## Связанные документы

* [Models Overview](overview.md)
* [Auth Handler](../auth/auth-handler.md)
* [Client Pool](../sessions/client-pool.md)
* [Service Account Pool](../sessions/service-account.md)
* [Monitoring Registry](../groups/group-registry.md)
* [User Groups](../auth/user-groups.md)
