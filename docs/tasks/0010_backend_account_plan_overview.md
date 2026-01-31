# План реализации Account Context Backend

## 📋 Обзор

Всего **23 задачи** разделены на следующие категории:

- **Shared Context (1)**: Enum тарифов
- **Исключения (5)**: DuplicateUserException, DuplicateMessengerException, UserNotFoundException, MessengerNotFoundException, TariffValidationException
- **Domain Model (4)**: UserId, UserMessenger, User, Events
- **Infrastructure (4)**: UserEntity, UserRepository, UserDataMapper, Doctrine Migration
- **Application Services (8)**: UserService, 4 Command Handlers, 2 Query Handlers, 1 Event Handler

## 📦 Последовательность реализации

### Fase 1: Подготовка (1-7 задачи)

Установка базовых компонентов:

1. [Tariff Enum в Shared Context](0011_backend_account_tariff_enum_shared.md)
2. [DuplicateUserException](0012_backend_account_duplicate_user_exception.md)
3. [DuplicateMessengerException](00013_backend_account_duplicate_messenger_exception.md)
4. [UserNotFoundException](0014_backend_account_user_not_found_exception.md)
5. [MessengerNotFoundException](0015_backend_account_messenger_not_found_exception.md)
6. [TariffValidationException](0016_backend_shared_tariff_validation_exception.md)
7. [UserId Value Object](0017_backend_account_user_id.md)

**Зависимости**: Все компоненты из Shared Context

### Phase 2: Domain Model (8-14 задачи)

Реализация доменной модели:

8. [UserMessenger Value Object](0018_backend_account_user_messenger_value_object.md)
9. [User Domain Model (AggregateRoot)](0019_backend_account_user_domain_model.md)
10. [UserEntity (Doctrine ORM)](0020_backend_account_user_entity_doctrine.md)
11. [UserRepositoryInterface](0021_backend_account_user_repository_interface.md)
12. [Doctrine UserRepository](0022_backend_account_user_repository_doctrine.md)
13. [UserDataMapper](0023_backend_account_user_data_mapper.md)
14. [Domain Events (UserRegistered, UserMessengersUpdated, UserTariffsUpdated)](0024_backend_account_domain_events.md)

**Зависимости**: Phase 1

### Phase 3: Application Services (15-23 задачи)

Реализация Application Layer:

15. [UserService](0025_backend_account_user_service.md)
16. [RegisterUserCommandHandler](0026_backend_account_register_user_command_handler.md)
17. [AddUserMessengerCommandHandler](0027_backend_account_add_user_messenger_command_handler.md)
18. [RemoveUserMessengerCommandHandler](0028_backend_account_remove_user_messenger_command_handler.md)
19. [UpdateUserTariffsCommandHandler](0029_backend_account_update_user_tariffs_command_handler.md)
20. [Doctrine Migration](0030_backend_account_doctrine_migration.md)
21. [UserTariffsUpdatedEventHandler (слушает из Billing Context)](0031_backend_account_user_tariffs_updated_event_handler.md)
22. [GetUserByIdQueryHandler](0032_backend_account_get_user_by_id_query_handler.md)
23. [GetUserByMessengerQueryHandler](0033_backend_account_get_user_by_messenger_query_handler.md)

**Зависимости**: Phase 2

## 🏗️ Архитектура по слоям

### Domain Layer
- User (AggregateRoot)
- UserId (Value Object)
- UserMessenger (Value Object)
- UserRepositoryInterface (contract)
- Исключения (DomainException based)
- События (DomainEventInterface based)

### Application Layer
- UserService (координирует Domain и Infrastructure)
- Command Handlers (RegisterUser, AddMessenger, RemoveMessenger, UpdateTariffs)
- Query Handlers (GetUserById, GetUserByMessenger)
- Event Handler (UserTariffsUpdatedEventHandler)

### Infrastructure Layer
- UserEntity (Doctrine ORM)
- UserRepository (реализация интерфейса)
- UserDataMapper (Domain ↔ ORM)
- Doctrine Migration
- Symfony DI конфигурация

### Shared Context
- Tariff Enum
- Messenger Enum (используется)
- UUID Value Object (используется)
- DomainEventInterface (используется)
- Исключения базовые (используются)

## 📊 Зависимости между задачами

```
Phase 1 (Preparation)
├─ 01: Tariff Enum (Shared)
├─ 02-06: Исключения
└─ 07: UserId

Phase 2 (Domain)
├─ 08: UserMessenger (зависит от 07)
├─ 09: User (зависит от 08, 14)
├─ 10: UserEntity
├─ 11: UserRepositoryInterface
├─ 12: UserRepository (зависит от 11, 13)
├─ 13: UserDataMapper
└─ 14: Events

Phase 3 (Application)
├─ 15: UserService (зависит от 09, 11, 02-06)
├─ 16-19: Command Handlers (зависят от 15)
├─ 20: Doctrine Migration
├─ 21: Event Handler (зависит от 19)
├─ 22: Query Handler GetById
└─ 23: Query Handler GetByMessenger
```

## 🎯 Критерии готовности проекта

После завершения всех 23 задач, система должна:

- ✅ Создавать новых пользователей через регистрацию
- ✅ Добавлять/удалять мессенджеры пользователю
- ✅ Обновлять тарифы пользователя
- ✅ Синхронизировать тарифы из Billing Context
- ✅ Предоставлять информацию о пользователях BOT Context
- ✅ Публиковать события для других контекстов
- ✅ Все данные сохраняются в БД через Doctrine
- ✅ Все процессы покрыты тестами

## 📝 Формат файлов задач

Все файлы находятся в `/docs/tasks/` с именем:
```
{TASK_NUM}_backend_account_{DESCRIPTION}.md
```

Например:
- `0010_backend_account_tariff_enum_shared.md`
- `0019_backend_account_user_domain_model.md`
- `0019_backend_account_update_user_tariffs_command_handler.md`

## 🔄 Интеграция с другими контекстами

### Из Billing Context
- **Входящее событие**: UserTariffsUpdatedEvent
- **Обработчик**: Task 21 (UserTariffsUpdatedEventHandler)
- **Результат**: Синхронизация тарифов в Account Context

### Для BOT Context
- **Query**: GetUserByMessengerQuery
- **Handler**: Task 23 (GetUserByMessengerQueryHandler)
- **Использование**: Идентификация пользователя по Telegram ID

### Для других контекстов
- **Events**: UserRegistered, UserMessengersUpdated, UserTariffsUpdated
- **Queries**: GetUserById, GetUserByMessenger
- **Использование**: Синхронизация данных пользователей

## 📞 Контакты при вопросах

При возникновении вопросов по реализации:
1. Проверьте спецификацию в `/docs/backend/account/`
2. Посмотрите процессы в `/docs/backend/account/processes/`
3. Проверьте сервисы в `/docs/backend/account/services/`

## Статус

- Phase 1: ✅ Done
- Phase 2: ✅ Done
- Phase 3: ✅ Done (кроме задачи 0031 - skip, требует интеграции с Billing Context)

**Дата создания плана**: 30 января 2026 года
**Дата завершения реализации**: 31 января 2026 года
