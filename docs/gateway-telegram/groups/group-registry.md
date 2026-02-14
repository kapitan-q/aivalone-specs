# Monitoring Registry

## Описание

`MonitoringRegistry` — in-memory реестр активного мониторинга групп. Ключевой принцип: **один Telethon listener на один chat_id** — несколько пользователей (MonitoringEntry) используют один event handler.

> Заменяет прежний `GroupRegistry` с его 1:1 маппингом group_id → GroupSubscription.

## Файл

`src/groups/monitoring_registry.py`

## Концепция

```
                   MonitoringRegistry
                          │
            ┌─────────────┼─────────────┐
            ▼             ▼             ▼
      ActiveListener  ActiveListener  ActiveListener
      chat_id: -100A  chat_id: -100B  chat_id: -100C
      session: sa_1   session: sa_2   session: user_x
      is_public: T    is_public: T    is_public: F
            │             │             │
        ┌───┴───┐     ┌──┘         ┌───┴───┐
        ▼       ▼     ▼            ▼       ▼
      Entry   Entry  Entry       Entry   Entry
      grp:1   grp:2  grp:3      grp:4   grp:5
```

## API

```python
class MonitoringRegistry:
    # --- Начало/остановка мониторинга ---
    def start_monitoring(
        self, chat_id: int, external_group_id: str,
        group_id: str, requesting_session_id: str | None,
        is_public: bool,
    ) -> bool:
        """
        Добавить MonitoringEntry для группы.
        Возвращает True если нужно создать новый listener (первый entry для этого chat_id).
        Возвращает False если listener уже существует (просто добавлен entry).
        """

    def stop_monitoring(self, group_id: str) -> StopResult:
        """
        Удалить MonitoringEntry для группы.
        Возвращает StopResult с информацией о том, нужно ли удалять listener.
        """

    # --- Управление listener ---
    def set_listener(
        self, chat_id: int, listener_handle: object, listener_session_id: str,
    ) -> None:
        """Установить/обновить Telethon event handler для chat_id."""

    def remove_listener(self, chat_id: int) -> object | None:
        """Удалить listener. Возвращает handle для client.remove_event_handler()."""

    # --- Поиск ---
    def get_listener(self, chat_id: int) -> ActiveListener | None:
        """Найти ActiveListener по chat_id."""

    def get_by_group_id(self, group_id: str) -> ActiveListener | None:
        """Найти ActiveListener по backend group_id."""

    def is_monitored(self, chat_id: int) -> bool:
        """Мониторится ли группа хоть кем-то."""

    def get_entries_for_chat(self, chat_id: int) -> list[MonitoringEntry]:
        """Все MonitoringEntry для chat_id."""

    # --- Cleanup ---
    def cleanup_session(self, session_id: str) -> CleanupResult:
        """
        Удалить все MonitoringEntry этой сессии.
        Возвращает CleanupResult: список chat_ids где нужен fallback + список где listener удалён.
        """

    def get_chats_by_listener_session(self, session_id: str) -> list[int]:
        """Все chat_ids где эта сессия является listener-ом."""

    def get_all_listeners(self) -> list[ActiveListener]:
        """Все активные listeners."""

    def count_entries(self) -> int:
        """Общее количество MonitoringEntry."""

    def count_listeners(self) -> int:
        """Количество ActiveListener (уникальных chat_id)."""
```

## Хранение

```python
class MonitoringRegistry:
    def __init__(self):
        # Основной индекс: chat_id → ActiveListener
        self._by_chat_id: dict[int, ActiveListener] = {}

        # Вспомогательные индексы
        self._group_to_chat: dict[str, int] = {}     # group_id → chat_id
        self._session_entries: dict[str, set[str]] = {}  # session_id → set[group_id]
        self._listener_sessions: dict[str, set[int]] = {}  # session_id → set[chat_id] (где эта сессия — listener)
```

## Pseudocode

### start_monitoring

