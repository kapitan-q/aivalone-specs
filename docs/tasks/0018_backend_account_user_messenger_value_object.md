# Задача 0018: Реализация UserMessenger Value Object

## Описание

Реализовать Value Object `UserMessenger` для представления связи между пользователем и мессенджером.
Является вложенной сущностью внутри агрегата `User` и не может существовать отдельно.

## Требования

- [x] Класс должен быть расположен в `src/Context/Account/Domain/Model/UserMessenger.php`
- [x] Быть immutable (неизменяемым) Value Object
- [x] Содержать информацию о мессенджере и ID пользователя в этом мессенджере
- [x] Поддерживать сравнение по значению

## Структура

```php
class UserMessenger
{
    private Messenger $messenger;
    private string $messengerId;

    public function __construct(Messenger $messenger, string $messengerId);

    public function getMessenger(): Messenger;

    public function getMessengerId(): string;

    public function equals(UserMessenger $other): bool;
}
```

## Требования

- [x] Конструктор принимает `Messenger` и `messengerId`
- [x] `getMessenger()` - возврат мессенджера
- [x] `getMessengerId()` - возврат ID пользователя в мессенджере
- [x] `equals()` - сравнение двух объектов (одинаковый messenger и messengerId)

## Инварианты

- [x] Messenger всегда валиден (Messenger enum)
- [x] messengerId не может быть пустой строкой
- [x] Объект immutable после создания
- [x] Сравнение по значению (не по identity)

## Использование

```php
$telegram = Messenger::fromCode('TELEGRAM');
$userMessenger = new UserMessenger($telegram, '12345678');

echo $userMessenger->getMessenger()->code(); // 'TELEGRAM'
echo $userMessenger->getMessengerId(); // '12345678'

if ($userMessenger->equals($other)) {
    // одинаковые мессенджеры
}
```

## Критерии готовности

- [x] Класс реализован с требуемыми методами
- [x] Покрыт unit-тестами (минимум 4 теста)
- [x] Документация актуальна

## Зависимости

- Shared Context: Messenger

## Статус

done
