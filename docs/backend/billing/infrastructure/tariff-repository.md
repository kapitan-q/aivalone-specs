# TariffRepository

## Описание

Репозиторий для работы с агрегатом `Tariff`. Предоставляет методы для поиска и сохранения тарифов.

## Расположение

```
# Интерфейс (Domain Layer)
src/Context/Billing/Domain/Repository/TariffRepositoryInterface.php

# Реализация (Infrastructure Layer)
src/Context/Billing/Infrastructure/Persistence/DoctrineTariffRepository.php
```

## Интерфейс

```php
namespace App\Context\Billing\Domain\Repository;

use App\Context\Billing\Domain\Model\Tariff;
use App\Context\Billing\Domain\Model\TariffId;
use App\Context\Shared\Domain\Model\Tariff as TariffEnum;

interface TariffRepositoryInterface
{
    /**
     * Найти тариф по ID
     */
    public function findById(TariffId $id): ?Tariff;

    /**
     * Найти тариф по коду (enum)
     */
    public function findByCode(TariffEnum $code): ?Tariff;

    /**
     * Получить все тарифы
     */
    public function findAll(): array;

    /**
     * Сохранить тариф (insert или update)
     */
    public function save(Tariff $tariff): void;

    /**
     * Удалить тариф
     */
    public function remove(Tariff $tariff): void;
}
```

## Методы

### findById(TariffId $id): ?Tariff

Находит тариф по уникальному идентификатору.

**Параметры**:
- `$id` — TariffId (UUID)

**Возвращает**: `Tariff` или `null` если не найден

---

### findByCode(TariffEnum $code): ?Tariff

Находит тариф по коду (FREE, BASE, PRO, ENTERPRISE).

**Параметры**:
- `$code` — Tariff enum из Shared Context

**Возвращает**: `Tariff` или `null` если не найден

**Использование**: Основной метод для поиска тарифа при создании подписки

---

### findAll(): array

Получает все доступные тарифы.

**Возвращает**: `array<Tariff>` — массив всех тарифов

**Использование**: GetTariffList Query, REST API

---

### save(Tariff $tariff): void

Сохраняет тариф (insert или update).

**Параметры**:
- `$tariff` — объект Tariff

**Логика**:
- Если тариф новый (не существует в БД) — INSERT
- Если тариф существует — UPDATE
- Сохраняет все связанные TariffOption

---

### remove(Tariff $tariff): void

Удаляет тариф из базы данных.

**Параметры**:
- `$tariff` — объект Tariff

**Логика**:
- Удаляет тариф и все связанные TariffOption (CASCADE)

## Doctrine реализация

```php
namespace App\Context\Billing\Infrastructure\Persistence;

use App\Context\Billing\Domain\Model\Tariff;
use App\Context\Billing\Domain\Model\TariffId;
use App\Context\Billing\Domain\Repository\TariffRepositoryInterface;
use App\Context\Shared\Domain\Model\Tariff as TariffEnum;
use Doctrine\ORM\EntityManagerInterface;

final class DoctrineTariffRepository implements TariffRepositoryInterface
{
    public function __construct(
        private readonly EntityManagerInterface $em,
    ) {}

    public function findById(TariffId $id): ?Tariff
    {
        return $this->em->find(Tariff::class, $id->toString());
    }

    public function findByCode(TariffEnum $code): ?Tariff
    {
        return $this->em->getRepository(Tariff::class)
            ->findOneBy(['code' => $code->code()]);
    }

    public function findAll(): array
    {
        return $this->em->getRepository(Tariff::class)
            ->findBy([], ['priority' => 'ASC']);
    }

    public function save(Tariff $tariff): void
    {
        $this->em->persist($tariff);
        $this->em->flush();
    }

    public function remove(Tariff $tariff): void
    {
        $this->em->remove($tariff);
        $this->em->flush();
    }
}
```

## Статус реализации

- [ ] Интерфейс создан в Domain Layer
- [ ] Doctrine реализация создана
- [ ] Все методы реализованы
- [ ] Маппинг Doctrine настроен
- [ ] Unit тесты написаны (6+ тестов)
- [ ] Integration тесты пройдены
- [ ] Сервис зарегистрирован в DI контейнере

## Связанные сущности

- [Tariff Aggregate Root](../models/tariff-aggregate-root.md)
- [TariffId Value Object](../models/tariff-id-value-object.md)
- [UserSubscriptionRepository](./user-subscription-repository.md)
- [Billing Context Overview](../overview.md)
