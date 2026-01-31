# Задача 0033: Реализация Query Handler: GetUserByMessengerQueryHandler

## Описание

Реализовать Query Handler для получения пользователя по мессенджеру и его ID в этом мессенджере.
Используется для идентификации пользователя при получении сообщения от бота.

## Требования

- [x] Класс должен быть расположен в `src/Context/Account/Application/QueryHandler/GetUserByMessengerQueryHandler.php`
- [x] Быть обработчиком запроса (Query Handler CQRS)
- [x] Возвращать DTO с информацией о пользователе или null
- [x] Обрабатывать случаи ошибок корректно

## Query

```php
class GetUserByMessengerQuery
{
    public function __construct(
        public readonly string $messengerCode,  // 'TELEGRAM', 'WHATSAPP'
        public readonly string $messengerId     // ID пользователя в мессенджере
    ) {}
}
```

## DTO (Data Transfer Object)

```php
class UserDTO
{
    public function __construct(
        public readonly string $userId,
        public readonly array $messengers,   // [{'messenger': 'TELEGRAM', 'messengerId': '123'}]
        public readonly array $tariffs,       // ['FREE', 'BASE'] (строковые коды для передачи наружу)
        public readonly string $createdAt,   // ISO 8601
        public readonly string $updatedAt    // ISO 8601
    ) {}
}
```

## Процесс обработки

```
GetUserByMessengerQuery
    ↓
1. Валидация входных данных
    - $messengerCode не пустой
    - $messengerId не пустой
    ↓
2. Конвертация входных данных
    - Messenger::fromCode($messengerCode) → может выбросить InvalidMessengerException
    ↓
3. Получение пользователя из Repository
    - repository->findByMessenger($messenger, $messengerId)
    ↓
4. Проверка результата
    - Если null - вернуть null (пользователь не найден, можно создать новый)
    - Если найден - преобразовать в DTO
    ↓
Результат: UserDTO или null
```

## Сигнатура

```php
class GetUserByMessengerQueryHandler
{
    public function __construct(
        private UserRepositoryInterface $repository
    ) {}

    public function handle(GetUserByMessengerQuery $query): ?UserDTO
    {
        // Обработка запроса
    }
}
```

## Обработка ошибок

| Исключение            | Статус | Сообщение                    |
|----------------------|--------|------------------------------|
| InvalidMessengerException | 400 | Некорректный код мессенджера |

**Важно**: userNotFound не является ошибкой - возвращается null, что означает создание нового пользователя.

## Использование (пример из BOT Context)

```php
// В обработчике сообщения от Telegram
$query = new GetUserByMessengerQuery('TELEGRAM', $message->from->id);
$user = $queryBus->ask($query); // null или UserDTO

if ($user === null) {
    // Создать нового пользователя через RegisterUserCommand
    $command = new RegisterUserCommand('TELEGRAM', $message->from->id);
    $commandBus->dispatch($command);

    // Получить только что созданного пользователя
    $user = $queryBus->ask($query);
}

// Использовать $user
```

## Критерии готовности

- [x] Класс реализован с полной обработкой
- [x] DTO класс используется (переиспользуется из Задачи 22)
- [x] Покрыт unit-тестами (минимум 3 теста: найден, не найден, некорректный мессенджер)
- [x] Документация актуальна

## Зависимости

- Account Context: UserRepositoryInterface
- Account Context: User
- Account Context: GetUserByMessengerQuery
- Account Context: UserDTO
- Account Context: Исключения


## Примечания

- Query Handler не должен изменять состояние (read-only)
- Возврат null означает, что пользователь не существует (это нормально)
- Query Handler может быть использован из BOT Context для идентификации пользователя

## Статус

done
