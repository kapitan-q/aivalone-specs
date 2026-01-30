# Диаграмма взаимодействия процессов и событий Account Context

## Диаграмма (conceptual)

```mermaid
graph TD
    subgraph Процессы
        CU[CreateUser]
        AUM[AddUserMessenger]
        RUM[RemoveUserMessenger]
        UUT[UpdateUserTariffs]
    end
    subgraph События
        UR[UserRegistered]
        UMU[UserMessengersUpdated]
        UTU[UserTariffsUpdated]
    end
    CU --создает--> UR
    CU --инициализация--> UMU
    AUM --добавляет--> UMU
    RUM --удаляет--> UMU
    UUT --обновляет--> UTU
    CU --ошибка--> DuplicateUserException
    CU --ошибка--> ValidationException
    AUM --ошибка--> DuplicateMessengerException
    AUM --ошибка--> UserNotFoundException
    RUM --ошибка--> MessengerNotFoundException
    RUM --ошибка--> UserNotFoundException
    UUT --ошибка--> TariffValidationException
    UUT --ошибка--> UserNotFoundException
```

## Описание

- Процессы инициируют события, отражающие изменения в домене.
- Исключения выбрасываются при нарушении инвариантов или бизнес-правил.
- Все события и ошибки связаны с конкретными процессами.

## Связанные документы

* [Account Context Overview](../overview.md)
* [Процессы](../processes/overview.md)
* [События](../events/overview.md)
* [Исключения](../exceptions/overview.md)
