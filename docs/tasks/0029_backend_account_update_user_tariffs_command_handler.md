# Задача 0029: Реализация Application Service: UpdateUserTariffsCommandHandler

## Описание

Реализовать Application Service (Command Handler) для обработки команды обновления тарифов пользователя.
Эта команда может быть вызвана как от Application Layer, так и при получении события из Billing Context.
Конвертирует входные данные и делегирует выполнение UserService, который отвечает за сохранение и публикацию событий.

## Требования

- [x] Класс должен быть расположен в `src/Context/Account/Application/CommandHandler/UpdateUserTariffsCommandHandler.php`
- [x] Быть обработчиком команды (Command Handler)
- [x] Обработать входные параметры и делегировать UserService

## Команда

```php
class UpdateUserTariffsCommand
{
    public function __construct(
        public readonly string $userId,     // UUID строка
        public readonly array $tariffCodes  // ['FREE', 'BASE']
    ) {}
}
```

## Процесс обработки

```
UpdateUserTariffsCommand
    ↓
1. Валидация входных данных
    - $userId не пустой (формат UUID)
    - $tariffCodes не пустой массив
    ↓
2. Конвертация входных данных
    - UserId::fromString($userId) → может выбросить InvalidUuidException
    - Конвертация $tariffCodes (строки) в объекты Tariff через Tariff::fromCode() → может выбросить TariffValidationException для некорректных кодов
    ↓
3. Вызов UserService с Tariff объектами
    - userService->updateTariffs($userId, $tariffs) → может выбросить UserNotFoundException
    - UserService сохраняет пользователя и публикует события
    ↓
Результат: Тарифы обновлены (с Tariff объектами внутри), события опубликованы (через UserService)
```

## Обработка ошибок

| Исключение                 | Статус | Сообщение                                      |
|----------------------------|--------|------------------------------------------------|
| InvalidUuidException       | 400    | Некорректный формат UserId                     |
| TariffValidationException  | 400    | Некорректные тарифные коды                     |
| UserNotFoundException      | 404    | Пользователь не найден                         |
| Другие исключения          | 500    | Internal Server Error                          |

## Сигнатура

```php
class UpdateUserTariffsCommandHandler
{
    public function __construct(
        private UserService $userService
    ) {}

    public function handle(UpdateUserTariffsCommand $command): void
    {
        // Обработка команды
    }
}
```

## Примечание: Использование из Event Handler

Этот же Command Handler может быть использован из Event Handler при получении события `UserTariffsUpdatedEvent` из Billing Context:

```php
class UserTariffsUpdatedEventHandler
{
    public function __construct(private CommandBusInterface $commandBus) {}

    public function handle(UserTariffsUpdatedEvent $event): void
    {
        // Event содержит массив Tariff объектов, конвертируем в строки для Command
        $tariffCodes = array_map(
            fn(Tariff $tariff) => $tariff->code(),
            $event->getTariffs()
        );

        $command = new UpdateUserTariffsCommand(
            $event->getUserId()->toString(),
            $tariffCodes
        );

        $this->commandBus->dispatch($command);
    }
}
```

## Критерии готовности

- [x] Класс реализован с полной обработкой
- [x] Конвертирует строковые коды в Tariff объекты перед вызовом UserService
- [x] Обрабатывает некорректные тарифные коды выбросом TariffValidationException
- [x] Покрыт интеграционными тестами (минимум 4 теста: успех, пользователь не найден, некорректные тарифы, пустой список)
- [x] Документация актуальна

## Зависимости

- Account Context: UserService
- Account Context: User
- Account Context: UpdateUserTariffsCommand
- Account Context: Исключения
- Shared Context: Tariff
- Shared Context: UserId

## Примечания

- Command должна быть в отдельном классе
- Handler должен быть зарегистрирован в Symfony DI с tagged service
- Валидация входных данных должна быть перед конвертацией типов
- Интеграция с Billing Context будет в отдельной задаче
- Command Handler НЕ работает напрямую с Repository и EventBus - это делает UserService

## Статус

done
