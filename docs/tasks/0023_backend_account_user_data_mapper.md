# Задача 0023: Реализация UserDataMapper

## Описание

Реализовать Mapper для преобразования между Domain Model `User` и ORM Entity `UserEntity`.
Используется Data Mapper pattern для чистого разделения слоев.

## Требования

- [x] Класс должен быть расположен в `src/Context/Account/Infrastructure/Persistence/Doctrine/Mapper/UserDataMapper.php`
- [x] Преобразовывать UserEntity → User (из БД в домен)
- [x] Преобразовывать User → UserEntity (из домена в БД)
- [x] Обрабатывать все поля и вложенные сущности

## Методы

### mapToDomainModel(UserEntity $entity): User

Преобразование ORM Entity в Domain Model.

**Логика**:
- Получить все поля из UserEntity
- Создать UserId из строки в БД
- Создать коллекцию UserMessenger объектов из поля messengers
- Создать объект User с этими данными
- **Важно**: Не должны публиковаться события (User только восстанавливается из БД)

**Возвращает**: Domain Model User

### mapToEntity(User $user): UserEntity

Преобразование Domain Model в ORM Entity.

**Логика**:
- Получить все данные из User
- Преобразовать UserId в строку для БД
- Преобразовать коллекцию UserMessenger в формат для хранения (объекты)
- Создать UserEntity с этими данными

**Возвращает**: ORM Entity UserEntity

## Особенности преобразования

### Messengers преобразование

**User Domain** → Массив UserMessenger объектов
**UserEntity** → JSON или массив структур

```php
// Domain Model
$messengers = [
    new UserMessenger(Messenger::TELEGRAM, '12345678'),
    new UserMessenger(Messenger::WHATSAPP, 'wa_12345678')
];

// ORM JSON
{
  "messengers": [
    {"messenger": "TELEGRAM", "messengerId": "12345678"},
    {"messenger": "WHATSAPP", "messengerId": "wa_12345678"}
  ]
}
```

### Tariffs преобразование

**User Domain** → Массив строк (коды тарифов)
**UserEntity** → JSON или простой массив

```php
// Domain Model
$tariffs = ['FREE', 'BASE'];

// ORM JSON
["FREE", "BASE"]
```

## Критерии готовности

- [x] Класс реализован с обоими методами
- [x] Покрыт unit-тестами (минимум 4 теста)
- [x] Документация актуальна

## Зависимости

- Account Context: User
- Account Context: UserId
- Account Context: UserMessenger
- Account Context: UserEntity
- Shared Context: Messenger

## Примечания

- Mapper не должен содержать бизнес-логику
- Преобразование должно быть полным (все данные должны передаваться)
- При преобразовании из Entity в User события не должны публиковаться
- При преобразовании из User в Entity события сохраняются отдельно (не в Entity)

## Статус

done
