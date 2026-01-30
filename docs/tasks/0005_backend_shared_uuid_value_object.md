# Задача 05: Реализация UUID Value Object

## Описание

Реализовать базовый Value Object `UUID` для типобезопасного представления уникальных идентификаторов с использованием библиотеки `ramsey/uuid`.

## Требования

- [ ] Класс должен быть расположен в `src/Context/Shared/Domain/Model/UUID.php`
- [ ] Инкапсулировать `Ramsey\Uuid\UuidInterface`
- [ ] Быть immutable (неизменяемым)
- [ ] Поддерживать сравнение по значению
- [ ] Не содержать бизнес-логики
- [ ] Не зависеть от инфраструктуры

## Интерфейс

```php
class UUID
{
    public function __construct(UuidInterface $uuid);
    
    public static function fromString(string $value): self
    
    public static function generate(): self
    
    public function toString(): string
    
    public function equals(UUID $other): bool
}
```

## Требования

- [ ] `__construct()` - приватный/защищённый конструктор
- [ ] `fromString()` - создание UUID из строки с валидацией (выбрасывает `InvalidUuidException`)
- [ ] `generate()` - генерация нового UUID v4
- [ ] `toString()` - возврат строкового представления
- [ ] `equals()` - сравнение двух UUID
- [ ] `__serialize()` и `__unserialize()` - сериализация/десериализация значения

## Обработка ошибок

- [ ] При невалидном UUID выбрасывается `InvalidUuidException`
- [ ] При пустой строке выбрасывается `InvalidUuidException`

## Инварианты

- [ ] UUID всегда валиден после создания
- [ ] UUID не может быть null или пустым
- [ ] UUID неизменяем после создания

## Критерии готовности

- [x] Класс реализован с требуемыми методами
- [ ] Покрыт unit-тестами (минимум 5 тестов)
- [ ] Документация актуальна

## Зависимости

- [Задача 03: InvalidUuidException](./backend_shared_03_invalid_uuid_exception.md)

## Статус

not-started
