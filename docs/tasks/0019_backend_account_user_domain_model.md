# Задача 0019: Реализация User Domain Model

## Описание

Реализовать доменную модель `User` как агрегат (AggregateRoot) для управления пользователями, их тарифами и мессенджерами.

## Требования

- [x] Класс должен быть расположен в `src/Context/Account/Domain/Model/User.php`
- [x] Наследует от `AggregateRoot` (Shared Context)
- [x] Содержит все поля согласно спецификации
- [x] Реализует все методы для управления тарифами и мессенджерами

## Основные методы

### static registerUser(Messenger $messenger, string $messengerId): self

Создание нового пользователя.

**Логика**:
- Инициализирует пользователя с начальным тарифом `Tariff::FREE`
- Записывает событие `UserRegistered` в агрегат

**Возвращает**: Новый объект `User` (не сохраненный)

### addMessenger(Messenger $messenger, string $messengerId): void

Добавление мессенджера к пользователю.

**Логика**:
- Создает новый объект `UserMessenger`
- Добавляет его в коллекцию `$messengers`
- Обновляет `updatedAt`
- Записывает событие `UserMessengersUpdated`

**Выбрасывает**: Нет (проверка уникальности на уровне Application Service)

### removeMessenger(Messenger $messenger, string $messengerId): void

Удаление мессенджера из пользователя.

**Логика**:
- Находит мессенджер в коллекции по `messenger` и `messengerId`
- Если не найден - выбрасывает `MessengerNotFoundException`
- Если это последний (единственный) мессенджер - выбрасывает `AtLeastOneMessengerRequiredException` (нарушение инварианта)
- Удаляет найденный мессенджер из коллекции
- Обновляет `updatedAt`
- Записывает событие `UserMessengersUpdated`

### replaceTariffs(array<Tariff> $tariffs): void

Замена всех тарифов пользователя на новый список.

**Логика**:
- Заменяет текущий список `$tariffs` на переданный массив объектов Tariff
- Обновляет `updatedAt`
- Записывает событие `UserTariffsUpdated`

**Примечание**: Ожидает уже сконвертированные объекты Tariff (конвертация происходит на уровне Command Handler)

### Методы доступа

- `getId(): UserId` - получить идентификатор
- `getMessengers(): array<UserMessenger>` - получить список мессенджеров
- `getTariffs(): array<Tariff>` - получить список объектов Tariff
- `getCreatedAt(): DateTimeImmutable`
- `getUpdatedAt(): DateTimeImmutable`

## Инварианты

- [x] User всегда имеет UserId
- [x] User всегда имеет минимум один мессенджер
- [x] User всегда имеет минимум один тариф
- [x] Мессенджеры не дублируются (разные мессенджеры или разные ID)
- [x] updatedAt обновляется при любом изменении

## Использование

```php
// Регистрация нового пользователя
$telegram = Messenger::fromCode('TELEGRAM');
$user = User::registerUser($telegram, '12345678');

// Добавление мессенджера
$whatsapp = Messenger::fromCode('WHATSAPP');
$user->addMessenger($whatsapp, 'wa_12345678');

// Обновление тарифов (конвертируем строки в Tariff объекты)
$user->replaceTariffs([Tariff::FREE, Tariff::BASE]);

// Получение событий
$events = $user->pullEvents(); // [UserRegistered, UserMessengersUpdated, UserTariffsUpdated]
```

## Критерии готовности

- [x] Класс реализован со всеми требуемыми методами
- [x] Покрыт unit-тестами (минимум 10 тестов)
- [x] Документация актуальна

## Зависимости

- Shared Context: AggregateRoot
- Shared Context: DomainEventInterface
- Shared Context: Messenger
- Shared Context: Tariff
- Account Context: UserId
- Account Context: UserMessenger
- Account Context: Исключения (MessengerNotFoundException, AtLeastOneMessengerRequiredException)
- Account Context: События (UserRegistered, UserMessengersUpdated, UserTariffsUpdated)

## Статус

done
