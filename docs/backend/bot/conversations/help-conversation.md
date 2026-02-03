# HelpConversation Specification

## Назначение

`HelpConversation` — диалог помощи и FAQ. Вызывается командой `/help`. Также используется как fallback при ошибках.

## Реализация

```php
final class HelpConversation extends AbstractConversation
{
    public function __construct(
        private ConversationRegistryInterface $registry,
    ) {}

    public function getCode(): string
    {
        return 'help';
    }

    public function getDescription(): string
    {
        return 'Помощь и поддержка';
    }

    protected function getInitialStep(): string
    {
        return 'menu';
    }

    public function getMenuInfo(): ?array
    {
        return [
            'icon' => '❓',
            'name' => 'Помощь',
            'link' => $this->getCommand(),
        ];
    }

    protected function stepMenu(?string $message, array $params): BotResponse
    {
        $text = <<<TEXT
❓ Помощь

Выберите интересующий раздел:
TEXT;

        return BotResponse::messageWithKeyboard($text, [
            [['text' => '📖 Как начать', 'action' => $this->getLink('getting_started')]],
            [['text' => '💳 Тарифы', 'action' => $this->getLink('pricing')]],
            [['text' => '❓ FAQ', 'action' => $this->getLink('faq')]],
            [['text' => '📞 Поддержка', 'action' => $this->getLink('support')]],
            [['text' => '◀️ Назад', 'action' => '/start']],
        ]);
    }

    protected function stepGettingStarted(?string $message, array $params): BotResponse
    {
        $text = <<<TEXT
📖 Как начать работу

1️⃣ Откройте приложение через кнопку "Открыть приложение"

2️⃣ Подключите Telegram-аккаунт для мониторинга

3️⃣ Добавьте каналы и группы для отслеживания

4️⃣ Настройте фильтры и ключевые слова

5️⃣ Получайте уведомления о важных событиях!
TEXT;

        return BotResponse::messageWithKeyboard($text, [
            [['text' => '🚀 Начать', 'action' => '/start']],
            [['text' => '◀️ Назад', 'action' => $this->getLink('menu')]],
        ]);
    }

    protected function stepPricing(?string $message, array $params): BotResponse
    {
        $text = <<<TEXT
💳 Тарифы Aivalone

🆓 FREE
• До 5 групп для мониторинга
• Базовые фильтры
• Email уведомления

💼 BASE — $9.99/мес
• До 50 групп
• Расширенные фильтры
• Push уведомления

⭐ PRO — $29.99/мес
• Неограниченно групп
• AI анализ контента
• Приоритетная поддержка
TEXT;

        return BotResponse::messageWithKeyboard($text, [
            [['text' => '📦 Изменить тариф', 'action' => '/billing']],
            [['text' => '◀️ Назад', 'action' => $this->getLink('menu')]],
        ]);
    }

    protected function stepFaq(?string $message, array $params): BotResponse
    {
        $text = <<<TEXT
❓ Часто задаваемые вопросы

Q: Безопасно ли подключать Telegram?
A: Да, мы используем официальный API и не храним пароли.

Q: Могу ли я отслеживать приватные группы?
A: Да, если вы являетесь их участником.

Q: Как отменить подписку?
A: В разделе "Тарифы" → "Управление подпиской".
TEXT;

        return BotResponse::messageWithKeyboard($text, [
            [['text' => '📞 Написать в поддержку', 'action' => $this->getLink('support')]],
            [['text' => '◀️ Назад', 'action' => $this->getLink('menu')]],
        ]);
    }

    protected function stepSupport(?string $message, array $params): BotResponse
    {
        $text = <<<TEXT
📞 Поддержка

Если у вас возникли вопросы или проблемы:

📧 Email: support@aivalone.com
💬 Telegram: @aivalone_support

Среднее время ответа: 2-4 часа
TEXT;

        return BotResponse::messageWithKeyboard($text, [
            [['text' => '✉️ Написать', 'action' => 'url:https://t.me/aivalone_support'],
            [['text' => '◀️ Назад', 'action' => $this->getLink('menu')]],
        ]);
    }
}
```

## Использование как Fallback

Router использует HelpConversation при ошибках:

```php
// В Router
private function handleFallback(BotRequest $request): BotResponse
{
    $help = $this->registry->get('help');
    return $help->handle(null, null, []);
}
```

## Связанные документы

* [Conversations Overview](overview.md)
* [AbstractConversation](abstract-conversation.md)
* [Router](../services/router.md)

## Статус реализации

* [ ] Класс HelpConversation создан
* [ ] Шаги menu, getting_started, pricing, faq, support реализованы
* [ ] getMenuInfo возвращает данные для меню
* [ ] Используется как fallback в Router
* [ ] Unit тесты написаны
