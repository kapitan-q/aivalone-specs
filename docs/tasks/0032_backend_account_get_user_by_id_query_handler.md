# Задача 0032: Реализация Query Handler: GetUserByIdQueryHandler

## Описание

Реализовать Query Handler для получения информации о пользователе по ID.
Используется для предоставления информации другим контекстам или BOT слою.

## Требования

- [x] Класс должен быть расположен в `src/Context/Account/Application/QueryHandler/GetUserByIdQueryHandler.php`
- [x] Быть обработчиком запроса (Query Handler CQRS)
- [x] Возвращать DTO с информацией о пользователе
- [x] Обрабатывать случаи ошибок корректно

## Query

```php
class GetUserByIdQuery
{
    public function __construct(
        public readonly string $userId  // UUID строка
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
GetUserByIdQuery
    ↓
1. Валидация входных данных
    - $userId не пустой (формат UUID)
    ↓
2. Конвертация входных данных
    - UserId::fromString($userId) → может выбросить InvalidUuidException
    ↓
3. Получение пользователя из Repository
    - repository->findById($userId)
    ↓
4. Проверка результата
    - Если null - выбросить UserNotFoundException
    ↓
5. Преобразование в DTO
    - Создать UserDTO с данными пользователя
    ↓
Результат: UserDTO с информацией о пользователе
```

## Сигнатура

```php
class GetUserByIdQueryHandler
{
    public function __construct(
        private UserRepositoryInterface $repository
    ) {}

    public function handle(GetUserByIdQuery $query): UserDTO
    {
        // Обработка запроса
    }
}
```

## Обработка ошибок

| Исключение            | Статус | Сообщение                    |
|----------------------|--------|------------------------------|
| InvalidUuidException | 400    | Некорректный формат UserId   |
| UserNotFoundException| 404    | Пользователь не найден       |

## Критерии готовности

- [x] Класс реализован с полной обработкой
- [x] DTO класс создан
- [x] Покрыт unit-тестами (минимум 3 теста: успех, не найден, некорректный ID)
- [x] Документация актуальна

## Зависимости

- Account Context: UserRepositoryInterface
- Account Context: User
- Account Context: GetUserByIdQuery
- Account Context: UserDTO
- Account Context: Исключения
- Shared Context: UserId

## Примечания

- Query Handler не должен изменять состояние (read-only)
- DTO предотвращает утечку доменных объектов
- Query Handler может кэшировать результаты (на уровне инфраструктуры)

## Статус

done
