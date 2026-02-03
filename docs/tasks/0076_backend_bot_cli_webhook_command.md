# Задача 0076: CLI Command для настройки Webhook

## Контекст

CLI команда для настройки и управления webhook. Используется для первоначальной настройки бота и диагностики.

## Цель

Создать Symfony Console Command для управления webhook.

## Файлы для создания

```
src/Context/Bot/Infrastructure/Cli/SetupWebhookCommand.php
src/Context/Bot/Infrastructure/Cli/WebhookInfoCommand.php
tests/Unit/Context/Bot/Infrastructure/Cli/WebhookCommandsTest.php
```

## Важно

**URL webhook формируется автоматически:**
- BASE_URL из ENV + `/bot/webhook/{messenger}`
- Пользователю не нужно вводить полный URL

## Требования

### SetupWebhookCommand

```php
<?php

declare(strict_types=1);

namespace App\Context\Bot\Infrastructure\Cli;

use App\Context\Bot\Application\Adapter\MessengerAdapterInterface;
use App\Context\Shared\Domain\Enum\Messenger;
use Symfony\Component\Console\Attribute\AsCommand;
use Symfony\Component\Console\Command\Command;
use Symfony\Component\Console\Input\InputArgument;
use Symfony\Component\Console\Input\InputInterface;
use Symfony\Component\Console\Input\InputOption;
use Symfony\Component\Console\Output\OutputInterface;
use Symfony\Component\Console\Style\SymfonyStyle;

#[AsCommand(
    name: 'bot:webhook:setup',
    description: 'Setup webhook for messenger bot',
)]
final class SetupWebhookCommand extends Command
{
    /**
     * @var array<string, MessengerAdapterInterface>
     */
    private array $adapters = [];

    /**
     * @param iterable<MessengerAdapterInterface> $adapters
     */
    public function __construct(
        iterable $adapters,
        private readonly string $baseUrl,
    ) {
        parent::__construct();

        foreach ($adapters as $adapter) {
            $this->adapters[$adapter->getMessenger()->value] = $adapter;
        }
    }

    protected function configure(): void
    {
        $this
            ->addArgument(
                'messenger',
                InputArgument::REQUIRED,
                'Messenger type (telegram, whatsapp, discord)',
            )
            ->addOption(
                'remove',
                'r',
                InputOption::VALUE_NONE,
                'Remove webhook instead of setting up',
            )
            ->addOption(
                'custom-url',
                null,
                InputOption::VALUE_REQUIRED,
                'Override auto-generated webhook URL',
            );
    }

    protected function execute(InputInterface $input, OutputInterface $output): int
    {
        $io = new SymfonyStyle($input, $output);

        $messengerValue = $input->getArgument('messenger');
        $remove = $input->getOption('remove');
        $customUrl = $input->getOption('custom-url');

        // Валидация messenger
        try {
            $messenger = Messenger::from($messengerValue);
        } catch (\ValueError) {
            $io->error(sprintf('Unknown messenger: %s', $messengerValue));
            $io->note('Available messengers: ' . implode(', ', array_column(Messenger::cases(), 'value')));
            return Command::FAILURE;
        }

        // Проверяем наличие адаптера
        if (!isset($this->adapters[$messenger->value])) {
            $io->error(sprintf('No adapter configured for %s', $messenger->value));
            return Command::FAILURE;
        }

        $adapter = $this->adapters[$messenger->value];

        if ($remove) {
            return $this->removeWebhook($io, $adapter, $messenger);
        }

        // Формируем URL автоматически или используем custom
        $webhookUrl = $customUrl ?? $this->buildWebhookUrl($messenger);

        return $this->setupWebhook($io, $adapter, $messenger, $webhookUrl);
    }

    private function buildWebhookUrl(Messenger $messenger): string
    {
        $baseUrl = rtrim($this->baseUrl, '/');

        return sprintf('%s/bot/webhook/%s', $baseUrl, $messenger->value);
    }

    private function setupWebhook(
        SymfonyStyle $io,
        MessengerAdapterInterface $adapter,
        Messenger $messenger,
        string $url,
    ): int {
        // Валидация URL
        if (!str_starts_with($url, 'https://')) {
            $io->error('Webhook URL must use HTTPS');
            return Command::FAILURE;
        }

        $io->title(sprintf('Setting up webhook for %s', $messenger->value));
        $io->text(sprintf('URL: %s', $url));

        try {
            $adapter->setWebhook($url);

            $io->success('Webhook configured successfully!');

            // Показываем текущую информацию
            $info = $adapter->getBotInfo();
            $io->table(
                ['Property', 'Value'],
                [
                    ['Bot Username', '@' . $info['username']],
                    ['Webhook URL', $url],
                ],
            );

            return Command::SUCCESS;

        } catch (\Throwable $e) {
            $io->error(sprintf('Failed to setup webhook: %s', $e->getMessage()));
            return Command::FAILURE;
        }
    }

    private function removeWebhook(
        SymfonyStyle $io,
        MessengerAdapterInterface $adapter,
        Messenger $messenger,
    ): int {
        $io->title(sprintf('Removing webhook for %s', $messenger->value));

        try {
            $adapter->deleteWebhook();

            $io->success('Webhook removed successfully!');
            return Command::SUCCESS;

        } catch (\Throwable $e) {
            $io->error(sprintf('Failed to remove webhook: %s', $e->getMessage()));
            return Command::FAILURE;
        }
    }
}
```

### WebhookInfoCommand

