# Client Pool

## Описание

`ClientPool` — центральный компонент для управления экземплярами `TelegramClient`. Все модули (auth, groups, sessions) обращаются к пулу для получения клиентов.

## Файл

`src/telegram/client_pool.py`

## API

### Методы

| Метод | Сигнатура | Описание |
|-------|-----------|----------|
| `create_client` | `(session_name: str) → TelegramClient` | Создать новый TelegramClient (не подключённый) |
| `add` | `(client: ActiveClient) → None` | Добавить подключённый клиент в пул |
| `get` | `(external_session_id: str) → ActiveClient \| None` | Получить клиент по ID |
| `get_service_account` | `() → ActiveClient` | Получить клиент сервисного аккаунта |
| `remove` | `(external_session_id: str) → None` | Отключить и удалить клиент |
| `has` | `(external_session_id: str) → bool` | Проверить наличие клиента |
| `get_all` | `() → list[ActiveClient]` | Все активные клиенты |
| `get_by_session_id` | `(session_id: str) → ActiveClient \| None` | Найти по backend session ID |
| `disconnect_all` | `() → None` | Отключить все клиенты (graceful shutdown) |

### Pseudocode

```python
class ClientPool:
    def __init__(self, settings: Settings):
        self._clients: dict[str, ActiveClient] = {}
        self._service_account: ActiveClient | None = None
        self._settings = settings

    def create_client(self, session_name: str) -> TelegramClient:
        session_path = Path(self._settings.telegram_session_dir) / session_name
        return TelegramClient(
            str(session_path),
            api_id=self._settings.telegram_api_id,
            api_hash=self._settings.telegram_api_hash,
        )

    async def add(self, active_client: ActiveClient) -> None:
        if active_client.external_session_id in self._clients:
            raise ValueError(f"Client already exists: {active_client.external_session_id}")
        self._clients[active_client.external_session_id] = active_client

    def get(self, external_session_id: str) -> ActiveClient | None:
        return self._clients.get(external_session_id)

    def get_service_account(self) -> ActiveClient:
        if self._service_account is None:
            raise RuntimeError("Service account not initialized")
        return self._service_account

    async def remove(self, external_session_id: str) -> None:
        client = self._clients.pop(external_session_id, None)
        if client:
            await client.client.disconnect()

    async def disconnect_all(self) -> None:
        for key in list(self._clients.keys()):
            await self.remove(key)
        if self._service_account:
            await self._service_account.client.disconnect()
```

## ClientFactory

Встроен в `ClientPool.create_client()`. Параметры создания клиента:

| Параметр | Источник | Описание |
|----------|---------|----------|
| `session` | `{TELEGRAM_SESSION_DIR}/{session_name}` | Путь к .session файлу |
| `api_id` | `TELEGRAM_API_ID` | API ID приложения |
| `api_hash` | `TELEGRAM_API_HASH` | API Hash приложения |

## Жизненный цикл клиента в пуле

```
create_client(name) → TelegramClient (не подключён)
    │
    ▼
await client.connect() → Подключён к Telegram
    │
    ▼
add(ActiveClient) → В пуле
    │
    ├── get(id) → Используется модулями auth/groups
    │
    ▼
remove(id) → disconnect() + удалён из пула
```

## Disconnect callback

При добавлении клиента в пул регистрируется disconnect callback для отправки `SessionExpired` при неожиданном отключении:

```python
async def _on_disconnect(client: ActiveClient):
    if not client.is_service_account:
        await publisher.publish(
            SessionExpiredEvent(
                session_id=client.session_id,
                reason="connection_lost",
            ),
            routing_key="telegram.auth.session_expired",
        )
    await self.remove(client.external_session_id)
```

## Thread Safety

Gateway работает в одном asyncio event loop. `ClientPool` не требует блокировок — все операции выполняются последовательно в одном потоке.

## Связанные документы

* [Sessions Overview](overview.md)
* [Session Lifecycle](session-lifecycle.md)
* [Models — Internal (ActiveClient)](../models/internal.md#activeclient)
