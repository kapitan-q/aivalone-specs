# Задача 07: Реализация DomainEventInterface

## Описание

Реализовать базовый контракт `DomainEventInterface` для доменных событий, которые фиксируют факты, произошедшие в доменной модели.

## Требования

- [x] Интерфейс должен быть расположен в `src/Context/Shared/Domain/Event/DomainEventInterface.php`
- [x] События должны быть immutable
- [x] События не должны знать об инфраструктуре
- [x] События должны быть сериализуемыми в array

## Интерфейс

```php
interface DomainEventInterface
{
    public function getEventName(): string;
    
    public function getOccurredAt(): DateTimeImmutable;
    
    public function getPayload(): array;
}
```

## Требования

- [x] `getEventName()` - возвращает уникальное имя события (например, 'user.registered')
- [x] `getOccurredAt()` - возвращает дату/время произошедшего события
- [x] `getPayload()` - возвращает сериализуемые данные события

## Соглашения

- [x] Имена событий должны быть в формате `domain.action` (например, `user.registered`, `user.messengers.updated`)
- [x] События именуются в прошедшем времени
- [x] Payload содержит только сериализуемые данные (строки, числа, массивы, null)
- [x] Payload НЕ содержит доменные объекты или ORM сущности

## Примеры событий

```
user.registered
user.tariffs.updated
user.messengers.updated
```

## Использование

```php
class UserRegistered implements DomainEventInterface
{
    public function __construct(
        private UserId $userId,
        private string $email,
        private DateTimeImmutable $occurredAt,
    ) {}
    
    public function getEventName(): string
    {
        return 'user.registered';
    }
    
    public function getOccurredAt(): DateTimeImmutable
    {
        return $this->occurredAt;
    }
    
    public function getPayload(): array
    {
        return [
            'userId' => $this->userId->toString(),
            'email' => $this->email,
        ];
    }
}
```

## Критерии готовности

- [x] Интерфейс реализован с требуемыми методами
- [ ] Покрыт unit-тестами (минимум одна имплементация)
- [ ] Документация актуальна

## Зависимости

Нет

## Статус

done
