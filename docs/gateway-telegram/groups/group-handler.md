# Group Handler

## Описание

`GroupHandler` — обработчик команд `JoinGroup` (MonitoringStart) и `LeaveGroup` (MonitoringStop). Реализует логику дедупликации: один Telethon listener на один chat_id.

## Файл

`src/groups/handler.py`

## Зависимости

| Компонент | Использование |
|-----------|--------------|
| `ClientPool` | Получение пользовательских TelegramClient |
| `ServiceAccountPool` | Получение SA для публичных групп |
| `MonitoringRegistry` | Управление подписками и listeners |
| `MessageListener` | Создание Telethon event handlers |
| `Publisher` | Публикация событий |
| `FallbackHandler` | Обработка disconnect |

## Интерфейс

```python
class GroupHandler:
    def __init__(
        self,
        client_pool: ClientPool,
        sa_pool: ServiceAccountPool,
        monitoring_registry: MonitoringRegistry,
        message_listener: MessageListener,
        publisher: Publisher,
        fallback_handler: FallbackHandler,
    ): ...

    async def handle_join_group(self, command: JoinGroupCommand) -> None: ...
    async def handle_leave_group(self, command: LeaveGroupCommand) -> None: ...
```

## handle_join_group (MonitoringStart)

```python
async def handle_join_group(self, command: JoinGroupCommand) -> None:
    is_public = command.session_id is None

    if is_public:
        await self._handle_public_monitoring_start(command)
    else:
        await self._handle_private_monitoring_start(command)
```

### Приватная группа

```python
async def _handle_private_monitoring_start(self, command: JoinGroupCommand) -> None:
    # 1. Проверить сессию
    active_client = self._client_pool.get(command.session_id)
    if not active_client:
        await self._publish_join_failed(
            command, reason="SESSION_NOT_FOUND",
            message="Сессия не найдена в gateway",
        )
        return

    # 2. Проверить подключение
    if not await active_client.client.is_user_authorized():
        await self._publish_join_failed(
            command, reason="SESSION_NOT_AUTHORIZED",
            message="Сессия не авторизована",
        )
        return

    try:
        chat_id = int(command.external_group_id)

        # 3. Проверить доступ к группе (обновляет кэш)
        has_access = await ensure_user_has_group(active_client, chat_id)
        if not has_access:
            await self._publish_join_failed(
                command, reason="NOT_A_MEMBER",
                message="Пользователь не является участником приватной группы",
            )
            return

        # 4. Проверить: уже мониторится?
        existing = self._monitoring_registry.get_listener(chat_id)

        if existing:
            # Группа уже слушается через другую сессию — добавляем запись
            self._monitoring_registry.start_monitoring(
                chat_id=chat_id,
                external_group_id=command.external_group_id,
                group_id=command.group_id,
                requesting_session_id=command.session_id,
                is_public=False,
            )

            # Публикуем GroupJoined с информацией из существующего listener
            await self._publish_group_joined(
                command,
                group_title="",  # TODO: кэшировать title в ActiveListener
                member_count=0,
            )
            return

        # 5. Нужен новый listener
        self._monitoring_registry.start_monitoring(
            chat_id=chat_id,
            external_group_id=command.external_group_id,
            group_id=command.group_id,
            requesting_session_id=command.session_id,
            is_public=False,
        )

        entity = await active_client.client.get_entity(chat_id)

        handler = self._message_listener.register(
            active_client.client, chat_id
        )
        self._monitoring_registry.set_listener(
            chat_id, handler, command.session_id
        )

        # 6. Информация о группе
        full = await active_client.client(GetFullChannelRequest(entity))
        group_title = full.chats[0].title
        member_count = full.full_chat.participants_count

        await self._publish_group_joined(command, group_title, member_count)

    except Exception as e:
        reason, message = map_group_error(e)
        await self._publish_join_failed(command, reason=reason, message=message)
```

### Публичная группа