```python
def start_monitoring(
    self, chat_id: int, external_group_id: str,
    group_id: str, requesting_session_id: str | None,
    is_public: bool,
) -> bool:
    needs_new_listener = False
    entry = MonitoringEntry(
        group_id=group_id,
        requesting_session_id=requesting_session_id,
        is_public=is_public,
        added_at=now(),
    )

    if chat_id not in self._by_chat_id:
        # Первая подписка — нужен новый listener
        self._by_chat_id[chat_id] = ActiveListener(
            chat_id=chat_id,
            external_group_id=external_group_id,
            listener_session_id="",  # будет установлен через set_listener()
            listener_handle=None,
            is_public=is_public,
            monitoring_entries=[entry],
            created_at=now(),
        )
        needs_new_listener = True
    else:
        # Listener уже есть — просто добавляем entry
        self._by_chat_id[chat_id].monitoring_entries.append(entry)

    # Обновить индексы
    self._group_to_chat[group_id] = chat_id
    if requesting_session_id:
        self._session_entries.setdefault(requesting_session_id, set()).add(group_id)

    return needs_new_listener
```

### stop_monitoring

```python
@dataclass
class StopResult:
    removed: bool                    # entry удалён?
    entries_remaining: int           # сколько entries осталось в listener
    listener_should_be_removed: bool # нужно ли удалять Telethon listener?
    chat_id: int | None
    is_public: bool

def stop_monitoring(self, group_id: str) -> StopResult:
    chat_id = self._group_to_chat.pop(group_id, None)
    if not chat_id or chat_id not in self._by_chat_id:
        return StopResult(removed=False, entries_remaining=0,
                          listener_should_be_removed=False, chat_id=None, is_public=False)

    listener = self._by_chat_id[chat_id]

    # Удалить entry
    listener.monitoring_entries = [
        e for e in listener.monitoring_entries if e.group_id != group_id
    ]

    # Обновить индексы сессий
    for session_id, group_ids in self._session_entries.items():
        group_ids.discard(group_id)

    remaining = len(listener.monitoring_entries)

    if remaining == 0:
        # Больше никто не мониторит — нужно решить что делать с listener
        if listener.is_public:
            # Публичная: SA продолжает получать сообщения (не удаляем listener)
            # Но удаляем ActiveListener из registry
            del self._by_chat_id[chat_id]
            self._remove_listener_session_index(listener.listener_session_id, chat_id)
            return StopResult(removed=True, entries_remaining=0,
                              listener_should_be_removed=False,
                              chat_id=chat_id, is_public=True)
        else:
            # Приватная: нужно отписаться от событий TG
            del self._by_chat_id[chat_id]
            self._remove_listener_session_index(listener.listener_session_id, chat_id)
            return StopResult(removed=True, entries_remaining=0,
                              listener_should_be_removed=True,
                              chat_id=chat_id, is_public=False)

    return StopResult(removed=True, entries_remaining=remaining,
                      listener_should_be_removed=False,
                      chat_id=chat_id, is_public=listener.is_public)
```

### cleanup_session

```python
@dataclass
class CleanupResult:
    removed_entries: list[str]        # group_ids удалённых entries
    needs_fallback: list[int]         # chat_ids где нужен fallback (сессия была listener-ом)
    listeners_removed: list[int]      # chat_ids где listener удалён (не осталось entries)

def cleanup_session(self, session_id: str) -> CleanupResult:
    result = CleanupResult([], [], [])

    # 1. Удалить все MonitoringEntry этой сессии
    group_ids = list(self._session_entries.pop(session_id, []))
    for group_id in group_ids:
        stop = self.stop_monitoring(group_id)
        if stop.removed:
            result.removed_entries.append(group_id)
            if stop.listener_should_be_removed:
                result.listeners_removed.append(stop.chat_id)

    # 2. Найти chat_ids где эта сессия — listener (но entries от других сессий остались)
    listener_chats = list(self._listener_sessions.pop(session_id, []))
    for chat_id in listener_chats:
        if chat_id in self._by_chat_id:
            # Listener ещё жив (есть entries от других сессий) → нужен fallback
            result.needs_fallback.append(chat_id)

    return result
```

## Инварианты

1. **Один ActiveListener на chat_id** — не может быть дублей
2. **group_id уникален** — одна группа не может мониториться дважды
3. **Пустой ActiveListener удаляется** — если entries == 0, ActiveListener удаляется (кроме публичных SA)
4. **listener_session_id** должна быть валидной сессией
5. При удалении MonitoringEntry → проверяем: нужен ли ещё listener

## Связанные документы

* [Groups Overview](overview.md)
* [Group Handler](group-handler.md)
* [Internal Models — ActiveListener, MonitoringEntry](../models/internal.md#activelistener)
* [Fallback](fallback.md)
