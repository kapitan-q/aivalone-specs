# Архитектура DDD

## Принципы Domain-Driven Design

Проект организован по принципам **Domain-Driven Design (DDD)** с разделением на **Bounded Contexts** (ограниченные контексты).

## Структура контекста

Каждый контекст содержит четыре слоя:

```
Context/
├── Domain/              # Доменный слой (чистые модели, интерфейсы)
│   ├── Model/          # Доменные модели
│   └── Repository/     # Интерфейсы репозиториев
├── Application/        # Слой приложения (use cases, сервисы)
│   ├── Service/        # Сервисы приложения
│   └── Command/        # Команды (CQRS)
├── Infrastructure/     # Слой инфраструктуры
│   ├── Entity/         # ORM сущности (Doctrine)
│   └── Persistence/    # Реализации репозиториев
└── Presentation/       # Слой представления
    └── Controller/     # REST API контроллеры
```

## Описание слоев

### 1. Domain Layer (Доменный слой)

**Назначение:** Содержит чистые бизнес-модели без зависимостей от инфраструктуры.

**Принципы:**
- Никаких зависимостей от Doctrine, Symfony или других фреймворков
- Интерфейсы репозиториев и сервисов определены здесь
- Использование Value Objects для типизации (например, `UserId`)
- Доменная модель отделена от ORM Entity

**Пример:** `App\Context\Account\Domain\Model\User`

### 2. Application Layer (Слой приложения)

**Назначение:** Содержит use cases и бизнес-логику приложения.

**Принципы:**
- Координирует работу доменных моделей
- Использует интерфейсы из Domain слоя
- Подготовлен для CQRS (разделение Command и Query)

**Пример:** `App\Context\Account\Application\Service\UserService`

### 3. Infrastructure Layer (Слой инфраструктуры)

**Назначение:** Реализация интерфейсов из Domain слоя.

**Принципы:**
- ORM сущности для маппинга в БД
- Внешние интеграции (API, очереди)
- Преобразование между доменной моделью и ORM Entity через методы `toDomain()` и `fromDomain()`

**Пример:** `App\Context\Account\Infrastructure\Persistence\UserRepository`

### 4. Presentation Layer (Слой представления)

**Назначение:** REST API контроллеры и обработка HTTP-запросов.

**Принципы:**
- Валидация входных данных
- Сериализация ответов
- Не содержит бизнес-логики

**Пример:** `App\Context\Account\Presentation\Controller\UserController`

## Разделение доменной модели и ORM Entity

Важно: доменная модель (`Domain\Model\User`) и ORM Entity (`Infrastructure\Entity\UserOrmEntity`) разделены.

**Преимущества:**
- Доменная модель остается независимой от Doctrine
- Легко менять способ персистентности (БД, файлы, API)
- Тестирование доменной логики без БД

**Преобразование:**
- `toDomain()` - преобразование ORM Entity в доменную модель
- `fromDomain()` - преобразование доменной модели в ORM Entity

## Межконтекстное взаимодействие

### Принципы

1. **Изоляция контекстов:** Каждый контекст максимально независим
2. **Общие компоненты:** Выносятся в Shared Context
3. **Асинхронное взаимодействие:** Используется Symfony Messenger для команд и событий
4. **Порты и адаптеры:** Используются интерфейсы (Ports) для взаимодействия между контекстами

### Примеры взаимодействия

- **Billing → Account:** События об изменении прав (`UserPermissionsUpdatedEvent`)
- **Monitoring → Bot:** Команды на отправку уведомлений (`SendNotificationCommand`)
- **Bot → Account:** Проверка прав пользователя через `AccountPortInterface`

## Масштабируемость

При необходимости вынесения контекста в микросервис:
1. Вынести директорию `src/Context/ContextName` в отдельный проект
2. Настроить взаимодействие через API или Message Broker
3. Обновить конфигурацию портов и адаптеров

## Связанные документы

- [Обзор проекта](overview.md)
- [Account Context](../backend/account/overview.md)
- [Billing Context](../backend/billing/overview.md)
- [Bot Context](../backend/bot/overview.md)
- [Monitoring Context](../backend/monitoring/overview.md)
- [Shared Context](../backend/shared/overview.md)
