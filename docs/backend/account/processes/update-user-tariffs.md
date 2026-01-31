# UpdateUserTariffs Process Specification

## Назначение

Процесс **обновления тарифов пользователя** в системе Aivalone.  
Отвечает за корректное изменение активных тарифных планов пользователя, проверку бизнес-инвариантов и публикацию события об изменении.

## Предусловия

* Пользователь должен существовать в системе (идентифицируется по `UserId`).  
* Новый список тарифов должен быть валидным (например, содержать только доступные коды тарифов).  
* Все изменения тарифов должны фиксировать дату последнего изменения (`updatedAt`).

## Поток выполнения

1. **Валидация входных данных** (Command Handler уровень)
   * Проверка наличия `userId`.
   * Проверка наличия списка `tariffCodes` (строки).
   * **Конвертация строковых кодов в объекты Tariff** — все коды должны соответствовать валидным значениям из [Tariff Enum](../../shared/models/tariff.md).
   * Выброс `TariffValidationException` если есть некорректные коды.
   * Делегация в `UserService::updateTariffs()`

2. **Получение пользователя** (UserService — Application Layer)
   * Получение пользователя из репозитория через `UserRepository::findById(UserId)`.
   * Выброс `UserNotFoundException` если пользователь не найден.

3. **Обновление тарифов пользователя** (Domain Layer)
   * Вызов `User::replaceTariffs(array<Tariff> $tariffs)` для замены текущих тарифов пользователя на новые.
   * Метод ожидает уже сконвертированные объекты Tariff (не строки).
   * Метод обновляет поле `updatedAt` текущим временем.
   * Автоматически записывается событие `UserTariffsUpdated` в агрегат с объектами Tariff.

4. **Сохранение пользователя** (UserService → Infrastructure Layer)
   * Сохранение агрегата `User` через `UserRepository::save(User)`.

5. **Публикация событий** (UserService)
   * Вызов `user->pullEvents()` для получения накопленных событий.
   * Публикация событий через `EventBus::publish()` (события содержат Tariff объекты, но getPayload() возвращает строки).

## Исключения / Ошибки

* Некорректный формат UserId — выбрасывается `InvalidUuidException`.
* Некорректные тарифные коды — выбрасывается [TariffValidationException](../exceptions/tariff-validation-exception.md) на уровне Command Handler.
* Пользователь не найден — выбрасывается [UserNotFoundException](../exceptions/user-not-found-exception.md).

## Примечания

* Процесс может быть вызван:
  - Command Handler (`UpdateUserTariffsCommandHandler`) при получении команды с UI, API или других контекстов
  - Event Handler при получении события `UserTariffsUpdatedEvent` из Billing Context
* **Важно**: Конвертация строк в Tariff объекты происходит на уровне **Command Handler**, перед вызовом UserService
* Внутри контекста (User Model, Events) используются объекты Tariff для типизации и валидации
* При передаче данных через границы контекста (getPayload() в событиях, DTO в ответах) используются строковые коды тарифов
* Валидные тарифные коды определены в [Tariff Enum](../../shared/models/tariff.md) в Shared Context