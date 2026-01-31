# Задача 08: Реализация AggregateRoot

## Описание

Реализовать базовый абстрактный класс `AggregateRoot` для всех агрегатов в системе. Класс должен обеспечивать накопление и управление доменными событиями.

## Требования

- [x] Класс должен быть расположен в `src/Context/Shared/Domain/Model/AggregateRoot.php`
- [x] Быть абстрактным классом
- [x] Не знать о способе публикации событий (EventBus, Message Broker и т.д.)
- [x] Гарантировать неизменяемость уже опубликованных событий

## Интерфейс

```php
abstract class AggregateRoot
{
    protected function recordEvent(DomainEventInterface $event): void;
    
    public function pullEvents(): array;
    
    public function hasEvents(): bool;
}
```

## Требования

- [x] `recordEvent()` - защищённый метод для регистрации события внутри агрегата
- [x] `pullEvents()` - возвращает ВСЕ накопленные события и ОЧИЩАЕТ внутреннее хранилище
- [x] `hasEvents()` - проверяет наличие непубликованных событий
- [x] Каждое событие может быть опубликовано только один раз

## Жизненный цикл событий

```
1. Агрегат изменяет состояние
2. Агрегат вызывает recordEvent(...)
3. Application Service сохраняет агрегат в БД
4. Application Service вызывает pullEvents()
5. События публикуются в EventBus
```

## Использование

```php
class User extends AggregateRoot
{
    private UserId $id;
    
    public static function register(UserId $id, string $email): self
    {
        $user = new self();
        $user->id = $id;
        
        $user->recordEvent(new UserRegistered(
            $id,
            $email,
            new DateTimeImmutable()
        ));
        
        return $user;
    }
}

// В Application Service
$user = User::register($userId, $email);
$this->userRepository->save($user);

$events = $user->pullEvents(); // [ UserRegistered event ]
$this->eventBus->publish(...$events);

$user->hasEvents(); // false (события очищены)
```

## Хранение событий

- [x] Внутренний массив `$recordedEvents` для хранения событий
- [x] События добавляются только внутри агрегата через `recordEvent()`
- [x] События недоступны для прямой модификации извне

## Инварианты

- [x] События накапливаются в порядке их возникновения
- [x] После вызова `pullEvents()` внутреннее хранилище пусто
- [x] `pullEvents()` может быть вызван многократно (последующие вызовы возвращают пустой массив)

## Критерии готовности

- [x] Абстрактный класс реализован с требуемыми методами
- [ ] Покрыт unit-тестами (конкретная реализация через мок-класс, минимум 4 теста)
- [ ] Документация актуальна

## Зависимости

- [Задача 07: DomainEventInterface](./backend_shared_07_domain_event_interface.md)

## Статус

done
