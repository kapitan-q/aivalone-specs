# Пропускаем!!! Задача 0031: Реализация Event Handler: UserTariffsUpdatedEventHandler

## Описание

Реализовать Event Handler для обработки события `UserTariffsUpdatedEvent` из Billing Context.
Синхронизирует обновленные тарифы пользователя в Account Context через Command Handler.

## Требования

- [ ] Класс должен быть расположен в `src/Context/Account/Application/EventHandler/UserTariffsUpdatedEventHandler.php`
- [ ] Быть обработчиком события (Event Subscriber / Event Handler)
- [ ] Слушать событие из Billing Context
- [ ] Делегировать обновление через UpdateUserTariffsCommandHandler
- [ ] Обрабатывать ошибки корректно

## Входящее событие (из Billing Context)

```php
class UserTariffsUpdatedEvent implements DomainEventInterface
{
    // из Billing Context
    // содержит userId и новые tariffs (строковые коды ['FREE', 'BASE'])
}
```

## Процесс обработки

```
UserTariffsUpdatedEvent (из Billing Context)
    ↓
1. Получение данных из события
    - userId
    - tariffs (новые тарифы как строковые коды ['FREE', 'BASE'])
    ↓
2. Создание команды
    - new UpdateUserTariffsCommand($userId, $tariffs)
    ↓
3. Отправка команды в Command Bus
    - commandBus->dispatch($command)
    ↓
4. Command Handler обрабатывает команду
    - Конвертирует строки в Tariff объекты
    - Вызывает UserService с Tariff объектами
    - (см. Задача 29)
    ↓
Результат: Тарифы синхронизированы в Account Context
```

## Сигнатура

```php
class UserTariffsUpdatedEventHandler
{
    public function __construct(
        private CommandBusInterface $commandBus
    ) {}
    
    public function handle(UserTariffsUpdatedEvent $event): void
    {
        // Обработка события
    }
}
```

## Регистрация в Symfony DI

```yaml
# config/services.yaml
App\Context\Account\Application\EventHandler\UserTariffsUpdatedEventHandler:
    tags:
        - { name: 'account.event_handler', event: 'user.tariffs.updated' }
```

## Обработка ошибок

- Если команда выбросит исключение, Event Handler может:
  - Залогировать ошибку и продолжить
  - Перебросить исключение (для обработки на уровне инфраструктуры)
  - Создать compensating action (возврат к старым тарифам)

## Критерии готовности

- [x] Класс реализован
- [ ] Покрыт unit-тестами (минимум 2 теста: успех, ошибка команды)
- [ ] Зарегистрирован в Symfony DI
- [ ] Документация актуальна

## Зависимости

- Account Context: UpdateUserTariffsCommandHandler
- Account Context: UpdateUserTariffsCommand
- Shared Context: EventBusInterface (для слушания)
- Symfony Messenger (для Command Bus)

## Примечания

- Event Handler должен быть асинхронным (использовать Symfony Messenger)
- Обработка ошибок может быть на уровне инфраструктуры (retry, deadletter queue)
- Integration с Billing Context будет требовать определения формата события

## Статус

not-started
