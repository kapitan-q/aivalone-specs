# Задача 06: Реализация Messenger Enum

## Описание

Реализовать доменный enum `Messenger` для унификации идентификаторов поддерживаемых мессенджеров и предотвращения использования "магических строк".

## Требования

- [ ] Класс должен быть расположен в `src/Context/Shared/Domain/Model/Messenger.php`
- [ ] Должен быть PHP 8.1+ enum
- [ ] Поддерживать значения: TELEGRAM, WHATSAPP, DISCORD, VIBER
- [ ] Методы для работы с enum значениями

## Поддерживаемые мессенджеры

| Значение | Код        | Статус    |
|----------|------------|-----------|
| TELEGRAM | `TELEGRAM` | Активен   |
| WHATSAPP | `WHATSAPP` | План      |
| DISCORD  | `DISCORD`  | План      |
| VIBER    | `VIBER`    | План      |

## Интерфейс

```php
class Messenger
{
    public static function fromCode(string $code): self
    
    public function code(): string
}
```

## Требования

- [ ] `fromCode()` - создание из строкового кода (выбрасывает `InvalidMessengerException`)
- [ ] `code()` - возврат строкового кода
- [ ] Все значения должны быть доступны через константы или свойства

## Обработка ошибок

- [ ] При неподдерживаемом коде выбрасывается `InvalidMessengerException`
- [ ] При пустой или некорректной строке выбрасывается `InvalidMessengerException`

## Инварианты

- [ ] Enum может содержать только валидные мессенджеры
- [ ] Один Messenger однозначно определяет тип мессенджера
- [ ] Расширение списка происходит только через добавление нового значения
- [ ] Удаление значения допускается только через deprecated-период

## Использование

```php
$telegram = Messenger::fromCode('TELEGRAM');
echo $telegram->code(); // 'TELEGRAM'

$telegram2 = Messenger::fromCode('TELEGRAM');
assert($telegram->equals($telegram2)); // true
```

## Критерии готовности

- [x] Класс/enum реализован с требуемыми методами
- [ ] Покрыт unit-тестами (минимум 4 теста)
- [ ] Документация актуальна

## Зависимости

- [Задача 04: InvalidMessengerException](./backend_shared_04_invalid_messenger_exception.md)

## Статус

not-started
