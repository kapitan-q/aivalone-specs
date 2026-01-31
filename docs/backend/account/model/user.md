# User Model Specification

## Назначение

Доменная модель пользователя платформы Aivalone. 
Отвечает за хранение идентификации пользователя, активных тарифов и подключенных мессенджеров. 
`User` является [AggregateRoot](../../shared/models/aggregate-root.md).

### Поля

| Поле          | Тип                    | Описание                                                                           |
| ------------- | ---------------------- | ---------------------------------------------------------------------------------- |
| `userId`      | `UserId`               | Уникальный идентификатор пользователя (UUID)                                       |
| `tariffs`     | `array<Tariff>`        | Список активных тарифов пользователя (enum объекты из Shared Context) |
| `messengers`  | `array<UserMessenger>` | Список подключенных мессенджеров пользователя (Telegram, WhatsApp, Discord и т.п.) |
| `createdAt`   | `DateTimeImmutable`    | Дата создания пользователя                                                         |
| `updatedAt`   | `DateTimeImmutable`    | Дата последнего изменения пользователя                                             |

### Методы / Возможные действия

* `static registerUser(Messenger $messenger, string $messengerId): self` — создание нового пользователя с генерацией `UserId` внутри метода.
* `addTariff(Tariff $tariff): void` — добавить тариф к пользователю.
* `removeTariff(Tariff $tariff): void` — удалить тариф у пользователя.
* `replaceTariffs(array<Tariff> $tariffs): void` — заменить все текущие тарифы. Публикует событие `UserTariffsUpdated`.
* `addMessenger(Messenger $messenger, string $messengerId): void` — добавить мессенджер. Публикует событие `UserMessengersUpdated`.
* `removeMessenger(Messenger $messenger, string $messengerId): void` — удалить мессенджер. Публикует событие `UserMessengersUpdated`. Выбрасывает `MessengerNotFoundException` если мессенджер не найден. Выбрасывает `AtLeastOneMessengerRequiredException` при попытке удалить последний мессенджер (пользователь должен иметь минимум один).
* `getMessengers(): array` — получить список подключенных мессенджеров.
* `getTariffs(): array<Tariff>` — получить список тарифов.
* `getId(): UserId` — получить идентификатор пользователя.
* `getCreatedAt(): DateTimeImmutable` — получить дату создания.
* `getUpdatedAt(): DateTimeImmutable` — получить дату последнего обновления.

### UserMessenger

**Описание:**

Вложенная сущность внутри User. Не может существовать без User.

**Поля:**

* `userId` — идентификатор владельца ([UserId](./user-id.md))
* `messenger` — код мессенджера ([Messenger](../../shared/models/messenger.md))
* `messengerId` — идентификатор пользователя в конкретном мессенджере

### Репозиторий

**UserRepositoryInterface**

Методы:

* `findById(UserId $id): ?User` — получить пользователя по id.
* `save(User $user): void` — сохранить пользователя и все вложенные сущности (`tariffPlans`, `messengers`).
* `findByMessenger(Messenger $messenger, string $messengerId): ?User` — поиск пользователя по мессенджеру.

### События

При изменении инвариантов пользователя формируются и накапливаются следующие события

* [UserRegistered](../events/user-registered.md) - при первичном создании
* [UserTariffsUpdated](../events/user-tariffs-updated.md) - при изменении тарифов пользователя
* [UserMessengersUpdated](../events/user-messengers-updated) - при изменении мессенджеров пользователя

### Инварианты

- [x] User всегда имеет UserId
- [x] User всегда имеет **минимум один мессенджер** (не может быть без мессенджеров)
- [x] User всегда имеет минимум один тариф
- [x] Мессенджеры не дублируются (разные мессенджеры или разные ID)
- [x] updatedAt обновляется при любом изменении

### Хранение данных

**В доменной модели (Domain Layer)**:
- `$messengers` хранится как простой PHP `array<UserMessenger>` (в памяти)
- `$tariffs` хранится как `array<Tariff>` (enum объекты)

**В базе данных (Infrastructure Layer)**:
- Мессенджеры пользователя хранятся в отдельной таблице `user_messengers` с внешним ключом на `users.id`
- **Data Mapper pattern** преобразует таблицы в доменные объекты при загрузке и обратно при сохранении

### Замечания

* Модель является частью **Domain Layer**, не зависит от инфраструктуры или ORM.
* **Важно**: Существует два класса:
  - **`User`** — чистая доменная модель (Domain Layer), работает с простыми array структурами
  - **`UserEntity`** — Doctrine ORM сущность (Infrastructure Layer), имеет OneToMany отношения к таблицам `user_messengers`
  - Между ними используется **Data Mapper pattern** для преобразования
* **Инвариант минимум один мессенджер**: При попытке удалить последний мессенджер через `removeMessenger()` должно выбросить исключение (см. [AtLeastOneMessengerRequiredException](../exceptions/at-least-one-messenger-required-exception.md))
* Список `tariffs` хранит объекты [Tariff Enum](../../shared/models/tariff.md) из Shared Context. Это внутреннее представление; при передаче через границы контекста (события, API) тарифы конвертируются в строковые коды.
* `messengers` хранит объекты `UserMessenger`, которые описывают тип мессенджера и идентификатор пользователя в этом мессенджере.
* `createdAt` фиксируется при создании `User`, `updatedAt` обновляется при любых изменениях (тарифы, мессенджеры).
* Любые изменения тарифов или мессенджеров должны проходить через методы модели для сохранения инвариантов.
* `registerUser()` генерирует `UserId` внутри метода с помощью `UserId::generate()`.

## Связанные документы

* [Account Overview](../overview.md)

## Статус

* [x] Реализация доменных сущностей User и UserMessenger с базовыми полями. Публикация события `UserRegistered`
* [x] Реализация статического метода для регистрации пользователя
* [x] Добавление методов управления тарифами (`replaceTariffs`). Публикация события `UserTariffsUpdated` при изменении тарифов
* [x] Добавление методов управления мессенджерами (`addMessenger`, `removeMessenger`, `getMessengers`). Публикация события `UserMessengersUpdated` при изменении мессенджеров
* [x] Обновление `updatedAt` при любом изменении инвариантов
* [x] Добавление интерфейса репозитория `UserRepositoryInterface` с методами `findById`, `save`, `findByMessenger`
* [x] Добавление реализации интерфейса `UserRepositoryInterface` в инфраструктуре (Doctrine)
* [x] Data Mapper для преобразования Domain ↔ Entity
* [ ] Покрыто unit-тестами