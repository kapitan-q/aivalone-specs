# Задача 0028: Реализация Application Service: RemoveUserMessengerCommandHandler

## Описание

Реализовать Application Service (Command Handler) для обработки команды удаления мессенджера у пользователя.
Конвертирует входные данные и делегирует выполнение UserService, который отвечает за сохранение и публикацию событий.

## Требования

- [x] Класс должен быть расположен в `src/Context/Account/Application/CommandHandler/RemoveUserMessengerCommandHandler.php`
- [x] Быть обработчиком команды (Command Handler)
- [x] Обработать входные параметры и делегировать UserService

## Команда

```php
class RemoveUserMessengerCommand
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
RemoveUserMessengerCommand
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
    - userService->removeMessenger($userId, $messenger, $messengerId)
    - Может выбросить:
      - UserNotFoundException
      - MessengerNotFoundException
      - AtLeastOneMessengerRequiredException
    - UserService сохраняет пользователя и публикует события
    ↓
Результат: Мессенджер удален, события опубликованы (через UserService)
```

## Обработка ошибок

| Исключение                 | Статус | Сообщение                                      |
|----------------------------|--------|------------------------------------------------|
| InvalidUuidException       | 400    | Некорректный формат UserId                     |
| InvalidMessengerException  | 400    | Некорректный код мессенджера                   |
| UserNotFoundException      | 404    | Пользователь не найден                         |
| MessengerNotFoundException | 404    | Мессенджер не привязан к пользователю          |
| Другие исключения          | 500    | Internal Server Error                          |

## Сигнатура

```php
class RemoveUserMessengerCommandHandler
{
    public function __construct(
        private UserService $userService
    ) {}

    public function handle(RemoveUserMessengerCommand $command): void
    {
        // Обработка команды
    }
}
```

## Критерии готовности

- [x] Класс реализован с полной обработкой
- [x] Покрыт интеграционными тестами (минимум 4 теста: успех, пользователь не найден, мессенджер не найден, некорректные данные)
- [x] Документация актуальна

## Зависимости

- Account Context: UserService
- Account Context: User
- Account Context: RemoveUserMessengerCommand
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
