# User Model Specification

## Назначение

Доменная модель пользователя платформы Aivalone. 
Отвечает за хранение идентификации пользователя, активных тарифов и подключенных мессенджеров. 
`User` является [AggregateRoot](../../shared/models/aggregate-root.md).

### Поля

| Поле          | Тип                    | Описание                                                                           |
| ------------- | ---------------------- | ---------------------------------------------------------------------------------- |
| `userId`      | `UserId`               | Уникальный идентификатор пользователя (UUID)                                       |
| `tariff`      | `array<string>`        | Список кодов активных тарифов пользователя (например, ['FREE', 'BASE'])            |
| `messengers`  | `array<UserMessenger>` | Список подключенных мессенджеров пользователя (Telegram, WhatsApp, Discord и т.п.) |
| `createdAt`   | `DateTimeImmutable`    | Дата создания пользователя                                                         |
| `updatedAt`   | `DateTimeImmutable`    | Дата последнего изменения пользователя                                             |

### Методы / Возможные действия

* `static registerUser(UserMessenger $messenger)` — создание нового пользователя.
* `addTariff(string $tariffCode): void` — добавить тариф к пользователю.
* `removeTariff(string $tariffCode): void` — удалить тариф у пользователя.
* `replaceTariffs(array $tariffCodes): void` — заменить все текущие тарифы.
* `addMessenger(UserMessenger $messenger): void` — добавить мессенджер.
* `removeMessenger(UserMessenger $messenger): void` — удалить мессенджер.
* `getMessengers(): array` — получить список подключенных мессенджеров.
* `getTariffPlans(): array` — получить список тарифных кодов.

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

### Замечания

* Модель является частью **Domain Layer**, не зависит от инфраструктуры или ORM.
* Список `tariffs` хранит только коды тарифов, фактическая логика доступа к функциям строится через отдельный сервис проверки прав.
* `messengers` хранит объекты `UserMessenger`, которые описывают тип мессенджера и идентификатор пользователя в этом мессенджере.
* `createdAt` фиксируется при создании `User`, `updatedAt` обновляется при любых изменениях (тарифы, мессенджеры).
* Любые изменения тарифов или мессенджеров должны проходить через методы модели для сохранения инвариантов.

## Связанные документы

* [Account Overview](../overview.md)

# Статус

* [] Реализация доменных сущностей User и UserMessenger с базовыми полями. Публикация события `UserRegistered`
* [] Реализация статического метода для регистрации пользователя
* [] Добавление методов управления тарифами (`addTariffPlan`, `removeTariffPlan`, `replaceTariffPlans`). Публикация события `UserTariffsUpdated` при изменении тарифов
* [] Добавление методов управления мессенджерами (`addMessenger`, `removeMessenger`, `getMessengers`). Публикация события `UserMessengersUpdated` при изменении мессенджеров
* [] Обновление `updatedAt` любом изменении инвариантов
* [] Добавление интерфейса репозитория `UserRepositoryInterface` с методами `findById`, `save`, `findByMessenger`. 
* [] Добавление реализации интерфейса `UserRepositoryInterface` в инфраструктуре