```python
async def _handle_public_monitoring_start(self, command: JoinGroupCommand) -> None:
    try:
        # 1. Resolve chat_id из external_group_id (@username → числовой ID)
        # Для публичных: сначала проверяем listener, потом resolve
        # Если listener уже есть — используем его chat_id

        existing = self._find_listener_by_external_id(command.external_group_id)

        if existing:
            # Уже мониторится — добавляем запись
            self._monitoring_registry.start_monitoring(
                chat_id=existing.chat_id,
                external_group_id=command.external_group_id,
                group_id=command.group_id,
                requesting_session_id=None,
                is_public=True,
            )
            await self._publish_group_joined(command, group_title="", member_count=0)
            return

        # 2. Выбрать SA
        sa = self._sa_pool.select_sa_for_group(
            self._resolve_chat_id_from_cache(command.external_group_id)
        )

        # 3. Resolve и join
        entity = await sa.client.get_entity(command.external_group_id)
        chat_id = entity.id

        # Проверить: SA уже в группе?
        if chat_id not in sa.joined_groups:
            try:
                await sa.client(JoinChannelRequest(entity))
            except UserAlreadyParticipantError:
                pass
            self._sa_pool.add_group(sa.external_session_id, chat_id)

        # 4. Подписаться
        self._monitoring_registry.start_monitoring(
            chat_id=chat_id,
            external_group_id=command.external_group_id,
            group_id=command.group_id,
            requesting_session_id=None,
            is_public=True,
        )

        handler = self._message_listener.register(sa.client, chat_id)
        self._monitoring_registry.set_listener(
            chat_id, handler, sa.external_session_id
        )

        # 5. Информация
        full = await sa.client(GetFullChannelRequest(entity))
        group_title = full.chats[0].title
        member_count = full.full_chat.participants_count

        await self._publish_group_joined(command, group_title, member_count)

    except Exception as e:
        reason, message = map_group_error(e)
        await self._publish_join_failed(command, reason=reason, message=message)
```

## handle_leave_group (MonitoringStop)

```python
async def handle_leave_group(self, command: LeaveGroupCommand) -> None:
    listener = self._monitoring_registry.get_by_group_id(command.group_id)
    if not listener:
        logger.warning("not_monitored", group_id=command.group_id)
        return

    result = self._monitoring_registry.stop_monitoring(command.group_id)

    if not result.removed:
        return

    if result.entries_remaining > 0:
        # Другие пользователи мониторят — ничего не делаем
        logger.info(
            "monitoring_stopped_entry_only",
            group_id=command.group_id,
            remaining=result.entries_remaining,
        )
        return

    # Больше никто не мониторит
    if result.is_public:
        # SA остаётся в группе, просто не отправляем события
        logger.info("monitoring_stopped_public", group_id=command.group_id)
    else:
        # Приватная: снять Telethon event handler
        active_client = self._client_pool.get(listener.listener_session_id)
        if active_client and listener.listener_handle:
            try:
                active_client.client.remove_event_handler(listener.listener_handle)
            except Exception:
                pass
        logger.info("monitoring_stopped_private", group_id=command.group_id)
```

## Вспомогательные методы

```python
async def _publish_group_joined(self, command, group_title, member_count):
    await self._publisher.publish(
        GroupJoinedEvent(
            group_id=command.group_id,
            external_group_id=command.external_group_id,
            group_title=group_title,
            member_count=member_count,
            correlation_id=command.correlation_id,
        ),
        routing_key="telegram.group.joined",
    )

async def _publish_join_failed(self, command, reason, message):
    await self._publisher.publish(
        GroupJoinFailedEvent(
            group_id=command.group_id,
            external_group_id=command.external_group_id,
            reason=reason,
            message=message,
            correlation_id=command.correlation_id,
        ),
        routing_key="telegram.group.join_failed",
    )
```

## Связанные документы

* [Groups Overview](overview.md)
* [Join/Leave Flow](join-leave-flow.md)
* [Monitoring Registry](group-registry.md)
* [Message Listener](message-listener.md)
* [Fallback](fallback.md)
* [Service Account Pool](../sessions/service-account.md)
* [User Groups](../auth/user-groups.md)
* [Error Handling](error-handling.md)
