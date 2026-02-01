# План реализации Billing Context Backend

## 📋 Обзор

Всего **16 задач** разделены на следующие категории:

- **Value Objects & Enums (3)**: TariffId, UserSubscriptionId, SubscriptionStatus (TariffOptionType & Tariff из Shared)
- **Исключения (7)**: TariffNotFoundException, InvalidSubscriptionException и т.д.
- **Domain Models (3)**: Tariff, TariffOption, UserSubscription
- **Domain Events (4)**: TariffUpdated, UserSubscriptionUpdated, SubscriptionExpired, SubscriptionExpiringSoon
- **Application Services (1)**: SubscriptionService
- **Command Handlers (2)**: AddUserSubscription, RemoveSubscription
- **Query Handlers (1)**: GetTariffList
- **Infrastructure (3)**: Doctrine Entity/Repository, Migrations, Event Handlers
- **CLI Commands (1)**: CheckSubscriptionExpiration

## 📦 Последовательность реализации

### Phase 1: Foundation (Задачи 0034-0040)

Установка базовых компонентов: Value Objects, Enums.

1. [TariffId Value Object](../../../tasks/0037_backend_billing_tariff_id_value_object.md)
2. [UserSubscriptionId Value Object](../../../tasks/0038_backend_billing_user_subscription_id_value_object.md)
3. [SubscriptionStatus Enum](../../../tasks/0039_backend_billing_subscription_status_enum.md)

**Примечание**: Tariff enum используется из [Shared Context](../../shared/models/tariff.md)

**Зависимости**: Все используют Shared Context (UUID, ValidationException, Tariff enum и т.д.)

---

### Phase 2: Domain Models & Events (Задачи 0034-0036, 0042)

Реализация доменных моделей и событий.

5. [Tariff Domain Model (AggregateRoot)](../../../tasks/0034_backend_billing_tariff_domain_model.md)
6. [TariffOption Value Object](../../../tasks/0035_backend_billing_tariff_option_value_object.md)
7. [UserSubscription Domain Model (AggregateRoot)](../../../tasks/0036_backend_billing_user_subscription_domain_model.md)
8. [Исключения Billing](../../../tasks/0041_backend_billing_exceptions.md)
9. [Domain Events](../../../tasks/0042_backend_billing_domain_events.md)

**Зависимости**: Phase 1 + Исключения

---

### Phase 3: Application Layer (Задачи 0046, 0043-0045)

Реализация бизнес-логики и обработчиков команд/запросов.

10. [SubscriptionService](../../../tasks/0046_backend_billing_subscription_service.md)
11. [AddUserSubscription Command Handler](../../../tasks/0043_backend_billing_add_user_subscription_command_handler.md)
12. [RemoveSubscription Command Handler](../../../tasks/0044_backend_billing_remove_subscription_command_handler.md)
13. [GetTariffList Query Handler](../../../tasks/0045_backend_billing_get_tariff_list_query_handler.md)

**Зависимости**: Phase 1 & 2 + Application interfaces (Shared Context)

---

### Phase 4: Infrastructure (Задачи 0048-0049, 0047)

Реализация персистентности и интеграции.

14. [Doctrine Entity, Repository, DataMapper](../../../tasks/0048_backend_billing_doctrine_infrastructure.md)
15. [Event Handlers](../../../tasks/0049_backend_billing_event_handlers.md)
16. [CLI Command: Check Subscription Expiration](../../../tasks/0047_backend_billing_cli_check_subscription_expiration.md)

**Зависимости**: Phase 1-3 + Doctrine ORM, Symfony Console

---

## 🎯 Критические пути

```
Phase 1 (Foundation)
    ↓
Phase 2 (Domain)
    ↓
Phase 3 (Application)
    ↓
Phase 4 (Infrastructure)
```

**Параллельные задачи в одной Phase**:
- Все задачи одной Phase можно выполнять в параллель (с учетом внутренних зависимостей)

---

## 📊 Статистика

| Категория | Количество | Статус |
| --------- | ---------- | ------ |
| Value Objects | 2 | ⏳ не начата |
| Enum'ы | 2 | ⏳ не начата |
| Domain Models | 3 | ⏳ не начата |
| Exceptions | 1 (7 исключений) | ⏳ не начата |
| Domain Events | 1 (4 события) | ⏳ не начата |
| Services | 1 | ⏳ не начата |
| Command Handlers | 2 | ⏳ не начата |
| Query Handlers | 1 | ⏳ не начата |
| Infrastructure | 3 | ⏳ не начата |
| CLI Commands | 1 | ⏳ не начата |
| **ИТОГО** | **16 задач** | ⏳ |

---

## 📝 Документация

Все спецификации разделены на отдельные файлы:

- [Модели](../models/overview.md)
- [События](../events/overview.md)
- [Исключения](../exceptions/overview.md)
- [Сервисы](../services/overview.md)

Каждая спецификация содержит:
- Подробное описание
- Методы и атрибуты
- Примеры использования
- Статус реализации с чекбоксами
- Ссылки на связанные сущности

---

## Связанные документы

* [Billing Context Overview](../overview.md)
* [Процессы](../processes/overview.md)
* [Account Context Plan](../../account/plan-overview.md)
