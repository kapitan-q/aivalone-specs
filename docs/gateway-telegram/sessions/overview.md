# Sessions — Обзор модуля

## Назначение

Модуль `sessions` управляет жизненным циклом TelegramClient: создание, подключение, хранение в пуле, отключение, удаление `.session` файлов. Также управляет сервисным аккаунтом для прослушивания публичных групп.

## Зона ответственности

### Отвечает за:

* Создание и подключение `TelegramClient` экземпляров
* Пул активных клиентов (`ClientPool`)
* Запуск/остановка сессий по командам `StartSession`/`StopSession`
* Инициализация сервисного аккаунта при старте gateway
* Обнаружение отозванных/истёкших сессий → `SessionExpired`
* Graceful shutdown: корректное отключение всех клиентов

### НЕ отвечает за:

* Процесс авторизации (модуль `auth`)
* Управление группами (модуль `groups`)
* RabbitMQ коммуникацию (модуль `messaging`)

## Компоненты

| Компонент | Файл | Описание |
|-----------|------|----------|
| `SessionLifecycleHandler` | `src/sessions/handler.py` | Обработчики StartSession, StopSession |
| `ClientPool` | `src/telegram/client_pool.py` | Пул TelegramClient, CRUD операции |
| `ClientFactory` | `src/telegram/client_pool.py` | Создание TelegramClient с правильными параметрами |
| `ServiceAccountManager` | `src/sessions/service_account.py` | Управление сервисным аккаунтом |

## Диаграмма взаимодействия

```
Backend                    Sessions Module              Telegram
  │                              │                         │
  │  StartSession               │                         │
  │─────────────────────────────→│                         │
  │                              │  connect()              │
  │                              │────────────────────────→│
  │                              │  is_user_authorized()   │
  │                              │────────────────────────→│
  │                              │◄────────────────────────│
  │                              │  add to ClientPool      │
  │  AuthSuccess                │                         │
  │◄─────────────────────────────│                         │
  │                              │                         │
  │  StopSession                │                         │
  │─────────────────────────────→│                         │
  │                              │  log_out()              │
  │                              │────────────────────────→│
  │                              │  disconnect()           │
  │                              │────────────────────────→│
  │                              │  remove from ClientPool │
  │                              │  delete .session file   │
  │                              │                         │
```

## Внутреннее состояние

| Данные | Хранение | Описание |
|--------|----------|----------|
| Активные клиенты | In-memory (`ClientPool`) | `Dict[externalSessionId, ActiveClient]` |
| `.session` файлы | Docker volume | SQLite файлы Telethon |
| Сервисный аккаунт | In-memory + volume | Фиксированный клиент, загружается при старте |

## Инварианты

1. **Один клиент на `externalSessionId`** — пул не допускает дубликатов
2. **Сервисный аккаунт всегда активен** — перезапускается при потере соединения
3. **`.session` файл ↔ `externalSessionId`** — 1:1 соответствие
4. **При удалении из пула** — клиент обязательно disconnect-ится
5. **При перезапуске gateway** — пул пуст, backend должен повторно отправить `StartSession`

## Связанные документы

* [Client Pool](client-pool.md)
* [Session Lifecycle](session-lifecycle.md)
* [Service Account](service-account.md)
* [Models — Internal](../models/internal.md)
* [Gateway Overview](../overview.md)
