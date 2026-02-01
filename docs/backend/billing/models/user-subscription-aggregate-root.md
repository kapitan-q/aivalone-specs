# UserSubscription Domain Model (AggregateRoot)

## Описание

Доменная модель `UserSubscription` управляет информацией о подписке пользователя на определенный тариф. Является AggregateRoot и отвечает за отслеживание статуса подписки и генерацию соответствующих событий.

## Атрибуты

- `id: UserSubscriptionId` — уникальный идентификатор (UUID)
- `userId: UserId` — идентификатор пользователя (из Account Context)
- `tariffId: TariffId` — идентификатор тарифа
- `period: SubscriptionPeriod | null` — период подписки (MONTH, YEAR); null для бессрочных (FREE)
- `createdAt: DateTimeImmutable` — дата начала подписки
- `validUntil: DateTimeImmutable | null` — дата окончания (null = бессрочная)
- `status: SubscriptionStatus` — статус (ACTIVE, EXPIRED, CANCELLED)
- `previousSubscriptionId: UserSubscriptionId | null` — ID предыдущей подписки (для продлений)

## Основные методы

### static create(UserId $userId, TariffId $tariffId, ?SubscriptionPeriod $period = null, ?UserSubscriptionId $previousSubscriptionId = null): self

Создание новой подписки.

**Логика**:
- Генерирует новый UserSubscriptionId (UUID)
- Если period указан — вычисляет validUntil = now + period
- Если period = null — validUntil = null (бессрочная подписка, например FREE)
- Устанавливает status = ACTIVE
- Сохраняет previousSubscriptionId (если это продление)
- Регистрирует событие `UserSubscriptionUpdated` (action: ADDED)
- Возвращает новый объект

**Параметры**:
- `$userId` — идентификатор пользователя
- `$tariffId` — идентификатор тарифа
- `$period` — период подписки (опционально, null для бессрочных)
- `$previousSubscriptionId` — ID предыдущей подписки при продлении (опционально)

**Выбрасывает**: ValidationException если period невалиден

**Возвращает**: Новый объект `UserSubscription`

---

### static createFree(UserId $userId, TariffId $freeTariffId): self

Создание бессрочной FREE подписки.

**Логика**:
- Вызывает `create()` с period = null
- Специальный фабричный метод для читаемости

**Параметры**:
- `$userId` — идентификатор пользователя
- `$freeTariffId` — идентификатор FREE тарифа

**Возвращает**: Новый объект `UserSubscription` с validUntil = null

---

### expire(): void

Истечение подписки.

**Логика**:
- Проверяет, что статус = ACTIVE
- Устанавливает status = EXPIRED
- Обновляет `updatedAt`
- Регистрирует событие `SubscriptionExpired`

---

### cancel(): void

Отмена подписки.

**Логика**:
- Проверяет, что статус = ACTIVE
- Устанавливает status = CANCELLED
- Обновляет `updatedAt`
- Регистрирует событие `UserSubscriptionUpdated` (action: REMOVED)

---

### markExpiringSoon(int $daysUntilExpiration): void

Отмечает подписку как скоро истекающую и генерирует событие.

**Логика**:
- Проверяет, что подписка активна (status = ACTIVE)
- Проверяет, что daysUntilExpiration валидное значение (целое число)
- Регистрирует событие `SubscriptionExpiringSoon`

**Параметры**:
- `$daysUntilExpiration` — количество дней до истечения (например 7, 3 или 1)

**Выбрасывает**: DomainException если статус не ACTIVE или дни невалидны

---

### isRenewal(): bool

Проверяет, является ли подписка продлением.

**Возвращает**: `true` если previousSubscriptionId не null

---

### Методы доступа

```php
public function getId(): UserSubscriptionId
public function getUserId(): UserId
public function getTariffId(): TariffId
public function getPeriod(): SubscriptionPeriod|null
public function getCreatedAt(): DateTimeImmutable
public function getValidUntil(): DateTimeImmutable|null
public function getStatus(): SubscriptionStatus
public function getUpdatedAt(): DateTimeImmutable
public function getPreviousSubscriptionId(): UserSubscriptionId|null
public function isActive(): bool
public function isExpired(): bool
public function isCancelled(): bool
public function isPermanent(): bool  // validUntil === null
public function isRenewal(): bool
```

## Инварианты

- UserSubscription всегда имеет SubscriptionId
- UserSubscription всегда связан с валидным userId и tariffId
- Если period указан, validUntil вычисляется автоматически
- Если period = null, validUntil = null (бессрочная подписка)
- status обновляется при изменении состояния подписки
- События регистрируются при создании, истечении и отмене
- У пользователя не может быть двух активных подписок на **один и тот же** тариф
- Пользователь может иметь активные подписки на **разные** тарифы (например, FREE + PRO)

## Статус реализации

- [ ] Класс создан с базовой структурой
- [ ] Атрибут period добавлен
- [ ] Атрибут previousSubscriptionId добавлен
- [ ] Метод createFree реализован
- [ ] Все методы реализованы
- [ ] Все валидации работают
- [ ] События регистрируются корректно
- [ ] Методы доступа реализованы
- [ ] Unit тесты написаны (15+ тестов)
- [ ] Integration тесты пройдены
- [ ] Код соответствует стилю проекта
- [ ] Документация актуальна

## Связанные сущности

- [UserSubscriptionId Value Object](user-subscription-id-value-object.md)
- [SubscriptionStatus Enum](../enums/subscription-status.md)
- [SubscriptionPeriod Enum](../enums/subscription-period.md)
- [UserSubscriptionUpdated Event](../events/user-subscription-updated-event.md)
- [SubscriptionExpired Event](../events/subscription-expired-event.md)
- [SubscriptionExpiringSoon Event](../events/subscription-expiring-soon-event.md)
- [SubscriptionRenewed Event](../events/subscription-renewed-event.md)
- [RenewSubscription Process](../processes/renew-subscription-process.md)
- [Billing Context Overview](../overview.md)
