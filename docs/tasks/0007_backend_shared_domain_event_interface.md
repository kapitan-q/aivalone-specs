# Задача 07: Реализация DomainEventInterface

## Описание

Реализовать базовый контракт `DomainEventInterface` для доменных событий, которые фиксируют факты, произошедшие в доменной модели.

## Требования

- [ ] Интерфейс должен быть расположен в `src/Context/Shared/Domain/Event/DomainEventInterface.php`
- [ ] События должны быть immutable
- [ ] События не должны знать об инфраструктуре
- [ ] События должны быть сериализуемыми в array

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

- [ ] `getEventName()` - возвращает уникальное имя события (например, 'user.registered')
- [ ] `getOccurredAt()` - возвращает дату/время произошедшего события
- [ ] `getPayload()` - возвращает сериализуемые данные события

## Соглашения

- [ ] Имена событий должны быть в формате `domain.action` (например, `user.registered`, `user.messengers.updated`)
- [ ] События именуются в прошедшем времени
- [ ] Payload содержит только сериализуемые данные (строки, числа, массивы, null)
- [ ] Payload НЕ содержит доменные объекты или ORM сущности

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

not-started
