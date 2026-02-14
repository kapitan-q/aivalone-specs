# Auth — Обзор модуля

## Назначение

Модуль `auth` реализует полный flow авторизации пользовательских Telegram-сессий: от запроса кода до успешной авторизации или ошибки. Выступает как **auth-service** для мессенджера Telegram.

## Зона ответственности

### Отвечает за:

* Обработку команд `RequestAuthCode`, `SubmitAuthCode`, `Submit2FA`
* Взаимодействие с Telethon API для авторизации (`send_code_request`, `sign_in`)
* Хранение промежуточного состояния auth flow (`AuthState`)
* Генерацию `externalSessionId` (UUID4)
* Создание `.session` файлов для новых сессий
* Маппинг ошибок Telethon → события `SessionFailed`
* Очистку ресурсов при неудачной авторизации

### НЕ отвечает за:

* Запуск/остановку уже авторизованных сессий (модуль `sessions`)
* Управление группами (модуль `groups`)
* Хранение auth state между перезапусками gateway (ephemeral)

## Компоненты

| Компонент | Файл | Описание |
|-----------|------|----------|
| `AuthHandler` | `src/auth/handler.py` | Обработчики трёх auth-команд |
| `AuthState` | `src/auth/auth_state.py` | Промежуточное состояние auth flow |
| `SessionManager` | `src/auth/session_manager.py` | Создание/удаление .session файлов, генерация externalSessionId |

## Диаграмма взаимодействия

```
Backend        AuthHandler        SessionManager     ClientPool      Telegram
  │                │                    │                │               │
  │ RequestAuthCode│                    │                │               │
  │───────────────→│                    │                │               │
  │                │ create_session()   │                │               │
  │                │───────────────────→│                │               │
  │                │ externalSessionId  │                │               │
  │                │◄───────────────────│                │               │
  │                │                    │  create_client │               │
  │                │────────────────────┼───────────────→│               │
  │                │                    │                │  connect()    │
  │                │                    │                │──────────────→│
  │                │                    │                │  send_code()  │
  │                │                    │                │──────────────→│
  │                │ save AuthState     │                │               │
  │  AuthCodeSent  │                    │                │               │
  │◄───────────────│                    │                │               │
  │                │                    │                │               │
  │ SubmitAuthCode │                    │                │               │
  │───────────────→│                    │                │               │
  │                │ get AuthState      │                │               │
  │                │                    │ get(extSessId) │               │
  │                │────────────────────┼───────────────→│               │
  │                │                    │                │  sign_in()    │
  │                │                    │                │──────────────→│
  │                │                    │                │               │
  │  AuthSuccess   │ delete AuthState   │                │               │
  │◄───────────────│                    │                │               │
```

## Внутреннее состояние

| Данные | Хранение | TTL | Описание |
|--------|----------|-----|----------|
| `_auth_states` | In-memory dict | 15 мин | `Dict[sessionId, AuthState]` |

При перезапуске gateway все незавершённые auth flow теряются. Backend получит timeout и может повторить `RequestAuthCode`.

## Инварианты

1. **Один AuthState на sessionId** — повторный `RequestAuthCode` для того же sessionId перезаписывает предыдущий
2. **AuthState удаляется при завершении** — успех или терминальная ошибка
3. **При canRetry=true** — AuthState сохраняется для повторной попытки
4. **phone_number и phone_code_hash не логируются**
5. **При ошибке без canRetry** — client disconnect, .session удаляется

## Связанные документы

* [Auth Flow](auth-flow.md)
* [Auth Handler](auth-handler.md)
* [Auth State](auth-state.md)
* [Session Manager](session-manager.md)
* [Error Handling](error-handling.md)
* [Messaging Commands](../messaging/commands/request-auth-code.md)
