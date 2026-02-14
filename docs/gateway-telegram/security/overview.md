# Security — Обзор

## Описание

Спецификация безопасности Gateway Telegram: классификация чувствительных данных, правила хранения, логирования, передачи, и модель угроз.

## Классификация данных

### Критические (никогда не покидают gateway)

| Данные | Где хранятся | Логируются | Передаются в MQ |
|--------|-------------|-----------|-----------------|
| `.session` файлы (auth key) | Docker volume | ❌ | ❌ |
| `phone_code_hash` | In-memory (AuthState) | ❌ | ❌ |
| 2FA пароль | Не хранится (транзитно) | ❌ | ❌ (приходит, не отправляется) |

### Чувствительные (ограниченная передача)

| Данные | Где хранятся | Логируются | Передаются в MQ |
|--------|-------------|-----------|-----------------|
| `phoneNumber` | In-memory (AuthState, TTL 15 мин) | ❌ | ❌ (приходит в InitAuth) |
| Код подтверждения | Не хранится (транзитно) | ❌ | ❌ (приходит в SubmitCode) |
| `externalSessionId` | In-memory (ClientPool) | ✅ | ✅ (только AuthSuccess) |
| `externalGroupId` | In-memory (GroupRegistry) | ✅ | ✅ (MQ события) |

### Некритические

| Данные | Логируются | Передаются |
|--------|-----------|-----------|
| `sessionId` (backend UUID) | ✅ | ✅ |
| `groupId` (backend UUID) | ✅ | ✅ |
| `correlationId` | ✅ | ✅ |
| `displayName` | ✅ | ✅ |
| `groupTitle` | ✅ | ✅ |
| `memberCount` | ✅ | ✅ |
| `reason`, `message` (ошибки) | ✅ | ✅ |

## Правила логирования

### Структурированные логи (structlog + JSON)

```python
# ПРАВИЛЬНО
logger.info("auth_code_sent", session_id=session_id, code_length=5)

# НЕПРАВИЛЬНО — НИКОГДА так не делать
logger.info("auth_code_sent", phone="+79001234567", code="12345")
```

### Запрещённые поля в логах

* `phoneNumber` / `phone` / `phone_number`
* `code` (код подтверждения)
* `password` (2FA)
* `phone_code_hash`
* Содержимое `.session` файлов

### Разрешённые поля в логах

* `session_id`, `external_session_id`, `group_id`, `external_group_id`
* `correlation_id`, `command`, `event`
* `error_reason`, `telethon_error` (тип исключения)
* `display_name`, `group_title`, `member_count`
* Таймстемпы, длительности, счётчики

## Хранение .session файлов

### Docker Volume

```yaml
volumes:
  gateway_telegram_sessions:
    driver: local
```

* Volume монтируется **только** в контейнер gateway-telegram
* Другие контейнеры (backend, postgres, redis) **не имеют доступа**
* Volume persistent — переживает перезапуск контейнера

### Права доступа

```dockerfile
# В Dockerfile
RUN mkdir -p /app/sessions && chmod 700 /app/sessions
```

* Только процесс gateway (пользователь контейнера) может читать/писать
* `chmod 700` — нет доступа для group/other

### Удаление

`.session` файлы удаляются при:

| Событие | Действие |
|---------|---------|
| `StopSession` | `unlink(session_file)` |
| `SessionExpired` (session_revoked) | `unlink(session_file)` |
| Терминальная ошибка auth (canRetry=false) | `unlink(session_file)` |

> Сервисный аккаунт **никогда не удаляется** автоматически.

## RabbitMQ безопасность

### Транспорт

| Аспект | MVP | Post-MVP |
|--------|-----|----------|
| Протокол | AMQP (plain) | AMQPS (TLS) |
| Аутентификация | username/password | username/password + vhost isolation |
| Авторизация | Доступ ко всем exchanges | Per-user permissions |

### Sensitive data в MQ

| Поле | В командах (Backend → GW) | В событиях (GW → Backend) |
|------|--------------------------|--------------------------|
| `phoneNumber` | ✅ (InitAuth) | ❌ |
| `code` | ✅ (SubmitCode) | ❌ |
| `password` | ✅ (Submit2FA) | ❌ |
| `externalSessionId` | ✅ (StartSession, StopSession) | ✅ (AuthSuccess только) |
| `externalGroupId` | ✅ (JoinGroup, LeaveGroup) | ✅ (GroupJoined, MessageReceived) |

Backend отвечает за то, чтобы `externalSessionId` и `externalGroupId` **не попадали в domain events и API responses**.

## Модель угроз (MVP)

### 1. Компрометация .session файла

**Угроза**: Злоумышленник получает доступ к Docker volume и копирует .session файл.

**Последствия**: Полный доступ к Telegram-аккаунту пользователя.

**Митигации**:
* Docker volume изолирован (только gateway контейнер)
* Post-MVP: шифрование .session файлов at rest

### 2. Перехват MQ сообщений

**Угроза**: Злоумышленник подключается к RabbitMQ и читает команды.

**Последствия**: Доступ к phoneNumber, code, password.

**Митигации**:
* RabbitMQ в Docker-сети (не exposed наружу)
* Post-MVP: AMQPS (TLS), per-user permissions

### 3. Утечка через логи

**Угроза**: Чувствительные данные попадают в логи.

**Последствия**: Доступ к номерам телефонов, кодам.

**Митигации**:
* Строгий allowlist полей для логирования
* Code review на наличие чувствительных данных в логах

### 4. Denial of Service через FloodWait

**Угроза**: Telegram блокирует аккаунт из-за слишком частых запросов.

**Последствия**: Сервисный аккаунт заблокирован, публичные группы не мониторятся.

**Митигации**:
* Rate limiting на стороне gateway (post-MVP)
* Пул сервисных аккаунтов (post-MVP)
* Уважение FloodWaitError — не retry-ить

## Чеклист безопасности при реализации

- [ ] `.session` файлы в Docker volume с `chmod 700`
- [ ] Structlog настроен с явным allowlist полей
- [ ] `phoneNumber`, `code`, `password` не логируются ни при каких условиях
- [ ] `phone_code_hash` не логируется
- [ ] `.session` файлы удаляются при StopSession и SessionExpired
- [ ] RabbitMQ не exposed за пределы Docker-сети
- [ ] Env-переменные с секретами не попадают в docker logs
- [ ] `.env` файл в `.gitignore`

## Связанные документы

* [Gateway Overview](../overview.md)
* [Infrastructure Overview](../infrastructure/overview.md)
* [Auth State](../auth/auth-state.md)
* [Session Lifecycle](../sessions/session-lifecycle.md)
