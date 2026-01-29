# 00018: Создать файл .env.example

## Категория
Конфигурация / Документация

## Приоритет
**Важный** - Стандартная практика для проектов Symfony

## Описание проблемы

В проекте отсутствует файл `.env.example` (или `.env.dist`), который является стандартной практикой для Symfony-проектов:

1. **Документирование** - Разработчики видят все необходимые переменные
2. **Безопасность** - В `.env.example` не должно быть реальных секретов
3. **Onboarding** - Новые разработчики могут быстро настроить проект
4. **CI/CD** - Используется как шаблон для создания `.env` в пайплайнах

Файл `.env` содержит реальные значения и **не должен** коммититься в репозиторий (должен быть в `.gitignore`).

## Текущее состояние

- `.env` - существует, содержит рабочую конфигурацию
- `.env.example` - **отсутствует**

## Необходимые действия

### 1. Создать файл `.env.example`

```env
###> Symfony Environment ###
APP_ENV=dev
APP_DEBUG=1
APP_SECRET=change_me_to_random_string
APP_URL=http://localhost
###< Symfony Environment ###

###> Database Configuration ###
# PostgreSQL connection
DATABASE_URL="pgsql://aivalone:aivalone@db/aivalone?serverVersion=16&charset=utf8"
###< Database Configuration ###

###> Mailer Configuration ###
# Use MailHog for local development
MAILER_DSN=smtp://mailhog:1025
###< Mailer Configuration ###

###> Cache / Messenger / Session (Redis) ###
REDIS_URL=redis://redis:6379
###< Cache / Messenger / Session ###

###> Symfony Messenger (внутренние сообщения) ###
MESSENGER_TRANSPORT_DSN=redis://redis:6379/messages
###< Symfony Messenger ###

###> Python Bridge (RabbitMQ) ###
PYTHON_BRIDGE_AMQP_DSN=amqp://guest:guest@rabbitmq:5672/%2f
###< Python Bridge ###

###> Telegram Bot ###
# Токен от @BotFather
TELEGRAM_BOT_TOKEN=your_bot_token_here
# Случайная строка для валидации вебхуков
TELEGRAM_SECRET_TOKEN=your_random_secret_string_here
###< Telegram Bot ###

###> Logging ###
LOG_CHANNEL=stack
###< Logging ###
```

### 2. Обновить `.gitignore`

Убедиться, что `.env` (но не `.env.example`) в `.gitignore`:

```gitignore
/.env
/.env.local
/.env.*.local
!/.env.example
```

### 3. Обновить README.md

Добавить в секцию "Быстрый старт":
```markdown
4. Создайте файл `.env` из примера:
\`\`\`bash
cp .env.example .env
\`\`\`

5. Отредактируйте `.env` и установите ваши значения:
   - `APP_SECRET` - случайная строка (можно сгенерировать через `openssl rand -hex 32`)
   - `TELEGRAM_BOT_TOKEN` - токен от @BotFather
   - `TELEGRAM_SECRET_TOKEN` - случайная строка (можно сгенерировать через `openssl rand -hex 16`)
```

## Связанные задачи

- [00017_bot_backend_missing_env_vars.md](00017_bot_backend_missing_env_vars.md) - Добавление переменных окружения

## Файлы для создания/изменения

- `services/aivalone-backend/.env.example` - **создать**
- `services/aivalone-backend/.gitignore` - **проверить/обновить**
- `services/aivalone-backend/README.md` - **обновить**