```php
<?php

declare(strict_types=1);

namespace App\Context\Bot\Infrastructure\Cli;

use App\Context\Bot\Application\Adapter\MessengerAdapterInterface;
use App\Context\Shared\Domain\Enum\Messenger;
use Symfony\Component\Console\Attribute\AsCommand;
use Symfony\Component\Console\Command\Command;
use Symfony\Component\Console\Input\InputArgument;
use Symfony\Component\Console\Input\InputInterface;
use Symfony\Component\Console\Output\OutputInterface;
use Symfony\Component\Console\Style\SymfonyStyle;

#[AsCommand(
    name: 'bot:webhook:info',
    description: 'Show webhook information for messenger bot',
)]
final class WebhookInfoCommand extends Command
{
    /**
     * @var array<string, MessengerAdapterInterface>
     */
    private array $adapters = [];

    /**
     * @param iterable<MessengerAdapterInterface> $adapters
     */
    public function __construct(
        iterable $adapters,
        private readonly string $baseUrl,
    ) {
        parent::__construct();

        foreach ($adapters as $adapter) {
            $this->adapters[$adapter->getMessenger()->value] = $adapter;
        }
    }

    protected function configure(): void
    {
        $this->addArgument(
            'messenger',
            InputArgument::OPTIONAL,
            'Messenger type (telegram, whatsapp, discord). If not specified, shows all.',
        );
    }

    protected function execute(InputInterface $input, OutputInterface $output): int
    {
        $io = new SymfonyStyle($input, $output);

        $messengerValue = $input->getArgument('messenger');

        if ($messengerValue !== null) {
            try {
                $messenger = Messenger::from($messengerValue);
                return $this->showInfo($io, $messenger);
            } catch (\ValueError) {
                $io->error(sprintf('Unknown messenger: %s', $messengerValue));
                return Command::FAILURE;
            }
        }

        // Показываем информацию по всем настроенным адаптерам
        foreach ($this->adapters as $messengerCode => $adapter) {
            $this->showInfo($io, Messenger::from($messengerCode));
        }

        return Command::SUCCESS;
    }

    private function showInfo(SymfonyStyle $io, Messenger $messenger): int
    {
        if (!isset($this->adapters[$messenger->value])) {
            $io->warning(sprintf('No adapter configured for %s', $messenger->value));
            return Command::SUCCESS;
        }

        $adapter = $this->adapters[$messenger->value];

        $io->section(sprintf('Webhook info for %s', $messenger->value));

        // Информация о боте
        try {
            $botInfo = $adapter->getBotInfo();
            $io->text(sprintf('Bot: @%s (%s)', $botInfo['username'], $botInfo['name']));
        } catch (\Throwable $e) {
            $io->warning('Failed to get bot info: ' . $e->getMessage());
            return Command::FAILURE;
        }

        // Ожидаемый URL
        $expectedUrl = sprintf('%s/bot/webhook/%s', rtrim($this->baseUrl, '/'), $messenger->value);
        $io->text(sprintf('Expected webhook URL: %s', $expectedUrl));

        // Health check
        $health = $adapter->checkHealth();
        $io->text(sprintf('Health: %s', $health ? '✅ OK' : '❌ Failed'));

        $io->newLine();

        return Command::SUCCESS;
    }
}
```

## Configuration

```yaml
# config/services.yaml
parameters:
    app.base_url: '%env(APP_BASE_URL)%'

services:
    App\Context\Bot\Infrastructure\Cli\SetupWebhookCommand:
        arguments:
            $adapters: !tagged_iterator bot.messenger_adapter
            $baseUrl: '%app.base_url%'

    App\Context\Bot\Infrastructure\Cli\WebhookInfoCommand:
        arguments:
            $adapters: !tagged_iterator bot.messenger_adapter
            $baseUrl: '%app.base_url%'
```

## Environment Variables

```env
# Base URL для автогенерации webhook URL
APP_BASE_URL=https://api.aivalone.com
```

## Использование

```bash
# Настройка webhook (URL формируется автоматически)
php bin/console bot:webhook:setup telegram
# -> URL: https://api.aivalone.com/bot/webhook/telegram

# Настройка с кастомным URL
php bin/console bot:webhook:setup telegram --custom-url=https://custom.domain.com/webhook

# Удаление webhook
php bin/console bot:webhook:setup telegram --remove

# Просмотр информации о конкретном мессенджере
php bin/console bot:webhook:info telegram

# Просмотр всех webhook
php bin/console bot:webhook:info
```

## Примеры вывода

```
Setting up webhook for telegram
===============================

URL: https://api.aivalone.com/bot/webhook/telegram

 [OK] Webhook configured successfully!

 ------------- --------------------------------------------
  Property      Value
 ------------- --------------------------------------------
  Bot Username  @aivalone_bot
  Webhook URL   https://api.aivalone.com/bot/webhook/telegram
 ------------- --------------------------------------------
```

## Тесты

### SetupWebhookCommand
- [ ] Команда формирует URL автоматически из baseUrl
- [ ] Команда настраивает webhook для telegram
- [ ] Команда удаляет webhook с флагом --remove
- [ ] Команда использует custom-url если указан
- [ ] Команда выводит ошибку для неизвестного messenger
- [ ] Команда выводит ошибку для HTTP URL
- [ ] Команда показывает информацию после настройки

### WebhookInfoCommand
- [ ] Команда показывает информацию для указанного messenger
- [ ] Команда показывает информацию для всех messenger без аргумента
- [ ] Команда показывает ожидаемый webhook URL
- [ ] Команда показывает health status

## Зависимости

- `MessengerAdapterInterface` (задача 0063)
- `App\Context\Shared\Domain\Enum\Messenger` (задача 0006)
- Symfony Console

## Definition of Done

- [ ] SetupWebhookCommand создан
- [ ] WebhookInfoCommand создан
- [ ] URL формируется автоматически из ENV
- [ ] Unit-тесты написаны и проходят
- [ ] Код соответствует PSR-12
- [ ] PHPStan level 9 без ошибок
