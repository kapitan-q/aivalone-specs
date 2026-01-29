# 00019: Добавить RabbitMQ в docker-compose.yaml

## Категория
Инфраструктура / Docker

## Приоритет
**Критический** - Без RabbitMQ не работает взаимодействие с Python-bridge

## Описание проблемы

В `docker-compose.yaml` отсутствует сервис RabbitMQ, хотя он:

1. **Документирован в README.md**:
   - Порт AMQP: 5672
   - Порт Management UI: 15672
   - Используется для Python-bridge

2. **Используется в конфигурации Messenger** (`config/packages/messenger.yaml`):
   - Транспорт `python_bridge` - команды в Python-bridge
   - Транспорт `python_bridge_events` - события от Python-bridge

3. **Упоминается в переменной окружения**:
   - `PYTHON_BRIDGE_AMQP_DSN=amqp://guest:guest@rabbitmq:5672/%2f`

## Текущий docker-compose.yaml

```yaml
services:
  php-fpm:
    # ...
  nginx:
    # ...
  db:
    # ...
  redis:
    # ...
  mailhog:
    # ...
```

**Отсутствует сервис `rabbitmq`!**

## Необходимые изменения

Добавить в `docker-compose.yaml`:

```yaml
  rabbitmq:
    image: rabbitmq:3-management-alpine
    container_name: aivalone-rabbitmq
    environment:
      RABBITMQ_DEFAULT_USER: guest
      RABBITMQ_DEFAULT_PASS: guest
    # ports:
    #   - "5672:5672"
    #   - "15672:15672"
    volumes:
      - rabbitmq_data:/var/lib/rabbitmq
    networks:
      - aivalone-network
```

И добавить volume:

```yaml
volumes:
  postgres_data:
  rabbitmq_data:  # Добавить
```

## Также добавить depends_on

Обновить сервис `php-fpm`:

```yaml
  php-fpm:
    # ...
    depends_on:
      - db
      - redis
      - rabbitmq  # Добавить
```

## Связанные задачи

- [00017_bot_backend_missing_env_vars.md](00017_bot_backend_missing_env_vars.md) - Переменная `PYTHON_BRIDGE_AMQP_DSN`

## Файлы для изменения

- `services/aivalone-backend/docker-compose.yaml`

## Верификация

После внесения изменений:
```bash
docker-compose up -d
docker-compose exec php-fpm php bin/console messenger:consume python_bridge --limit=1 --dry-run
```

Или проверить доступность RabbitMQ Management UI:
```
http://localhost:15672
# login: guest / guest
```
