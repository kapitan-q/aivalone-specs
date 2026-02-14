# Сканирование групп пользователя

## Описание

После успешной авторизации gateway получает список групп и каналов, в которых состоит пользователь. Этот кэш используется для:

1. **Проверки доступа** при MonitoringStart (приватная группа) — быстрая проверка без API-вызова
2. **Выбора fallback-сессии** — при обрыве listener-сессии нужно знать, у кого есть доступ к группе
3. **Валидации** — если группы нет в кэше, дополнительно проверяем через `GetParticipantRequest`

## Файл

`src/auth/user_groups.py`

## Когда выполняется сканирование

| Момент | Действие | Обязательность |
|--------|---------|---------------|
| После `_complete_auth()` | Полный `get_dialogs()` | Best-effort |
| При `StartSession` (reconnect) | Полный `get_dialogs()` | Best-effort |
| При `MonitoringStart` (приватная) | `get_dialogs()` обновление кэша | Best-effort |

> **Best-effort**: если `get_dialogs()` не удался (timeout, FloodWait) — используем старый кэш. Если кэша нет — проверяем конкретную группу через `GetParticipantRequest`.

## Pseudocode

### Сканирование при авторизации

```python
async def scan_user_groups(active_client: ActiveClient) -> UserGroupCache:
    """Получить список групп/каналов пользователя через Telethon."""
    try:
        dialogs = await active_client.client.get_dialogs()

        chat_ids: set[int] = set()
        for dialog in dialogs:
            # Только группы и каналы (не личные чаты, не боты)
            if dialog.is_group or dialog.is_channel:
                chat_ids.add(dialog.id)

        cache = UserGroupCache(
            session_id=active_client.external_session_id,
            chat_ids=chat_ids,
            last_updated=now(),
        )
        active_client.user_groups = cache

        logger.info(
            "user_groups_scanned",
            session_id=active_client.external_session_id,
            groups_count=len(chat_ids),
        )
        return cache

    except FloodWaitError as e:
        logger.warning(
            "user_groups_scan_flood",
            wait_seconds=e.seconds,
            session_id=active_client.external_session_id,
        )
        # Не блокируем авторизацию — кэш будет пустым
        return active_client.user_groups  # может быть None

    except Exception as e:
        logger.warning(
            "user_groups_scan_failed",
            error=type(e).__name__,
            session_id=active_client.external_session_id,
        )
        return active_client.user_groups
```

### Обновление при MonitoringStart

```python
async def ensure_user_has_group(
    active_client: ActiveClient, chat_id: int
) -> bool:
    """Проверить что пользователь имеет доступ к группе. Обновляет кэш."""

    # 1. Быстрая проверка: есть ли в кэше
    if active_client.user_groups and chat_id in active_client.user_groups.chat_ids:
        return True

    # 2. Кэш устарел или пуст — обновляем
    cache = await scan_user_groups(active_client)
    if cache and chat_id in cache.chat_ids:
        return True

    # 3. Нет в обновлённом кэше — точечная проверка через API
    try:
        entity = await active_client.client.get_entity(chat_id)
        await active_client.client(GetParticipantRequest(entity, 'me'))
        # Есть доступ — добавляем в кэш
        if active_client.user_groups:
            active_client.user_groups.chat_ids.add(chat_id)
        return True
    except (UserNotParticipantError, ChannelPrivateError):
        return False
    except Exception:
        # Неизвестная ошибка — считаем что нет доступа
        return False
```

### Интеграция с auth flow

```python
# В AuthHandler._complete_auth():
async def _complete_auth(self, auth_state: AuthState, active_client: ActiveClient):
    me = await active_client.client.get_me()
    display_name = format_display_name(me)

    # Удалить AuthState
    del self._auth_states[auth_state.session_id]

    # Сканировать группы (best-effort, не блокирует авторизацию)
    await scan_user_groups(active_client)

    # Опубликовать событие
    await self._publisher.publish(
        SessionAuthorizedEvent(
            session_id=auth_state.session_id,
            external_session_id=auth_state.external_session_id,
            display_name=display_name,
            correlation_id=auth_state.correlation_id,
        ),
        routing_key="telegram.auth.session_authorized",
    )
```

## Модель UserGroupCache

Определена в [models/internal.md](../models/internal.md#usergroupcache).

```python
@dataclass
class UserGroupCache:
    session_id: str         # external_session_id
    chat_ids: set[int]      # числовые Telegram chat IDs
    last_updated: datetime
```

## Производительность

### get_dialogs()

| Количество групп | Примерное время |
|-------------------|----------------|
| < 50 | < 1 сек |
| 50–200 | 1–3 сек |
| 200–500 | 3–10 сек |
| > 500 | 10+ сек |

> `get_dialogs()` загружает все диалоги пользователя (включая личные чаты). Для аккаунтов с большим числом диалогов может быть медленным. Поэтому сканирование — **best-effort** и **не блокирует** основной flow.

### Оптимизации

1. **Фоновое сканирование**: `scan_user_groups()` можно запускать как `asyncio.create_task()` после авторизации
2. **Кэш с TTL**: не обновлять кэш чаще чем раз в 5 минут
3. **Инкрементальное обновление**: при MonitoringStart проверять конкретную группу через `GetParticipantRequest` вместо полного `get_dialogs()`

## Безопасность

* Список групп пользователя — **чувствительная информация**, не логируется
* Логируется только количество: `groups_count=42`
* Кэш хранится только в памяти, теряется при перезапуске

## Связанные документы

* [Auth Handler](auth-handler.md)
* [Auth Flow](auth-flow.md)
* [Internal Models — UserGroupCache](../models/internal.md#usergroupcache)
* [Fallback](../groups/fallback.md)
* [Join/Leave Flow](../groups/join-leave-flow.md)
