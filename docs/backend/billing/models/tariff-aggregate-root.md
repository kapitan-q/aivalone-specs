# Tariff Domain Model (AggregateRoot)

## Описание

Доменная модель `Tariff` управляет информацией о тарифе, его свойствами и опциями (ограничениями). Является AggregateRoot и отвечает за инварианты своих опций.

## Атрибуты

- `id: TariffId` — уникальный идентификатор (UUID)
- `code: Tariff` — enum код тарифа (FREE, BASE, PRO, ENTERPRISE из Shared)
- `name: string` — название тарифа (100 символов максимум)
- `priority: int` — приоритет (0 для FREE, 1 для PRO и т.д.)
- `price: decimal` — стоимость (не может быть отрицательным)
- `options: Collection<TariffOption>` — коллекция опций тарифа
- `createdAt: DateTimeImmutable`
- `updatedAt: DateTimeImmutable`

## Основные методы

### static create(Tariff $code, string $name, int $priority, decimal $price): self

Создание нового тарифа.

**Логика**:
- Инициализирует Tariff с пусто коллекцией опций
- Устанавливает timestamps
- Возвращает новый объект

**Возвращает**: Новый объект `Tariff` (не сохраненный)

---

### addOption(string $name, string $code, TariffOptionType $type, mixed $value): void

Добавляет новую опцию к тарифу.

**Логика**:
- Создает новый объект `TariffOption`
- Проверяет, что опция с таким кодом еще не существует
- Добавляет опцию в коллекцию
- Обновляет `updatedAt`
- Регистрирует событие `TariffUpdated`

**Выбрасывает**: ValidationException, DomainException (если опция уже существует)

---

### updateOption(string $code, mixed $newValue): void

Обновляет значение существующей опции.

**Логика**:
- Находит опцию по коду
- Если не найдена — выбрасывает `TariffOptionNotFoundException`
- Обновляет значение и `updatedAt`
- Регистрирует событие `TariffUpdated`

---

### removeOption(string $code): void

Удаляет опцию из тарифа.

**Логика**:
- Находит опцию по коду
- Если не найдена — выбрасывает `TariffOptionNotFoundException`
- Удаляет опцию и обновляет `updatedAt`
- Регистрирует событие `TariffUpdated`

---

### updateTariffInfo(string $name, int $priority, decimal $price): void

Обновляет информацию о тарифе.

**Логика**:
- Валидирует входные параметры
- Обновляет поля и `updatedAt`
- Регистрирует событие `TariffUpdated`

---

### Методы доступа

```php
public function getId(): TariffId
public function getCode(): Tariff
public function getName(): string
public function getPriority(): int
public function getPrice(): decimal
public function getOptions(): Collection<TariffOption>
public function getOptionByCode(string $code): TariffOption|null
public function getCreatedAt(): DateTimeImmutable
public function getUpdatedAt(): DateTimeImmutable
```

## Инварианты

- Tariff всегда имеет TariffId
- Tariff всегда имеет валидный Tariff enum (из Shared Context)
- Tariff всегда имеет название (не пусто, максимум 100 символов)
- Price не может быть отрицательным
- Priority не может быть отрицательным
- TariffOption коды уникальны в рамках одного Tariff
- updatedAt обновляется при любом изменении
- TariffOption не может существовать отдельно от Tariff

## Статус реализации

- [ ] Класс создан с базовой структурой
- [ ] Все методы реализованы
- [ ] Все валидации работают
- [ ] События регистрируются корректно
- [ ] Unit тесты написаны (15+ тестов)
- [ ] Integration тесты пройдены
- [ ] Код соответствует стилю проекта
- [ ] Документация актуальна

## Связанные сущности

- [TariffOption Value Object](tariff-option-value-object.md)
- [TariffId Value Object](tariff-id-value-object.md)
- [Tariff Enum](../../shared/models/tariff.md) (из Shared Context)
- [TariffUpdated Event](../events/tariff-updated-event.md)
- [Billing Context Overview](../overview.md)
