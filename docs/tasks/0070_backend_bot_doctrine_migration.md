# Задача 0070: Bot Context Doctrine Migration

## Контекст

Создание миграции базы данных для таблиц Bot Context: notification_endpoints и conversation_states.

## Цель

Создать Doctrine миграцию для Bot Context.

## Файлы для создания

```
migrations/Version20240XXX_BotContext.php
```

## Требования

### Migration

```php
<?php

declare(strict_types=1);

namespace DoctrineMigrations;

use Doctrine\DBAL\Schema\Schema;
use Doctrine\Migrations\AbstractMigration;

final class Version20240XXX_BotContext extends AbstractMigration
{
    public function getDescription(): string
    {
        return 'Create Bot Context tables: notification_endpoints and conversation_states';
    }

    public function up(Schema $schema): void
    {
        // Таблица bot_notification_endpoints
        $this->addSql('
            CREATE TABLE bot_notification_endpoints (
                id VARCHAR(36) NOT NULL,
                user_id VARCHAR(36) NOT NULL,
                messenger VARCHAR(32) NOT NULL,
                external_target_id VARCHAR(255) NOT NULL,
                status VARCHAR(32) NOT NULL DEFAULT \'active\',
                created_at TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP,
                revoked_at TIMESTAMP NULL,
                PRIMARY KEY (id)
            )
        ');

        // Индексы для notification_endpoints
        $this->addSql('
            CREATE INDEX idx_endpoints_user_messenger_status
            ON bot_notification_endpoints (user_id, messenger, status)
        ');

        $this->addSql('
            CREATE INDEX idx_endpoints_messenger_external
            ON bot_notification_endpoints (messenger, external_target_id)
        ');

        // Уникальный constraint: один активный endpoint на user+messenger
        $this->addSql('
            CREATE UNIQUE INDEX uniq_endpoints_user_messenger
            ON bot_notification_endpoints (user_id, messenger)
            WHERE status = \'active\'
        ');

        // Таблица bot_conversation_states
        $this->addSql('
            CREATE TABLE bot_conversation_states (
                id VARCHAR(36) NOT NULL,
                user_id VARCHAR(36) NOT NULL,
                messenger VARCHAR(32) NOT NULL,
                conversation_code VARCHAR(64) NOT NULL,
                current_step VARCHAR(64) NOT NULL,
                data JSONB NOT NULL DEFAULT \'{}\',
                created_at TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP,
                updated_at TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP,
                PRIMARY KEY (id)
            )
        ');

        // Индексы для conversation_states
        $this->addSql('
            CREATE INDEX idx_states_user_id
            ON bot_conversation_states (user_id)
        ');

        // Уникальный constraint: один диалог на user+messenger
        $this->addSql('
            CREATE UNIQUE INDEX uniq_states_user_messenger
            ON bot_conversation_states (user_id, messenger)
        ');

        // Комментарии к таблицам
        $this->addSql("COMMENT ON TABLE bot_notification_endpoints IS 'Bot Context: точки доставки сообщений'");
        $this->addSql("COMMENT ON TABLE bot_conversation_states IS 'Bot Context: состояния активных диалогов'");

        // Комментарии к важным колонкам
        $this->addSql("COMMENT ON COLUMN bot_notification_endpoints.external_target_id IS 'Внешний ID чата (chat_id). НИКОГДА не передавать за пределы Bot Context!'");
    }

    public function down(Schema $schema): void
    {
        $this->addSql('DROP TABLE IF EXISTS bot_conversation_states');
        $this->addSql('DROP TABLE IF EXISTS bot_notification_endpoints');
    }
}
```

## Структура таблиц

### bot_notification_endpoints

| Колонка | Тип | Описание |
|---------|-----|----------|
| id | VARCHAR(36) | UUID идентификатор |
| user_id | VARCHAR(36) | FK на users (логическая связь) |
| messenger | VARCHAR(32) | telegram, whatsapp, discord |
| external_target_id | VARCHAR(255) | chat_id (секретные данные!) |
| status | VARCHAR(32) | active, revoked, blocked |
| created_at | TIMESTAMP | Дата создания |
| revoked_at | TIMESTAMP | Дата отзыва (nullable) |

### bot_conversation_states

| Колонка | Тип | Описание |
|---------|-----|----------|
| id | VARCHAR(36) | UUID идентификатор |
| user_id | VARCHAR(36) | FK на users (логическая связь) |
| messenger | VARCHAR(32) | telegram, whatsapp, discord |
| conversation_code | VARCHAR(64) | Код диалога (start, help, etc.) |
| current_step | VARCHAR(64) | Текущий шаг FSM |
| data | JSONB | Данные диалога |
| created_at | TIMESTAMP | Дата создания |
| updated_at | TIMESTAMP | Дата обновления |

## Индексы

- `idx_endpoints_user_messenger_status` — поиск активных endpoints пользователя
- `idx_endpoints_messenger_external` — поиск по chat_id (для webhook)
- `uniq_endpoints_user_messenger` — уникальность (partial index для active)
- `idx_states_user_id` — поиск диалогов пользователя
- `uniq_states_user_messenger` — один диалог на user+messenger

## Безопасность

**ВАЖНО:** Колонка `external_target_id` содержит sensitive данные (chat_id). Эти данные:
- Никогда не передаются за пределы Bot Context
- Не логируются
- Не включаются в события
- Доступны только внутри Bot Context

## Тесты

- [ ] Миграция успешно выполняется (up)
- [ ] Миграция успешно откатывается (down)
- [ ] Индексы созданы корректно
- [ ] Уникальные constraints работают
- [ ] JSONB колонка работает с PostgreSQL

## Зависимости

- PostgreSQL 16+
- Doctrine Migrations Bundle

## Definition of Done

- [x] Миграция создана
- [ ] Миграция протестирована на чистой базе
- [x] Rollback работает
- [x] Комментарии к таблицам добавлены
- [x] Код соответствует стандартам проекта
