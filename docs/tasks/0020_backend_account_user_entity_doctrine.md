# Задача 0020: Реализация UserEntity (Doctrine ORM)

## Описание

Реализовать Doctrine ORM Entity `UserEntity` для сохранения и загрузки пользователей из БД.
Используется Data Mapper pattern для преобразования между Domain Model и ORM Entity.

## Требования

- [x] Класс должен быть расположен в `src/Context/Account/Infrastructure/Persistence/Doctrine/Entity/UserEntity.php`
- [x] Быть аннотированной/атрибутивной Doctrine сущностью
- [x] Содержать все поля для хранения данных пользователя
- [x] Быть частью Infrastructure Layer (не должен использоваться в Domain)

## Структура сущности

```php
class UserEntity
{
    private string $id; // UUID как строка в БД
    private array $tariffs; // JSON или отдельная таблица
    private array $messengers; // Отдельная таблица или JSON
    private DateTimeImmutable $createdAt;
    private DateTimeImmutable $updatedAt;
}
```

## Требования к полям

| Поле        | Тип                | Doctrine Mapping      | БД тип                    | Описание                                  |
|-------------|--------------------|-----------------------|---------------------------|-------------------------------------------|
| `id`        | `string` (UUID)    | `@Id @Column(type="uuid")` | VARCHAR(36) / UUID     | Идентификатор пользователя                |
| `tariffs`   | `array<string>`    | `@Column(type="json")`    | JSON / TEXT           | Список кодов тарифов                      |
| `messengers`| Array of objects   | Отдельная таблица | Таблица      | Связь с мессенджерами                     |
| `createdAt` | `DateTimeImmutable` | `@Column(type="datetime_immutable")` | DATETIME | Дата создания  |
| `updatedAt` | `DateTimeImmutable` | `@Column(type="datetime_immutable")` | DATETIME | Дата обновления |

## Варианты реализации

Создать таблицу `user_messengers` с foreign key на `users`:

```sql
CREATE TABLE user_messengers (
    id INT AUTO_INCREMENT PRIMARY KEY,
    user_id VARCHAR(36) NOT NULL,
    messenger VARCHAR(20) NOT NULL,
    messenger_id VARCHAR(255) NOT NULL,
    FOREIGN KEY (user_id) REFERENCES users(id) ON DELETE CASCADE,
    UNIQUE KEY (messenger, messenger_id)
);
```

## Критерии готовности

- [x] Класс реализован с требуемыми полями
- [x] Doctrine mapping конфигурирован (атрибуты или XML)
- [x] Определена структура таблицы в БД
- [x] Документация актуальна

## Зависимости

- Doctrine ORM
- Doctrine DBAL

## Примечания

- UserEntity НЕ должен наследоваться от User Domain Model
- UserEntity используется только в Infrastructure Layer
- Data Mapper будет преобразовывать между UserEntity ↔ User
- Создание миграции Doctrine для создания таблиц будет в отдельной задаче

## Статус

done
