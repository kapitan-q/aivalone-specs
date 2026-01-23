# Вебхуки (Webhooks)

## Обзор

Система управления вебхуками обеспечивает безопасную связь между бэкендом и API мессенджеров (Telegram, WhatsApp и т.д.). Архитектура поддерживает работу с несколькими мессенджерами через единый интерфейс.

## Переменные окружения

### Общие (для всех мессенджеров)

- `APP_URL` - Базовый URL бэкенда (должен быть HTTPS)

### Для Telegram

- `TELEGRAM_BOT_TOKEN` - Уникальный токен от BotFather
- `TELEGRAM_SECRET_TOKEN` - Случайная строка для проверки, что запрос пришел именно от Telegram

## Консольные команды

### bot:webhook:set

Устанавливает или обновляет вебхук для мессенджера.

**Синтаксис:**
```bash
php bin/console bot:webhook:set [--messenger=telegram] [--drop-pending-updates]
```

**Параметры:**
- `--messenger` - Имя мессенджера (по умолчанию: `telegram`)
- `--drop-pending-updates` - Очистить очередь сообщений, накопившихся на серверах мессенджера (критически важно при деплое на "боевой" сервер)

**Логика работы:**

1. Формирует URL: `https://{APP_URL}/webhook/{messenger}`
2. Отправляет запрос к API мессенджера (например, `https://api.telegram.org/bot{TOKEN}/setWebhook`)
3. Передает параметры:
   - `url` - адрес нашего эндпоинта
   - `secret_token` - значение из конфига для защиты
   - `allowed_updates` - типы обновлений (например, `["message", "callback_query", "my_chat_member"]`)
   - `drop_pending_updates` - `true` (чтобы избежать очереди старых сообщений при смене логики)

**Повторный вызов:**
- Автоматически перезаписывает старые настройки в мессенджере

### bot:webhook:delete

Удаляет вебхук для мессенджера.

**Синтаксис:**
```bash
php bin/console bot:webhook:delete [--messenger=telegram]
```

**Применение:**
- Деактивация вебхука (например, для перевода бота в режим Long Polling при локальной отладке)
- Временная остановка обслуживания

**Действие:**
- Вызывает метод API мессенджера `deleteWebhook`

### bot:webhook:info

Выводит информацию о текущем состоянии вебхука.

**Синтаксис:**
```bash
php bin/console bot:webhook:info [--messenger=telegram]
```

**Выводимая информация:**
- `url` - Текущий адрес бэкенда
- `pending_update_count` - Количество сообщений, ожидающих обработки
- `last_error_date` / `last_error_message` - Диагностика проблем с SSL или доступностью сервера
- `allowed_updates` - Разрешенные типы обновлений

## Безопасность

### Проверка секретного токена

Контроллер вебхука (`WebhookController`) выполняет проверку секретного токена:

1. Проверяет заголовок `X-Telegram-Bot-Api-Secret-Token` (для Telegram)
2. Сравнивает с `TELEGRAM_SECRET_TOKEN` из переменных окружения
3. Возвращает `403 Forbidden` при несовпадении

**Важно:** Все запросы к вебхуку без корректного заголовка завершаются ошибкой 403.

### Валидация запроса

Валидация запроса происходит в адаптере мессенджера через метод `validateRequest()`.

**Для Telegram:**
- Проверка секретного токена
- Проверка структуры обновления
- Проверка подписи (если требуется)

## Fast Response

Контроллер возвращает `200 OK` сразу после проверки секрета. Обработка обновлений выполняется асинхронно через Symfony Messenger, что позволяет мессенджеру быстро получить подтверждение доставки.

**Алгоритм:**
1. **Security Check:** Middleware проверяет заголовок секретного токена
2. **Fast Response:** Сервер возвращает `200 OK` до начала выполнения тяжелых задач
3. **Async Processing:** Обработка обновления выполняется асинхронно через Messenger

## Архитектура

Система управления вебхуками построена на принципах DDD:

### Domain Layer

- `Domain/Messenger/WebhookManagerInterface` - Абстракция управления вебхуками

**Методы:**
- `setup(string $url, array $options = []): bool` - Установка вебхука
- `remove(): bool` - Удаление вебхука
- `getInfo(): array` - Получение информации о вебхуке

### Application Layer

- `Application/Service/WebhookService` - Сервис для управления вебхуками
- `Application/Service/WebhookManagerFactory` - Фабрика для получения адаптеров вебхуков

### Infrastructure Layer

- `Infrastructure/Messenger/TelegramWebhookAdapter` - Конкретная реализация для Telegram API

## WebhookService

Сервис для управления вебхуками разных мессенджеров.

**Методы:**
- `setup(string $messenger, bool $dropPendingUpdates = false): bool` - Установка вебхука
- `remove(string $messenger): bool` - Удаление вебхука
- `getInfo(string $messenger): array` - Получение информации о вебхуке

**Использование:**
- Использует `WebhookManagerFactory` для получения адаптера конкретного мессенджера
- Формирует URL на основе `APP_URL` и кода мессенджера

## WebhookManagerFactory

Фабрика для получения адаптеров вебхуков разных мессенджеров.

**Регистрация:**
- Через Symfony ServiceLocator
- Ключ в локаторе = код мессенджера (telegram, whatsapp и т.д.)

**Пример:**
```yaml
webhook.adapters.locator:
    class: Symfony\Component\DependencyInjection\ServiceLocator
    arguments: !service_locator
        telegram: '@App\Context\Bot\Infrastructure\Messenger\TelegramWebhookAdapter'
```

## WebhookController

Контроллер для приема вебхуков от мессенджеров.

**Роут:**
- `POST /webhook/{messenger}` - Прием обновлений от мессенджера

**Алгоритм обработки:**

1. **Security Check:** Middleware проверяет заголовок секретного токена
2. **Fast Response:** Возвращает `200 OK` сразу после проверки
3. **Async Processing:** Передает обновление в `BotRouter` для асинхронной обработки

**Важно:** Контроллер должен вернуть ответ как можно быстрее, чтобы мессенджер не считал запрос неудачным.

## Добавление нового мессенджера

Для добавления поддержки нового мессенджера (например, WhatsApp):

1. Создать `WhatsAppWebhookAdapter implements WebhookManagerInterface`
2. Добавить в ServiceLocator: `whatsapp: '@App\Context\Bot\Infrastructure\Messenger\WhatsAppWebhookAdapter'`
3. Добавить новый роут в контроллере: `/webhook/whatsapp`
4. Настроить переменные окружения для нового мессенджера

## Связанные документы

- [Обзор Bot Context](overview.md)
- [Роутинг](router.md)
- [Диалоги](conversations.md)
