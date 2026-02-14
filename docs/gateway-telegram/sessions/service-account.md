# Service Account Pool

## Описание

Пул сервисных аккаунтов — набор предварительно авторизованных Telegram-аккаунтов, используемых gateway для прослушивания **публичных** групп. Не привязаны к пользователям системы.

Один аккаунт может быть участником ≈500 групп/каналов в Telegram. Пул позволяет масштабировать мониторинг публичных групп.

## Когда используется

| Тип группы | sessionId в JoinGroup | Используемый клиент |
|------------|----------------------|---------------------|
| Публичная | `null` | Один из SA из пула |
| Приватная | `externalSessionId` | Клиент пользователя |

## Конфигурация

### Переменные окружения

| Переменная | Тип | Описание |
|-----------|-----|----------|
| `SERVICE_ACCOUNTS` | JSON array | Список сервисных аккаунтов |

### Формат SERVICE_ACCOUNTS

```json
[
    {"phone": "+79001234567", "session": "sa_1"},
    {"phone": "+79001234568", "session": "sa_2"},
    {"phone": "+79001234569", "session": "sa_3"}
]
```

### .session файлы

```
{TELEGRAM_SESSION_DIR}/{session_name}.session
→ /app/sessions/sa_1.session
→ /app/sessions/sa_2.session
→ /app/sessions/sa_3.session
```

## Файл

`src/sessions/service_account_pool.py`

## Компонент: ServiceAccountPool

### Интерфейс

```python
class ServiceAccountPool:
    def __init__(self, config: list[ServiceAccountConfig]): ...

    async def initialize(self) -> None: ...
    def get_for_group(self, chat_id: int) -> ServiceAccountInfo | None: ...
    def get_least_loaded(self) -> ServiceAccountInfo: ...
    def add_group(self, external_session_id: str, chat_id: int) -> None: ...
    def remove_group(self, chat_id: int) -> None: ...
    def get_all(self) -> list[ServiceAccountInfo]: ...
    def get_by_id(self, external_session_id: str) -> ServiceAccountInfo | None: ...
    def count(self) -> int: ...
    async def disconnect_all(self) -> None: ...
```

### Внутреннее хранилище

```python
_accounts: list[ServiceAccountInfo]              # все SA
_by_id: dict[str, ServiceAccountInfo]            # external_session_id → SA
_group_to_account: dict[int, str]                # chat_id → external_session_id
```

## Инициализация при старте gateway

```
initialize():
    for config in SERVICE_ACCOUNTS:
        1. Проверить: файл {config.session}.session существует?
           └── Нет → FATAL: "SA session '{config.session}' not found."
               → Gateway НЕ стартует

        2. Создать клиент:
           client = TelegramClient(session_path, api_id, api_hash)

        3. Подключиться:
           await client.connect()

        4. Проверить авторизацию:
           authorized = await client.is_user_authorized()
           └── Нет → FATAL: "SA '{config.session}' not authorized."
               → Gateway НЕ стартует

        5. Получить информацию:
           me = await client.get_me()
           logger.info("sa_ready", session=config.session, name=me.first_name)

        6. Добавить в пул:
           _accounts.append(ServiceAccountInfo(
               session_name=config.session,
               external_session_id=config.session,
               client=client,
               joined_groups=set(),
           ))

    logger.info("sa_pool_ready", count=len(_accounts))
```

> **FATAL**: Если хотя бы один SA не авторизован или файл отсутствует — gateway не стартует. Все SA должны быть исправны.

## Выбор SA для группы

### Алгоритм

```python
def select_sa_for_group(self, chat_id: int) -> ServiceAccountInfo:
    # 1. Проверить: уже есть SA в этой группе?
    existing = self.get_for_group(chat_id)
    if existing:
        return existing

    # 2. Выбрать SA с наименьшим числом групп
    sa = self.get_least_loaded()
    return sa
```

### get_for_group

```python
def get_for_group(self, chat_id: int) -> ServiceAccountInfo | None:
    """SA, который уже состоит в этой группе."""
    sa_id = self._group_to_account.get(chat_id)
    if sa_id:
        return self._by_id[sa_id]
    return None
```

### get_least_loaded

```python
def get_least_loaded(self) -> ServiceAccountInfo:
    """SA с наименьшим числом присоединённых групп."""
    return min(self._accounts, key=lambda sa: len(sa.joined_groups))
```

### add_group / remove_group

```python
def add_group(self, external_session_id: str, chat_id: int) -> None:
    """Зафиксировать: SA вступил в группу."""
    self._group_to_account[chat_id] = external_session_id
    self._by_id[external_session_id].joined_groups.add(chat_id)

def remove_group(self, chat_id: int) -> None:
    """Зафиксировать: SA покинул группу."""
    sa_id = self._group_to_account.pop(chat_id, None)
    if sa_id and sa_id in self._by_id:
        self._by_id[sa_id].joined_groups.discard(chat_id)
```

## Гарантия: два SA не вступают в одну группу

Единственная точка входа для вступления SA в группу — метод `select_sa_for_group()`. Он **всегда** сначала проверяет `_group_to_account`. Если группа уже привязана к SA — возвращает его.

```
JoinGroup (публичная)
    │
    ▼
select_sa_for_group(chat_id)
    ├── chat_id в _group_to_account? → возвращаем существующий SA
    └── нет → get_least_loaded() → JoinChannelRequest → add_group()
```

## Первоначальная настройка

Сервисные аккаунты авторизуются **один раз вручную** перед первым запуском gateway:

```python
# scripts/setup_service_account.py
import asyncio
from telethon import TelegramClient

async def main():
    session_name = input("Session name (e.g. sa_1): ")
    phone = input("Phone number: ")

    client = TelegramClient(
        f"sessions/{session_name}",
        api_id=API_ID,
        api_hash=API_HASH,
    )
    await client.start(phone=phone)
    me = await client.get_me()
    print(f"SA authorized: {me.first_name} ({session_name})")
    await client.disconnect()

asyncio.run(main())
```

После выполнения скрипта `.session` файлы размещаются в Docker volume.

## Reconnect

При потере соединения SA:

```
1. Telethon пытается автоматический reconnect
2. Если reconnect успешен → продолжаем работу
3. Если reconnect не удался:
   → Логируем CRITICAL
   → Пытаемся reconnect каждые 30 секунд
   → НЕ отправляем SessionExpired (у SA нет backend sessionId)
   → Публичные группы этого SA временно не получают сообщений
   → Другие SA продолжают работать
```

## Сервисные аккаунты всегда активны

SA **всегда** подписаны на события TG для своих групп. Даже если ни один пользователь не мониторит группу через этого SA, он продолжает получать сообщения. Это design decision — SA не выполняют LeaveChannel при MonitoringStop. Сообщения просто пропускаются (не мониторится → не отправляем событие).

## Безопасность

* `.session` файлы SA должны быть pre-provisioned в Docker volume
* Номера телефонов SA хранятся в env (не логируются)
* SA не должны использоваться для других целей
* Каждый SA — отдельный Telegram-аккаунт

## Связанные документы

* [Sessions Overview](overview.md)
* [Client Pool](client-pool.md)
* [Internal Models — ServiceAccountInfo](../models/internal.md#serviceaccountinfo)
* [Gateway Infrastructure](../infrastructure/overview.md)
* [Join/Leave Flow](../groups/join-leave-flow.md)
