# Task 00001: Backend Core Initialization (Symfony & DDD)

## 1. Контекст
Согласно `Readme.md` проекта, мы начинаем реализацию основного бэкенда на **PHP 8.4 / Symfony 8**. Сервис проектируется как модульный монолит в парадигме **DDD**, что позволит в будущем выносить контексты (например, Bot, Billing, Monitoring) в отдельные микросервисы.

## 2. Цели задачи
1. Развернуть базовый скелет Symfony в Docker-контейнере.
2. Настроить окружение (PostgreSQL, Redis/RabbitMQ).
3. Создать архитектурную структуру под **DDD контексты**.
4. Реализовать контекст **Account** для управления пользователями.
5. Реализовать базовую сущность User и Health Check.
6. Реализовать базовый контекст **Shared** для общих системных компонентов.
7. Создать файл `README.md` проекта, в котором описать все существенные, для дальнейшей разработки, моменты.

## 3. Технические требования
* **Stack:** PHP 8.3-fpm, Nginx, PostgreSQL 16.
* **Symfony Components:**
    * `webapp-pack` / `skeleton`
    * `messenger` (транспорт для межконтекстного взаимодействия)
    * `mailer` (для системных уведомлений)
    * `orm-pack` & `maker-bundle`
* **Architecture:** **DDD (Domain-Driven Design)**. 
    * Разделение на `Bounded Contexts`.
    * Каждый контекст содержит слои: `Domain`, `Application`, `Infrastructure`, `Presentation`.

## 4. План реализации

### Шаг 1: Инфраструктура (Docker)
Создать `docker-compose.yaml` в `backend-symfony/`:
* Сервис `php-fpm` (с расширениями `pdo_pgsql`, `intl`, `zip`).
* Сервис `db` (PostgreSQL 16) , при установке Doctrine может быть добавлен сервис database, можно использовать его.
* Сервис `mailhog` или `mailer` (для тестирования отправки писем в dev-режиме).
* Сервис `nginx`.

### Шаг 2: Инициализация и Структура DDD
1. `composer create-project symfony/skeleton .` и установка компонентов (`api`, `mailer`, `messenger`).
2. Создать корневую директорию `src/Context`.

### Шаг 3: Создание контекста Account
* **Domain:** Создать `src/Context/Account/Domain/Model/User.php`. Поля: `id`, `telegram_id`. Без зависимостей от Doctrine.
* **Infrastructure:** Настроить сохранение пользователя. Реализовать `UserRepository` через интерфейс из Domain.
* **Application:** Создать сервис/команду для регистрации/получения пользователя.

### Шаг 3: Контекст Shared
* **Domain:** Общие типы данных (например, `UserId` как Value Object).
* **Presentation:** Базовый `HealthCheckController` для проверки всей системы.

### Шаг 4: Mailer & Messenger
* Конфигурация Mailer для работы с Mailhog.
* Подготовка шины Messenger для взаимодействия между контекстами.

## 5. Definition of Done (Критерии приемки)
- [ ] Контейнеры (php, db, nginx, mailer) запускаются без ошибок.
- [ ] Соблюдена структура директорий `src/Context/[Name]`.
- [ ] Чистая доменная модель `App\Context\Account\Domain\User`, маппинг базы данных описывается исключительно в слое `Infrastructure`, через OrmEntity.
- [ ] Эндпоинт `/api/health` возвращает статус `ok`.
- [ ] База данных инициализирована, таблица `users` создана.
- [ ] Почтовый сервис (Mailer) сконфигурирован и готов к работе.

---
**Связи:** Фундаментальная задача. Архитектура DDD закладывает возможность независимого масштабирования контекста **Account** и других будущих модулей.