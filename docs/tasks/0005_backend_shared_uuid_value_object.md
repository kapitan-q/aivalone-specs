# Задача 05: Реализация UUID Value Object

## Описание

Реализовать базовый Value Object `UUID` для типобезопасного представления уникальных идентификаторов с использованием библиотеки `ramsey/uuid`.

## Требования

- [x] Класс должен быть расположен в `src/Context/Shared/Domain/Model/UUID.php`
- [x] Инкапсулировать `Ramsey\Uuid\UuidInterface`
- [x] Быть immutable (неизменяемым)
- [x] Поддерживать сравнение по значению
- [x] Не содержать бизнес-логики
- [x] Не зависеть от инфраструктуры

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

- [x] `__construct()` - приватный/защищённый конструктор
- [x] `fromString()` - создание UUID из строки с валидацией (выбрасывает `InvalidUuidException`)
- [x] `generate()` - генерация нового UUID v4
- [x] `toString()` - возврат строкового представления
- [x] `equals()` - сравнение двух UUID
- [x] `__serialize()` и `__unserialize()` - сериализация/десериализация значения

## Обработка ошибок

- [x] При невалидном UUID выбрасывается `InvalidUuidException`
- [x] При пустой строке выбрасывается `InvalidUuidException`

## Инварианты

- [x] UUID всегда валиден после создания
- [x] UUID не может быть null или пустым
- [x] UUID неизменяем после создания

## Критерии готовности

- [x] Класс реализован с требуемыми методами
- [ ] Покрыт unit-тестами (минимум 5 тестов)
- [ ] Документация актуальна

## Зависимости

- [Задача 03: InvalidUuidException](./backend_shared_03_invalid_uuid_exception.md)

## Статус

done
