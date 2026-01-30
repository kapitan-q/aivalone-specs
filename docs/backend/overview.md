# Aivalone Backend Overview

## Описание проекта

Backend-сервис платформы Aivalone. 
Проект спроектирован как модульный монолит с возможностью вынесения контекстов в отдельные микросервисы в будущем.

## Технологический стек

* **PHP:** 8.4-fpm
* **Framework:** Symfony 8.0
* **База данных:** PostgreSQL 16
* **Кэш:** Redis
* **Очереди:** RabbitMQ (AMQP)
* **Почтовый сервис:** MailHog (для разработки)
* **Веб-сервер:** Nginx

## Конфигурация проекта

### База данных

```
DATABASE_URL="postgresql://aivalone:aivalone@db:5432/aivalone?serverVersion=16&charset=utf8"
```

### Почта

* MailHog для разработки
* Web UI: [http://localhost:8025](http://localhost:8025)

### Очереди

* **Redis** — внутренние команды и события Symfony Messenger
* **RabbitMQ** — внешниее взаимодействие

```
MESSENGER_TRANSPORT_DSN=redis://redis:6379/messages
PYTHON_BRIDGE_AMQP_DSN=amqp://guest:guest@rabbitmq:5672/%2f
```

### Telegram Bot

```
TELEGRAM_BOT_TOKEN=your_bot_token_here
```

Webhook настраивается через API Telegram.

## Запуск проекта

1. Клонировать репозиторий и перейти в директорию проекта
2. Запустить Docker контейнеры:

```bash
docker-compose up -d
```

3. Установить зависимости:

```bash
docker-compose exec php-fpm composer install
```

4. Создать и настроить `.env` и `.env.local`
5. Выполнить миграции:

```bash
docker-compose exec php-fpm php bin/console doctrine:migrations:migrate
```

6. Проверить здоровье API:

```bash
curl http://localhost:8080/api/health
```

Ожидаемый ответ:

```json
{"status":"ok","timestamp":"2026-01-30T00:00:00+00:00","services":{"database":"ok"}}
```

## Разработка

* Чистый Domain Layer без зависимостей на фреймворки
* Value Objects для типизации данных
* Разделение Command и Query для CQRS
* Межконтекстное взаимодействие через Symfony Messenger
* Возможность вынесения контекста в отдельный микросервис при необходимости

## Инструменты поддержки

* Консольные команды для прогрева кэша, проверки подписок и управления вебхуками
* Миграции через Doctrine
* Тесты расположены в `tests/` и следуют структуре `src/`

## Порты и сервисы

* **Nginx:** 80
* **PostgreSQL:** 5432
* **Redis:** 6379
* **RabbitMQ:** 5672 (AMQP), 15672 (UI)
* **MailHog SMTP:** 1025, Web UI: 8025

Используйте docker-compose.override.yaml для настройки конкретных портов сервисов

## Лицензия

Proprietary

## Связанные документы

* [Project Overview: Aivalone](../overview.md)
* [Account Overview](account/overview.md)
* [Shared Overview](shared/overview.md)
