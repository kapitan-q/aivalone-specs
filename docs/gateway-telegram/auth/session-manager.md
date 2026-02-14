# Session Manager

## Описание

`SessionManager` отвечает за создание и удаление `.session` файлов Telethon, а также за генерацию `externalSessionId`.

## Файл

`src/auth/session_manager.py`

## API

```python
class SessionManager:
    def __init__(self, session_dir: Path): ...

    def create_session(self) -> str:
        """Генерирует externalSessionId (UUID4). Файл создаётся Telethon при connect."""

    def delete_session(self, external_session_id: str) -> None:
        """Удаляет .session файл."""

    def session_exists(self, external_session_id: str) -> bool:
        """Проверяет существование .session файла."""

    def get_session_path(self, external_session_id: str) -> Path:
        """Возвращает путь к .session файлу."""
```

## Pseudocode

```python
class SessionManager:
    def __init__(self, session_dir: Path):
        self._session_dir = session_dir
        self._session_dir.mkdir(parents=True, exist_ok=True)

    def create_session(self) -> str:
        external_session_id = str(uuid4())
        logger.info("session_created", external_session_id=external_session_id)
        return external_session_id
        # .session файл физически создаётся Telethon при TelegramClient.connect()

    def delete_session(self, external_session_id: str) -> None:
        session_path = self._get_path(external_session_id)
        session_path.unlink(missing_ok=True)
        logger.info("session_deleted", external_session_id=external_session_id)

    def session_exists(self, external_session_id: str) -> bool:
        return self._get_path(external_session_id).exists()

    def get_session_path(self, external_session_id: str) -> Path:
        return self._session_dir / external_session_id
        # Telethon добавляет .session к имени автоматически

    def _get_path(self, external_session_id: str) -> Path:
        return self._session_dir / f"{external_session_id}.session"
```

## Именование файлов

| Тип | Пример externalSessionId | Файл |
|-----|--------------------------|------|
| Пользовательская сессия | `a1b2c3d4-e5f6-7890-abcd-ef1234567890` | `a1b2c3d4-e5f6-7890-abcd-ef1234567890.session` |
| Сервисный аккаунт | `service_account` | `service_account.session` |

> **Примечание**: Telethon при создании `TelegramClient("path/name")` автоматически добавляет `.session` к имени файла.

## Безопасность

* `SessionManager` не хранит чувствительные данные
* `.session` файлы содержат auth key — доступ только из процесса gateway
* При `delete_session` используется `missing_ok=True` для идемпотентности

## Связанные документы

* [Auth Overview](overview.md)
* [Auth Handler](auth-handler.md)
* [Session Lifecycle](../sessions/session-lifecycle.md)
