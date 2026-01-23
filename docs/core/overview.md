# Обзор проекта Aivalone

## Описание

**Aivalone** — SaaS-платформа для мониторинга и фильтрации Telegram-контента. Система построена на базе микросервисного взаимодействия Python (обработка данных) и PHP/Symfony (бизнес-логика).

## Архитектура системы

Проект реализован как **модульный монолит** с применением принципов **Domain-Driven Design (DDD)**, что позволяет в будущем выносить контексты в отдельные микросервисы.

### Основные компоненты

1. **PHP Symfony Service (Core Backend)** - Основной монолит, отвечающий за бизнес-логику
   - Управление пользователями (Account Context)
   - Управление тарифами и подписками (Billing Context)
   - Telegram-бот и диалоги (Bot Context)
   - Мониторинг и фильтрация контента (Monitoring Context)
   - Общие компоненты (Shared Context)

2. **Python Service (Telegram Bridge)** - Специализированный сервис для прямого взаимодействия с Telegram API
   - Мониторинг новых сообщений в группах
   - Выполнение команд от PHP-сервиса

3. **AI Integration Service** - Сервис для работы с тяжелыми вычислениями (планируется)
   - Семантический анализ для Premium тарифа
   - Computer Vision для анализа изображений

## Технологический стек

- **Backend:** PHP 8.3, Symfony 8.0
- **База данных:** PostgreSQL 16
- **Кэш/Очереди:** Redis
- **Очереди для Python-bridge:** RabbitMQ (AMQP)
- **Почтовый сервис:** MailHog (для разработки)
- **Веб-сервер:** Nginx
- **Telegram Engine:** Python 3.x (Telethon/Pyrogram)

## Структура проекта

```
services/
└── aivalone-backend/          # PHP Symfony Backend
    └── src/Context/
        ├── Account/           # Контекст управления пользователями
        ├── Billing/          # Контекст тарифов и подписок
        ├── Bot/              # Контекст Telegram-бота
        ├── Monitoring/       # Контекст мониторинга и фильтрации
        └── Shared/           # Общий контекст
```

## Связанные документы

- [Архитектура DDD](architecture.md)
- [Account Context](../backend/account/overview.md)
- [Billing Context](../backend/billing/overview.md)
- [Bot Context](../backend/bot/overview.md)
- [Monitoring Context](../backend/monitoring/overview.md)
- [Shared Context](../backend/shared/overview.md)
