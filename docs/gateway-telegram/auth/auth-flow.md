# Auth Flow

## Описание

Полный процесс авторизации пользователя в Telegram через Gateway. Многошаговый flow с ветвлением на 2FA.

## Sequence Diagram

```
User        Bot       Backend           Gateway              Telegram
 │           │          │                  │                     │
 │ "Войти"   │          │                  │                     │
 │──────────→│          │                  │                     │
 │           │ StartAuth│                  │                     │
 │           │─────────→│                  │                     │
 │           │          │ InitAuth(phone)  │                     │
 │           │          │─────────────────→│                     │
 │           │          │                  │ send_code_request() │
 │           │          │                  │────────────────────→│
 │           │          │                  │   SentCode          │
 │           │          │                  │◄────────────────────│
 │           │          │ AuthCodeSent     │                     │
 │           │          │◄─────────────────│                     │
 │           │ "Код?"   │                  │                     │
 │           │◄─────────│                  │                     │
 │  "12345"  │          │                  │                     │
 │──────────→│          │                  │                     │
 │           │ SubmitCode(12345)           │                     │
 │           │─────────→│                  │                     │
 │           │          │ SubmitCode       │                     │
 │           │          │─────────────────→│                     │
 │           │          │                  │ sign_in(code)       │
 │           │          │                  │────────────────────→│
 │           │          │                  │                     │
 │           │          │        ┌─────────┼─────────────────────┤
 │           │          │        │ Вариант A: Без 2FA            │
 │           │          │        │         │  User authorized    │
 │           │          │        │         │◄────────────────────│
 │           │          │ AuthSuccess      │                     │
 │           │          │◄─────────────────│                     │
 │           │ "Готово!" │                 │                     │
 │           │◄─────────│                  │                     │
 │           │          │        │         │                     │
 │           │          │        ├─────────┼─────────────────────┤
 │           │          │        │ Вариант B: С 2FA              │
 │           │          │        │         │ PasswordNeeded      │
 │           │          │        │         │◄────────────────────│
 │           │          │ Auth2FARequired  │                     │
 │           │          │◄─────────────────│                     │
 │           │ "2FA?"   │                  │                     │
 │           │◄─────────│                  │                     │
 │ "password"│          │                  │                     │
 │──────────→│          │                  │                     │
 │           │ Submit2FA│                  │                     │
 │           │─────────→│                  │                     │
 │           │          │ Submit2FA        │                     │
 │           │          │─────────────────→│                     │
 │           │          │                  │ sign_in(password)   │
 │           │          │                  │────────────────────→│
 │           │          │                  │  User authorized    │
 │           │          │                  │◄────────────────────│
 │           │          │ AuthSuccess      │                     │
 │           │          │◄─────────────────│                     │
 │           │ "Готово!" │                 │                     │
 │           │◄─────────│                  │                     │
 │           │          │        │         │                     │
 │           │          │        ├─────────┼─────────────────────┤
 │           │          │        │ Вариант C: Ошибка             │
 │           │          │        │         │ PhoneCodeInvalid    │
 │           │          │        │         │◄────────────────────│
 │           │          │ AuthFailed       │                     │
 │           │          │ (canRetry=true)  │                     │
 │           │          │◄─────────────────│                     │
 │           │ "Неверный│код"              │                     │
 │           │◄─────────│                  │                     │
 │           │          │        └─────────┼─────────────────────┘
```

## Шаги flow

### Шаг 1: RequestAuthCode

**Вход**: `InitAuth { sessionId, authData.phoneNumber, correlationId }`

```python
async def handle_request_auth_code(command):
    # 1. Генерация externalSessionId
    external_session_id = session_manager.create_session()
    # → UUID4, создаёт пустой .session файл

    # 2. Создание клиента
    client = client_pool.create_client(external_session_id)
    await client.connect()

    # 3. Отправка кода
    try:
        result = await client.send_code_request(command.auth_data.phone_number)
    except Exception as e:
        reason, message, can_retry = map_auth_error(e)
        await publish(SessionFailed(reason=reason, message=message, canRetry=can_retry))
        session_manager.delete_session(external_session_id)
        return

    # 4. Сохранение состояния
    auth_state = AuthState(
        session_id=command.session_id,
        external_session_id=external_session_id,
        phone_number=command.auth_data.phone_number,
        phone_code_hash=result.phone_code_hash,
        state=AuthFlowState.AWAITING_CODE,
        correlation_id=command.correlation_id,
    )
    _auth_states[command.session_id] = auth_state

    # 5. Клиент в пул (пока без session_id mapping — ещё не авторизован полностью)
    client_pool.add(ActiveClient(
        external_session_id=external_session_id,
        session_id=command.session_id,
        client=client,
        is_service_account=False,
    ))

    # 6. Событие
    await publish(AuthCodeSent(
        session_id=command.session_id,
        code_length=result.type.length,
        correlation_id=command.correlation_id,
    ))
```

