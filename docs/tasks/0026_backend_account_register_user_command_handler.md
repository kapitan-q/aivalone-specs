# Задача 0026: Реализация Application Service: RegisterUserCommandHandler

## Описание

Реализовать Application Service (Command Handler) для обработки команды регистрации нового пользователя.
Конвертирует входные данные и делегирует выполнение UserService, который отвечает за сохранение и публикацию событий.

## Требования

- [x] Класс должен быть расположен в `src/Context/Account/Application/CommandHandler/RegisterUserCommandHandler.php`
- [x] Быть обработчиком команды (Symfony Command Handler / CQRS pattern)
- [x] Обработать входные параметры и делегировать UserService

## Команда

```php
class RegisterUserCommand
{
    public function __construct(
        public readonly string $messengerCode,  // например, 'TELEGRAM'
        public readonly string $messengerId     // например, '123456789'
    ) {}
}
```

## Процесс обработки

```
RegisterUserCommand
    ↓
1. Валидация входных данных
    - messengerCode не пустой
    - messengerId не пустой
    ↓
2. Конвертация messengerCode в Messenger
    - Messenger::fromCode($messengerCode)
    - Может выбросить InvalidMessengerException
    ↓
3. Вызов UserService
    - userService->registerUser($messenger, $messengerId)
    - Может выбросить DuplicateUserException
    - UserService сохраняет пользователя и публикует события
    ↓
Результат: User создан, события опубликованы (через UserService)
```

## Обработка ошибок

| Исключение                 | Статус | Сообщение                                      |
|----------------------------|--------|------------------------------------------------|
| InvalidMessengerException  | 400    | Некорректный код мессенджера                   |
| DuplicateUserException     | 409    | Пользователь с этим мессенджером уже существует |
| Другие исключения          | 500    | Internal Server Error                          |

## Сигнатура

```php
class RegisterUserCommandHandler
{
    public function __construct(
        private UserService $userService
    ) {}

    public function handle(RegisterUserCommand $command): void
    {
        // Обработка команды
    }
}
```

## Возвращаемое значение

- **void** или **UserId** (в зависимости от паттерна)

Если возвращать UserId:
```php
public function handle(RegisterUserCommand $command): UserId
{
    // ...обработка...
    return $user->getId();
}
```

## Критерии готовности

- [x] Класс реализован с полной обработкой
- [x] Покрыт интеграционными тестами (минимум 3 теста: успех, дубликат, некорректный мессенджер)
- [x] Документация актуальна

## Зависимости

- Account Context: UserService
- Account Context: User
- Account Context: RegisterUserCommand
- Account Context: Исключения
- Shared Context: Messenger

## Примечания

- Команда должна быть в отдельном классе (Command Object)
- Handler должен быть зарегистрирован в Symfony DI с tagged service
- Command Handler НЕ работает напрямую с Repository и EventBus - это делает UserService

## Статус

done
