# Задача 0057: Bot Context Exceptions

## Контекст

Bot Context требует набор исключений для обработки ошибок в домене: недействительные команды, отсутствующие endpoint, конфликты диалогов и т.д.

## Цель

Создать все исключения контекста Bot.

## Спецификации

- [Exceptions Overview](../backend/bot/exceptions/overview.md)
- [ConversationNotFoundException](../backend/bot/exceptions/conversation-not-found.md)
- [InvalidCommandException](../backend/bot/exceptions/invalid-command.md)
- [EndpointNotFoundException](../backend/bot/exceptions/endpoint-not-found.md)
- [EndpointAlreadyRevokedException](../backend/bot/exceptions/endpoint-already-revoked.md)
- [MessageDeliveryException](../backend/bot/exceptions/message-delivery.md)
- [ConversationConflictException](../backend/bot/exceptions/conversation-conflict.md)

## Файлы для создания

```
src/Context/Bot/Domain/Exception/ConversationNotFoundException.php
src/Context/Bot/Domain/Exception/InvalidCommandException.php
src/Context/Bot/Domain/Exception/EndpointNotFoundException.php
src/Context/Bot/Domain/Exception/EndpointAlreadyRevokedException.php
src/Context/Bot/Domain/Exception/MessageDeliveryException.php
src/Context/Bot/Domain/Exception/ConversationConflictException.php
tests/Unit/Context/Bot/Domain/Exception/BotExceptionsTest.php
```

## Требования

### ConversationNotFoundException

```php
<?php

declare(strict_types=1);

namespace App\Context\Bot\Domain\Exception;

use App\Context\Shared\Domain\Exception\DomainException;

final class ConversationNotFoundException extends DomainException
{
    public function __construct(string $conversationCode)
    {
        parent::__construct(
            sprintf('Conversation with code "%s" not found', $conversationCode),
            'CONVERSATION_NOT_FOUND',
        );
    }
}
```

### InvalidCommandException

```php
<?php

declare(strict_types=1);

namespace App\Context\Bot\Domain\Exception;

use App\Context\Shared\Domain\Exception\DomainException;

final class InvalidCommandException extends DomainException
{
    public function __construct(string $command, string $reason = '')
    {
        $message = sprintf('Invalid command: "%s"', $command);
        if ($reason !== '') {
            $message .= sprintf('. Reason: %s', $reason);
        }

        parent::__construct($message, 'INVALID_COMMAND');
    }
}
```

### EndpointNotFoundException

```php
<?php

declare(strict_types=1);

namespace App\Context\Bot\Domain\Exception;

use App\Context\Bot\Domain\Model\EndpointId;
use App\Context\Shared\Domain\Exception\DomainException;
use App\Context\Shared\Domain\ValueObject\Uuid;

final class EndpointNotFoundException extends DomainException
{
    public static function byId(EndpointId $id): self
    {
        return new self(
            sprintf('NotificationEndpoint with ID "%s" not found', $id->toString()),
            'ENDPOINT_NOT_FOUND',
        );
    }

    public static function byUserId(Uuid $userId): self
    {
        return new self(
            sprintf('NotificationEndpoint for user "%s" not found', $userId->toString()),
            'ENDPOINT_NOT_FOUND',
        );
    }
}
```

### EndpointAlreadyRevokedException

```php
<?php

declare(strict_types=1);

namespace App\Context\Bot\Domain\Exception;

use App\Context\Bot\Domain\Model\EndpointId;
use App\Context\Shared\Domain\Exception\DomainException;

final class EndpointAlreadyRevokedException extends DomainException
{
    public function __construct(EndpointId $id)
    {
        parent::__construct(
            sprintf('NotificationEndpoint "%s" is already revoked', $id->toString()),
            'ENDPOINT_ALREADY_REVOKED',
        );
    }
}
```

### MessageDeliveryException

```php
<?php

declare(strict_types=1);

namespace App\Context\Bot\Domain\Exception;

use App\Context\Bot\Domain\Model\EndpointId;
use App\Context\Shared\Domain\Exception\DomainException;

final class MessageDeliveryException extends DomainException
{
    public function __construct(
        EndpointId $endpointId,
        string $reason,
        ?\Throwable $previous = null,
    ) {
        parent::__construct(
            sprintf(
                'Failed to deliver message to endpoint "%s": %s',
                $endpointId->toString(),
                $reason,
            ),
            'MESSAGE_DELIVERY_FAILED',
            $previous,
        );
    }
}
```

### ConversationConflictException

```php
<?php

declare(strict_types=1);

namespace App\Context\Bot\Domain\Exception;

use App\Context\Shared\Domain\Exception\DomainException;

final class ConversationConflictException extends DomainException
{
    public function __construct(
        string $currentConversation,
        string $requestedConversation,
    ) {
        parent::__construct(
            sprintf(
                'Cannot start conversation "%s" while "%s" is active',
                $requestedConversation,
                $currentConversation,
            ),
            'CONVERSATION_CONFLICT',
        );
    }
}
```

## Тесты

- [ ] ConversationNotFoundException создаётся с правильным сообщением
- [ ] InvalidCommandException создаётся с command и опциональным reason
- [ ] EndpointNotFoundException::byId() формирует корректное сообщение
- [ ] EndpointNotFoundException::byUserId() формирует корректное сообщение
- [ ] EndpointAlreadyRevokedException содержит ID endpoint
- [ ] MessageDeliveryException содержит причину и поддерживает previous exception
- [ ] ConversationConflictException содержит оба кода диалогов
- [ ] Все исключения имеют корректные error codes

## Зависимости

- `App\Context\Shared\Domain\Exception\DomainException` (задача 0001)
- `EndpointId` (задача 0051)

## Definition of Done

- [ ] Все 6 классов исключений созданы
- [ ] Каждое исключение имеет уникальный error code
- [ ] Unit-тесты написаны и проходят
- [ ] Код соответствует PSR-12
- [ ] PHPStan level 9 без ошибок
