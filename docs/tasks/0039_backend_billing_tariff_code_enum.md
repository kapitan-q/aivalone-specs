# Задача 0039: Tariff Enum (Shared Context)

## Описание

Создать Enum `Tariff` в Shared Context — коды доступных тарифов.

> **Примечание**: Этот enum размещается в Shared Context, так как используется несколькими контекстами (Billing, Account, Bot).

## Фаза

**Phase 1: Foundation**

## Спецификация

📄 [tariff.md](../backend/shared/models/tariff.md)

## Зависимости

Нет зависимостей

## Расположение файла

```
src/Context/Shared/Domain/Enum/Tariff.php
```

## Требования к реализации

### Значения Enum

| Значение | Code | Описание |
| -------- | ---- | -------- |
| FREE | `free` | Бесплатный тариф |
| BASE | `base` | Базовый тариф |
| PRO | `pro` | Профессиональный тариф |
| ENTERPRISE | `enterprise` | Корпоративный тариф |

### Методы

| Метод | Описание |
| ----- | -------- |
| `code(): string` | Получить строковый код |
| `fromCode(string): self` | Создать из строки |

## Пример использования

```php
// Использование enum
$tariff = Tariff::FREE;
$code = $tariff->code(); // 'free'

// Создание из строки (например, из API)
$tariff = Tariff::fromCode('pro'); // Tariff::PRO

// Сравнение
if ($tariff === Tariff::PRO) {
    // ...
}
```

## Критерии готовности

- [ ] Создан PHP 8.1+ Enum `Tariff`
- [ ] Реализованы все 4 значения
- [ ] Метод `code()` возвращает строковый код
- [ ] Метод `fromCode()` создаёт enum из строки
- [ ] Написаны unit-тесты
- [ ] Код соответствует PSR-12

## Связанные задачи

- 0034: Tariff Aggregate Root
- 0036: UserSubscription Aggregate Root

## Примечание

Если этот enum уже существует в Shared Context (задача 0011 для Account), убедитесь что он соответствует спецификации или обновите его.
