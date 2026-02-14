# Auth Handler

## Описание

`AuthHandler` — центральный обработчик команд авторизации. Содержит три метода, по одному на каждую auth-команду.

## Файл

`src/auth/handler.py`

## Зависимости

| Компонент | Использование |
|-----------|--------------|
| `ClientPool` | Создание и получение TelegramClient |
| `SessionManager` | Генерация externalSessionId, управление .session файлами |
| `Publisher` | Публикация событий в RabbitMQ |
| `AuthState` storage | In-memory dict для хранения промежуточного состояния |

## Интерфейс

```python
class AuthHandler:
    def __init__(
        self,
        client_pool: ClientPool,
        session_manager: SessionManager,
        publisher: Publisher,
    ): ...

    async def handle_request_auth_code(self, command: RequestAuthCodeCommand) -> None: ...
    async def handle_submit_auth_code(self, command: SubmitAuthCodeCommand) -> None: ...
    async def handle_submit_2fa(self, command: Submit2FACommand) -> None: ...
```

## handle_request_auth_code

**Команда**: `InitAuth`
**Вход**: `RequestAuthCodeCommand`
**Выход**: `AuthCodeSent` или `SessionFailed`

### Pseudocode

```python
async def handle_request_auth_code(self, command: RequestAuthCodeCommand) -> None:
    session_id = command.session_id
    phone = command.auth_data.phone_number

    # Если уже есть auth flow для этого sessionId — очищаем предыдущий
    if session_id in self._auth_states:
        old_state = self._auth_states.pop(session_id)
        await self._cleanup_client(old_state.external_session_id)

    # Генерация нового externalSessionId
    external_session_id = self._session_manager.create_session()

    # Создание и подключение клиента
    client = self._client_pool.create_client(external_session_id)
    try:
        await client.connect()
        result = await client.send_code_request(phone)
    except Exception as e:
        reason, message, can_retry = map_auth_error(e)
        await self._publisher.publish(
            SessionFailedEvent(session_id=session_id, reason=reason, message=message,
                             can_retry=can_retry, correlation_id=command.correlation_id),
            routing_key="telegram.auth.session_failed",
        )
        self._session_manager.delete_session(external_session_id)
        await client.disconnect()
        return

    # Сохранение состояния
    self._auth_states[session_id] = AuthState(
        session_id=session_id,
        external_session_id=external_session_id,
        phone_number=phone,
        phone_code_hash=result.phone_code_hash,
        state=AuthFlowState.AWAITING_CODE,
        correlation_id=command.correlation_id,
        created_at=datetime.now(UTC),
    )

    # Добавление в пул
    await self._client_pool.add(ActiveClient(
        external_session_id=external_session_id,
        session_id=session_id,
        client=client,
        is_service_account=False,
    ))

    # Событие
    await self._publisher.publish(
        AuthCodeRequestedEvent(
            session_id=session_id,
            code_length=result.type.length,
            correlation_id=command.correlation_id,
        ),
        routing_key="telegram.auth.code_requested",
    )
```

## handle_submit_auth_code

**Команда**: `SubmitCode`
**Вход**: `SubmitAuthCodeCommand`
**Выход**: `AuthSuccess`, `Auth2FARequired`, или `SessionFailed`

### Pseudocode

```python
async def handle_submit_auth_code(self, command: SubmitAuthCodeCommand) -> None:
    auth_state = self._get_auth_state(command.session_id, AuthFlowState.AWAITING_CODE)
    if not auth_state:
        return  # SessionFailed уже отправлен в _get_auth_state

    active_client = self._client_pool.get(auth_state.external_session_id)

    try:
        await active_client.client.sign_in(
            phone=auth_state.phone_number,
            code=command.code,
            phone_code_hash=auth_state.phone_code_hash,
        )
        await self._complete_auth(auth_state, command.correlation_id)

    except SessionPasswordNeededError:
        password_info = await active_client.client(GetPasswordRequest())
        auth_state.state = AuthFlowState.AWAITING_2FA
        auth_state.correlation_id = command.correlation_id

        await self._publisher.publish(
            Auth2FARequiredEvent(
                session_id=command.session_id,
                hint=password_info.hint or "",
                correlation_id=command.correlation_id,
            ),
            routing_key="telegram.auth.2fa_required",
        )

    except Exception as e:
        await self._handle_auth_error(e, auth_state, command.correlation_id)
```

## handle_submit_2fa

**Команда**: `Submit2FA`
**Вход**: `Submit2FACommand`
**Выход**: `AuthSuccess` или `SessionFailed`

### Pseudocode

```python
async def handle_submit_2fa(self, command: Submit2FACommand) -> None:
    auth_state = self._get_auth_state(command.session_id, AuthFlowState.AWAITING_2FA)
    if not auth_state:
        return

    active_client = self._client_pool.get(auth_state.external_session_id)

    try:
        await active_client.client.sign_in(password=command.password)
        await self._complete_auth(auth_state, command.correlation_id)

    except Exception as e:
        await self._handle_auth_error(e, auth_state, command.correlation_id)
```

## Вспомогательные методы

```python
async def _complete_auth(self, auth_state: AuthState, correlation_id: str) -> None:
    """Завершение успешной авторизации."""
    active_client = self._client_pool.get(auth_state.external_session_id)
    me = await active_client.client.get_me()
    display_name = f"{me.first_name} {me.last_name or ''}".strip()

    del self._auth_states[auth_state.session_id]

    await self._publisher.publish(
        SessionAuthorizedEvent(
            session_id=auth_state.session_id,
            external_session_id=auth_state.external_session_id,
            display_name=display_name,
            correlation_id=correlation_id,
        ),
        routing_key="telegram.auth.session_authorized",
    )

async def _handle_auth_error(self, exc: Exception, auth_state: AuthState, correlation_id: str) -> None:
    """Обработка ошибки авторизации."""
    reason, message, can_retry = map_auth_error(exc)

    await self._publisher.publish(
        SessionFailedEvent(
            session_id=auth_state.session_id,
            reason=reason, message=message, can_retry=can_retry,
            correlation_id=correlation_id,
        ),
        routing_key="telegram.auth.session_failed",
    )

    if not can_retry:
        del self._auth_states[auth_state.session_id]
        await self._cleanup_client(auth_state.external_session_id)

async def _cleanup_client(self, external_session_id: str) -> None:
    """Очистка клиента при неудачной авторизации."""
    await self._client_pool.remove(external_session_id)
    self._session_manager.delete_session(external_session_id)

def _get_auth_state(self, session_id: str, expected_state: AuthFlowState) -> AuthState | None:
    """Получение и валидация AuthState."""
    auth_state = self._auth_states.get(session_id)
    if not auth_state:
        # Нет auth flow — возможно gateway перезапущен
        self._publisher.publish(SessionFailedEvent(
            session_id=session_id, reason="INVALID_STATE",
            message="Auth flow not found", can_retry=False,
        ))
        return None
    if auth_state.state != expected_state:
        self._publisher.publish(SessionFailedEvent(
            session_id=session_id, reason="INVALID_STATE",
            message=f"Expected {expected_state}, got {auth_state.state}", can_retry=False,
        ))
        return None
    return auth_state
```

## Связанные документы

* [Auth Overview](overview.md)
* [Auth Flow](auth-flow.md)
* [Auth State](auth-state.md)
* [Session Manager](session-manager.md)
* [Error Handling](error-handling.md)