### Шаг 2: SubmitAuthCode

**Вход**: `SubmitCode { sessionId, code, correlationId }`

```python
async def handle_submit_auth_code(command):
    # 1. Найти состояние
    auth_state = _auth_states.get(command.session_id)
    if not auth_state or auth_state.state != AuthFlowState.AWAITING_CODE:
        await publish(SessionFailed(reason="INVALID_STATE", canRetry=False))
        return

    # 2. Получить клиент
    active_client = client_pool.get(auth_state.external_session_id)

    # 3. Попытка входа
    try:
        result = await active_client.client.sign_in(
            phone=auth_state.phone_number,
            code=command.code,
            phone_code_hash=auth_state.phone_code_hash,
        )
        # Успех
        me = await active_client.client.get_me()
        display_name = f"{me.first_name} {me.last_name or ''}".strip()

        del _auth_states[command.session_id]
        await publish(AuthSuccess(
            session_id=command.session_id,
            external_session_id=auth_state.external_session_id,
            display_name=display_name,
            correlation_id=command.correlation_id,
        ))

    except SessionPasswordNeededError:
        # 2FA required
        password_info = await active_client.client(GetPasswordRequest())
        auth_state.state = AuthFlowState.AWAITING_2FA
        auth_state.correlation_id = command.correlation_id

        await publish(Auth2FARequired(
            session_id=command.session_id,
            hint=password_info.hint or "",
            correlation_id=command.correlation_id,
        ))

    except Exception as e:
        reason, message, can_retry = map_auth_error(e)
        await publish(SessionFailed(
            session_id=command.session_id,
            reason=reason, message=message, canRetry=can_retry,
            correlation_id=command.correlation_id,
        ))
        if not can_retry:
            del _auth_states[command.session_id]
            await cleanup_client(auth_state.external_session_id)
```

### Шаг 3: Submit2FA (опционально)

**Вход**: `Submit2FA { sessionId, password, correlationId }`

```python
async def handle_submit_2fa(command):
    # 1. Найти состояние
    auth_state = _auth_states.get(command.session_id)
    if not auth_state or auth_state.state != AuthFlowState.AWAITING_2FA:
        await publish(SessionFailed(reason="INVALID_STATE", canRetry=False))
        return

    # 2. Получить клиент
    active_client = client_pool.get(auth_state.external_session_id)

    # 3. Попытка входа с паролем
    try:
        await active_client.client.sign_in(password=command.password)

        me = await active_client.client.get_me()
        display_name = f"{me.first_name} {me.last_name or ''}".strip()

        del _auth_states[command.session_id]
        await publish(AuthSuccess(
            session_id=command.session_id,
            external_session_id=auth_state.external_session_id,
            display_name=display_name,
            correlation_id=command.correlation_id,
        ))

    except PasswordHashInvalidError:
        await publish(SessionFailed(
            reason="PASSWORD_HASH_INVALID",
            message="Неверный пароль 2FA",
            canRetry=True,
            correlation_id=command.correlation_id,
        ))
        # AuthState сохраняется для повторной попытки

    except Exception as e:
        reason, message, can_retry = map_auth_error(e)
        await publish(SessionFailed(
            reason=reason, message=message, canRetry=can_retry,
            correlation_id=command.correlation_id,
        ))
        if not can_retry:
            del _auth_states[command.session_id]
            await cleanup_client(auth_state.external_session_id)
```

## Таймауты

| Этап | Таймаут | Действие при истечении |
|------|---------|----------------------|
| Весь auth flow | 15 мин | Удалить AuthState, disconnect client |
| Telegram API call | 30 сек | `asyncio.timeout`, SessionFailed (TIMEOUT) |

## Связанные документы

* [Auth Overview](overview.md)
* [Auth Handler](auth-handler.md)
* [Auth State](auth-state.md)
* [Error Handling](error-handling.md)
