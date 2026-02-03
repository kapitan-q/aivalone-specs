# NotificationEndpoint Model Specification

## Назначение

Доменная модель `NotificationEndpoint` представляет унифицированную точку доставки сообщений пользователю. Инкапсулирует внешние идентификаторы мессенджеров (chat_id, phone_id, channel_id) и предоставляет единый интерфейс для отправки сообщений.

`NotificationEndpoint` является AggregateRoot.

## Атрибуты

| Поле               | Тип                    | Описание                                                    |
| ------------------ | ---------------------- | ----------------------------------------------------------- |
| `endpointId`       | `EndpointId`           | Уникальный идентификатор (UUID)                             |
| `userId`           | `UserId`               | Идентификатор пользователя (из Account Context)             |
| `messenger`        | `Messenger`            | Тип мессенджера (telegram, whatsapp, discord)               |
| `externalTargetId` | `string`               | Внешний идентификатор (chat_id, phone_id и т.д.)            |
| `status`           | `EndpointStatus`       | Статус (ACTIVE, REVOKED)                                    |
| `createdAt`        | `DateTimeImmutable`    | Дата создания                                               |
| `revokedAt`        | `?DateTimeImmutable`   | Дата отзыва (null если активен)                             |

## Методы

### static create(UserId $userId, Messenger $messenger, string $externalTargetId): self

Создание нового endpoint.

**Логика**:
- Генерирует новый EndpointId (UUID)
- Устанавливает status = ACTIVE
- createdAt = now
- revokedAt = null
- Регистрирует событие `NotificationEndpointRegistered`

**Параметры**:
- `$userId` — идентификатор пользователя из Account Context
- `$messenger` — тип мессенджера
- `$externalTargetId` — внешний идентификатор в мессенджере

**Возвращает**: Новый объект `NotificationEndpoint`

---

### revoke(): void

Отзыв endpoint.

**Логика**:
- Проверяет, что статус = ACTIVE
- Устанавливает status = REVOKED
- revokedAt = now
- Регистрирует событие `NotificationEndpointRevoked`

**Выбрасывает**: DomainException если статус не ACTIVE

---

### Методы доступа

```php
public function getId(): EndpointId
public function getUserId(): UserId
public function getMessenger(): Messenger
public function getExternalTargetId(): string  // ВАЖНО: только для использования внутри Bot Context
public function getStatus(): EndpointStatus
public function getCreatedAt(): DateTimeImmutable
public function getRevokedAt(): ?DateTimeImmutable
public function isActive(): bool
public function isRevoked(): bool
```

## Инварианты

- [x] NotificationEndpoint всегда имеет EndpointId
- [x] NotificationEndpoint всегда связан с валидным userId
- [x] externalTargetId никогда не передаётся за пределы Bot Context
- [x] externalTargetId не логируется в domain events
- [x] Один пользователь может иметь несколько endpoints (в будущем)
- [x] В MVP: 1 endpoint = 1 messenger = 1 user

## Репозиторий

**NotificationEndpointRepositoryInterface**

Методы:

* `findById(EndpointId $id): ?NotificationEndpoint`
* `findByUserId(UserId $userId): array<NotificationEndpoint>`
* `findByUserIdAndMessenger(UserId $userId, Messenger $messenger): ?NotificationEndpoint`
* `findByExternalTarget(Messenger $messenger, string $externalTargetId): ?NotificationEndpoint`
* `save(NotificationEndpoint $endpoint): void`

## События

При изменении инвариантов формируются следующие события:

* [NotificationEndpointRegistered](../events/notification-endpoint-registered.md) - при создании
* [NotificationEndpointRevoked](../events/notification-endpoint-revoked.md) - при отзыве

## Безопасность

* `externalTargetId` никогда не включается в payload событий
* `externalTargetId` не логируется
* Доступ к `getExternalTargetId()` разрешён только для Messenger Adapters

## Связанные документы

* [EndpointId Value Object](endpoint-id.md)
* [EndpointStatus Enum](endpoint-status-enum.md)
* [Bot Context Overview](../overview.md)

## Статус реализации

* [ ] Класс EndpointId создан
* [ ] Класс NotificationEndpoint создан с базовой структурой
* [ ] Enum EndpointStatus создан
* [ ] Метод create() реализован
* [ ] Метод revoke() реализован
* [ ] События регистрируются корректно
* [ ] Интерфейс репозитория создан
* [ ] Doctrine Entity и Repository реализованы
* [ ] Unit тесты написаны
* [ ] Integration тесты пройдены
