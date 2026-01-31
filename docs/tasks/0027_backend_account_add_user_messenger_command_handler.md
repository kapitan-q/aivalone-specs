# Задача 0027: Реализация Application Service: AddUserMessengerCommandHandler

## Описание

Реализовать Application Service (Command Handler) для обработки команды добавления мессенджера к существующему пользователю.
Конвертирует входные данные и делегирует выполнение UserService, который отвечает за сохранение и публикацию событий.

## Требования

- [x] Класс должен быть расположен в `src/Context/Account/Application/CommandHandler/AddUserMessengerCommandHandler.php`
- [x] Быть обработчиком команды (Command Handler)
- [x] Обработать входные параметры и делегировать UserService

## Команда

```php
class AddUserMessengerCommand
{
    public function __construct(
        public readonly string $userId,        // UUID строка
        public readonly string $messengerCode, // 'TELEGRAM', 'WHATSAPP'
        public readonly string $messengerId    // ID в мессенджере
    ) {}
}
```

## Процесс обработки

```
AddUserMessengerCommand
    ↓
1. Валидация входных данных
    - $userId не пустой (формат UUID)
    - $messengerCode не пустой
    - $messengerId не пустой
    ↓
2. Конвертация входных данных
    - UserId::fromString($userId) → может выбросить InvalidUuidException
    - Messenger::fromCode($messengerCode) → может выбросить InvalidMessengerException
    ↓
3. Вызов UserService
    - userService->addMessenger($userId, $messenger, $messengerId)
    - Может выбросить:
      - UserNotFoundException
      - DuplicateMessengerException
    - UserService сохраняет пользователя и публикует события
    ↓
Результат: Мессенджер добавлен, события опубликованы (через UserService)
```

## Обработка ошибок

| Исключение                 | Статус | Сообщение                                      |
|----------------------------|--------|------------------------------------------------|
| InvalidUuidException       | 400    | Некорректный формат UserId                     |
| InvalidMessengerException  | 400    | Некорректный код мессенджера                   |
| UserNotFoundException      | 404    | Пользователь не найден                         |
| DuplicateMessengerException| 409    | Мессенджер уже привязан к другому пользователю |
| Другие исключения          | 500    | Internal Server Error                          |

## Сигнатура

```php
class AddUserMessengerCommandHandler
{
    public function __construct(
        private UserService $userService
    ) {}

    public function handle(AddUserMessengerCommand $command): void
    {
        // Обработка команды
    }
}
```

## Критерии готовности

- [x] Класс реализован с полной обработкой
- [x] Покрыт интеграционными тестами (минимум 4 теста: успех, пользователь не найден, дубликат мессенджера, некорректные данные)
- [x] Документация актуальна

## Зависимости

- Account Context: UserService
- Account Context: User
- Account Context: AddUserMessengerCommand
- Account Context: Исключения
- Shared Context: Messenger
- Shared Context: UserId

## Примечания

- Command должна быть в отдельном классе
- Handler должен быть зарегистрирован в Symfony DI с tagged service
- Валидация входных данных должна быть перед конвертацией типов
- Command Handler НЕ работает напрямую с Repository и EventBus - это делает UserService

## Статус

done
