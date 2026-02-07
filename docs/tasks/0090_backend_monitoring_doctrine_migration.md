# Задача 0090: Doctrine Migration для Monitoring Context

## Контекст

Monitoring Context требует создание таблиц в базе данных для хранения профилей, фильтров, сессий, групп и связей.

## Цель

Создать Doctrine Migration для всех таблиц Monitoring Context.

## Спецификация

- [Infrastructure Overview](../backend/monitoring/infrastructure/overview.md)

## Файлы для создания

```
migrations/VersionXXXX_CreateMonitoringTables.php
```

## Требования

### Таблицы

#### monitoring_profiles

```sql
CREATE TABLE monitoring_profiles (
    id VARCHAR(36) NOT NULL,
    user_id VARCHAR(36) NOT NULL,
    is_active BOOLEAN NOT NULL DEFAULT TRUE,
    created_at TIMESTAMP NOT NULL,
    updated_at TIMESTAMP NOT NULL,
    PRIMARY KEY (id),
    UNIQUE INDEX idx_profiles_user_id (user_id)
);
```

#### monitoring_filters

```sql
CREATE TABLE monitoring_filters (
    id VARCHAR(36) NOT NULL,
    profile_id VARCHAR(36) NOT NULL,
    value VARCHAR(500) NOT NULL,
    filter_type VARCHAR(20) NOT NULL,
    name VARCHAR(255) DEFAULT NULL,
    is_active BOOLEAN NOT NULL DEFAULT TRUE,
    created_at TIMESTAMP NOT NULL,
    updated_at TIMESTAMP NOT NULL,
    PRIMARY KEY (id),
    INDEX idx_filters_profile_id (profile_id),
    UNIQUE INDEX idx_filters_unique (profile_id, value, filter_type),
    CONSTRAINT fk_filters_profile FOREIGN KEY (profile_id) REFERENCES monitoring_profiles (id) ON DELETE CASCADE
);
```

#### monitoring_sessions

```sql
CREATE TABLE monitoring_sessions (
    id VARCHAR(36) NOT NULL,
    profile_id VARCHAR(36) NOT NULL,
    messenger_type VARCHAR(20) NOT NULL,
    external_session_id VARCHAR(255) DEFAULT NULL,
    status VARCHAR(20) NOT NULL,
    is_active BOOLEAN NOT NULL DEFAULT TRUE,
    display_name VARCHAR(255) DEFAULT NULL,
    last_activity_at TIMESTAMP DEFAULT NULL,
    created_at TIMESTAMP NOT NULL,
    updated_at TIMESTAMP NOT NULL,
    PRIMARY KEY (id),
    INDEX idx_sessions_profile_id (profile_id),
    UNIQUE INDEX idx_sessions_unique (profile_id, messenger_type),
    CONSTRAINT fk_sessions_profile FOREIGN KEY (profile_id) REFERENCES monitoring_profiles (id) ON DELETE CASCADE
);
```

#### monitoring_groups

```sql
CREATE TABLE monitoring_groups (
    id VARCHAR(36) NOT NULL,
    profile_id VARCHAR(36) NOT NULL,
    external_group_id VARCHAR(255) NOT NULL,
    group_title VARCHAR(255) NOT NULL,
    messenger_type VARCHAR(20) NOT NULL,
    session_id VARCHAR(36) DEFAULT NULL,
    is_private BOOLEAN NOT NULL DEFAULT FALSE,
    status VARCHAR(20) NOT NULL,
    last_message_at TIMESTAMP DEFAULT NULL,
    created_at TIMESTAMP NOT NULL,
    updated_at TIMESTAMP NOT NULL,
    PRIMARY KEY (id),
    INDEX idx_groups_profile_id (profile_id),
    UNIQUE INDEX idx_groups_unique (profile_id, external_group_id, messenger_type),
    INDEX idx_groups_external (external_group_id, messenger_type),
    CONSTRAINT fk_groups_profile FOREIGN KEY (profile_id) REFERENCES monitoring_profiles (id) ON DELETE CASCADE,
    CONSTRAINT fk_groups_session FOREIGN KEY (session_id) REFERENCES monitoring_sessions (id) ON DELETE SET NULL
);
```

#### monitoring_filter_group_bindings

```sql
CREATE TABLE monitoring_filter_group_bindings (
    filter_id VARCHAR(36) NOT NULL,
    group_id VARCHAR(36) NOT NULL,
    created_at TIMESTAMP NOT NULL,
    PRIMARY KEY (filter_id, group_id),
    CONSTRAINT fk_bindings_filter FOREIGN KEY (filter_id) REFERENCES monitoring_filters (id) ON DELETE CASCADE,
    CONSTRAINT fk_bindings_group FOREIGN KEY (group_id) REFERENCES monitoring_groups (id) ON DELETE CASCADE
);
```

## Тесты

- [ ] Миграция применяется без ошибок (up) — требует DB
- [ ] Миграция откатывается без ошибок (down) — требует DB
- [x] Все индексы создаются корректно (в SQL)
- [x] Foreign keys работают (в SQL)

## Зависимости

- Doctrine Entities (задача 0089)

## Definition of Done

- [x] Миграция создана
- [x] Все таблицы, индексы и FK корректны
- [ ] Миграция применяется и откатывается — требует DB
