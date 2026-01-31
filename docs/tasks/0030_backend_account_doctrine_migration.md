# Задача 0030: Создание Doctrine миграции для Account Context

## Описание

Создать Doctrine миграцию для создания таблиц и структуры БД Account Context.

## Требования

- [x] Файл миграции должен быть расположен в `migrations/`
- [x] Создать таблицу `users` для хранения User агрегатов
- [x] Создать индексы для оптимизации поиска

## Структура таблицы users

```sql
CREATE TABLE users (
    id CHAR(36) NOT NULL PRIMARY KEY COMMENT 'UUID идентификатор пользователя',
    tariffs JSON NOT NULL COMMENT 'JSON массив кодов тарифов: ["FREE", "BASE"]',
    created_at DATETIME NOT NULL COMMENT 'Дата создания пользователя',
    updated_at DATETIME NOT NULL COMMENT 'Дата последнего обновления',
    INDEX idx_created_at (created_at),
    INDEX idx_updated_at (updated_at)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_unicode_ci;
```

## Структура таблицы user_messengers

```sql
CREATE TABLE user_messengers (
    id INT AUTO_INCREMENT PRIMARY KEY,
    user_id CHAR(36) NOT NULL,
    messenger VARCHAR(20) NOT NULL COMMENT 'Код мессенджера: TELEGRAM, WHATSAPP',
    messenger_id VARCHAR(255) NOT NULL COMMENT 'ID пользователя в мессенджере',
    created_at DATETIME NOT NULL,
    FOREIGN KEY (user_id) REFERENCES users(id) ON DELETE CASCADE,
    UNIQUE KEY uq_messenger_id (messenger, messenger_id),
    INDEX idx_user_id (user_id),
    INDEX idx_messenger (messenger)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_unicode_ci;
```

## Параметры Doctrine миграции

- **Версия**: Будет определена Doctrine автоматически
- **Класс**: `Version<timestamp>`
- **Методы**: `up()` для создания, `down()` для удаления

## Пример миграции

```php
namespace DoctrineMigrations;

use Doctrine\DBAL\Schema\Schema;
use Doctrine\Migrations\AbstractMigration;

final class VersionAccountContext extends AbstractMigration
{
    public function getDescription(): string
    {
        return 'Create users table for Account Context';
    }

    public function up(Schema $schema): void
    {
        $this->addSql('CREATE TABLE users (...)');
    }

    public function down(Schema $schema): void
    {
        $this->addSql('DROP TABLE users');
    }
}
```

## Критерии готовности

- [x] Миграция создана и может быть выполнена
- [x] Структура таблиц соответствует UserEntity
- [x] Индексы добавлены для оптимизации
- [x] Миграция может быть откачена (down метод работает)
- [x] Документация актуальна

## Зависимости

- Doctrine Migrations
- Doctrine DBAL

## Примечания

- UUID в MySQL может быть CHAR(36) или BINARY(16) в зависимости от конфигурации Doctrine
- JSON тип поддерживается в MySQL 5.7+
- Foreign key ограничения помогут поддерживать целостность данных
- Индексы улучшат производительность поиска

## Статус

done
