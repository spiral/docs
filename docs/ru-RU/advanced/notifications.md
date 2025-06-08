# Продвинутые возможности — Уведомления

Spiral имеет довольно крутую систему уведомлений, которая позволяет отправлять всевозможные сообщения вашим пользователям через различные каналы. Вы можете отправлять электронные письма, SMS-сообщения, сообщения в Slack, push-уведомления и многое другое, используя пакет [spiral-packages/notifications](https://github.com/spiral-packages/notifications). Он работает на компоненте под названием Symfony Notifier и очень прост в использовании.

Вам просто нужно установить пакет, зарегистрировать его в вашем приложении, и всё готово!

## Установка

Чтобы установить пакет, вы можете использовать команду composer:

```terminal
composer require spiral-packages/notifications
```

После установки пакета вам необходимо зарегистрировать загрузчик (bootloader) в списке загрузчиков вашего приложения:

:::: tabs

::: tab Использование метода

```php app/src/Application/Kernel.php
public function defineBootloaders(): array
{
    return [
        // ...
        \Spiral\Notifications\Bootloader\NotificationsBootloader::class,
        // ...
    ];
}
```

Читайте больше о загрузчиках в разделе [Фреймворк — Загрузчики](../framework/bootloaders.md).
:::

::: tab Использование константы

```php app/src/Application/Kernel.php
protected const LOAD = [
    // ...
    \Spiral\Notifications\Bootloader\NotificationsBootloader::class,
    // ...
];
```

Читайте больше о загрузчиках в разделе [Фреймворк — Загрузчики](../framework/bootloaders.md).
:::

::::

После выполнения этих шагов вы полностью интегрируете пакет уведомлений в ваше приложение.

## Конфигурация

Чтобы полностью использовать возможности пакета, вам нужно настроить различные каналы, транспорты и политики в файле конфигурации.

Этот файл расположен по пути `app/config/notifications.php` и позволяет указать, как и куда должны отправляться уведомления, а также любые дополнительные параметры, которые могут понадобиться.

Вот пример файла конфигурации:

```php app/config/notifications.php
use Symfony\Component\Notifier\Channel\BrowserChannel;
use Symfony\Component\Notifier\Channel\ChatChannel;
use Symfony\Component\Notifier\Channel\EmailChannel;
use Symfony\Component\Notifier\Channel\PushChannel;
use Symfony\Component\Notifier\Channel\SmsChannel;

return [
    'channels' => [
        'nexmo_sms' => [
            'type' => 'sms',
            'transport' => 'nexmo',
        ],
        'default_email' => [
            'type' => 'email',
            'transport' => 'smtp',
        ],
        'roundrobin_email' => [
            'type' => 'email',
            'transport' => ['smtp', 'smtp_1'], // будет использован алгоритм roundrobin
        ],
        'chat/slack' => [
            'type' => 'chat',
            'transport' => 'slack',
        ],
    ],

    'transports' => [
        'nexmo' => 'nexmo://KEY:SECRET@default?from=FROM',
        'smtp' => 'smtp://user:pass@smtp.example.com:25',
        'smtp_1' => 'smtp://user:pass@smtp.example.com:25',
        'slack' => 'slack://TOKEN@default?channel=CHANNEL'
    ],

    'policies' => [
        'urgent' => ['sms', 'chat/slack', 'email'],
        'high' => ['chat/slack', 'push/firebase'],
    ],

    'queueConnection' => env('NOTIFICATIONS_QUEUE_CONNECTION', 'sync'),

    'typeAliases' => [
        'browser' => BrowserChannel::class,
        'chat' => ChatChannel::class,
        'email' => EmailChannel::class,
        'push' => PushChannel::class,
        'sms' => SmsChannel::class,
    ],
];
```

### Типы каналов

Пакет поддерживает множество каналов уведомлений, каждый со своими уникальными возможностями и опциями интеграции.

#### Эти каналы включают:

- **SMS канал**: позволяет отправлять SMS-сообщения на номера телефонов. Может быть интегрирован с провайдерами, такими как Twilio, для отправки сообщений.
- **Chat канал**: отправляет уведомления в чат-сервисы, такие как Slack и Telegram.
- **Email канал**: интегрируется с Symfony Mailer для отправки email уведомлений.
- **Browser канал**: использует flash-сообщения для отправки уведомлений в браузер пользователя.
- **Push канал**: позволяет отправлять push-уведомления на мобильные устройства и веб-браузеры.

> **Читайте больше**
> Читайте больше о типах каналов в официальной [документации Symfony](https://symfony.com/doc/current/notifier.html).

В файле конфигурации все каналы, которые могут использоваться для отправки уведомлений, регистрируются в секции `typeAliases`. Эта секция представляет собой массив, где **ключ** представляет тип канала, а **значение** представляет класс, который будет обрабатывать канал.

Ключи в массиве `typeAliases` должны соответствовать `type`, определенному в секции `channels` файла конфигурации, чтобы система знала, какой класс использовать для каждого канала.

### Транспорты

Секция `transports` определяет различные сервисы, которые могут использоваться для отправки уведомлений, такие как nexmo, smtp, slack и т.д. Каждый транспорт имеет уникальный ключ, а значение — это строка подключения, которая содержит учетные данные, необходимые для подключения к сервису.

Например:

```php
'transports' => [
    'nexmo' => 'nexmo://KEY:SECRET@default?from=FROM',
    'smtp' => 'smtp://user:pass@smtp.example.com:25',
    'smtp_1' => 'smtp://user:pass@smtp.example.com:25',
    'slack' => 'slack://TOKEN@default?channel=CHANNEL'
],
```

> **Примечание**
> Полный список доступных транспортов вы можете увидеть, перейдя по [ссылке](https://symfony.com/doc/current/notifier.html#channels-chatters-texters-email-browser-and-push).

### Каналы

В файле конфигурации вы можете зарегистрировать столько каналов, сколько нужно для вашего приложения. Каждый канал определяется в массиве, где ключ — это имя канала.

Давайте посмотрим на пример:

```php app/config/notifications.php
'email' => [
    'type' => 'email',
    'transport' => 'smtp',
],
```

- Ключ `email` — это имя канала, которое вы можете использовать для маршрутизации уведомлений в вашем классе уведомлений.
- Ключ `type` — это тип канала, куда будет отправлено уведомление.
- Ключ `transport` указывает транспорт, который будет использоваться для отправки уведомлений. Если вы установите значение как массив, пакет будет использовать алгоритм round-robin для выбора транспорта. Это означает, что пакет будет циклически проходить через массив транспортов, используя один транспорт для одного уведомления, затем следующий транспорт для следующего уведомления и так далее.

Например, если у вас есть следующая конфигурация:

```php app/config/notifications.php
'roundrobin_email' => [
  'type' => 'email',
  'transport' => ['smtp', 'smtp_1'],
],
```

### Политики

Секция `policies` определяет различные политики уведомлений, которые указывают, какие каналы должны использоваться для различных типов уведомлений.

```php app/config/notifications.php
'policies' => [
    'urgent' => ['sms', 'chat/slack', 'email'],
    'high' => ['chat/slack', 'push/firebase'],
],
```

Например, политика `urgent` указывает, что должны использоваться уведомления `SMS`, `chat/slack` и `email`.

## Использование

Чтобы отправить уведомление с помощью пакета, вам нужны и получатель, и уведомление.

### Получатель

Получатель должен реализовывать интерфейс `Symfony\Component\Notifier\Recipient\RecipientInterface`.

Если получатель должен получать SMS-уведомления, он также должен реализовывать интерфейс `Symfony\Component\Notifier\Recipient\SmsRecipientInterface`, который определяет дополнительные методы, специфичные для SMS-уведомлений. Аналогично, для email-уведомлений получатель должен реализовывать интерфейс `Symfony\Component\Notifier\Recipient\EmailRecipientInterface`.

Это гарантирует, что у пакета есть вся необходимая информация для отправки уведомления правильному получателю через правильный канал.

Вот пример класса пользователя, который может быть получателем уведомлений:

```php
use Symfony\Component\Notifier\Recipient\RecipientInterface;
use Symfony\Component\Notifier\Recipient\SmsRecipientInterface;

final class User implements RecipientInterface, SmsRecipientInterface
{
    // ...

    public function getPhone(): string
    {
        return '+8(000)000-00-00';
    }
}
```

### Уведомление

Класс уведомления должен расширять класс `Symfony\Component\Notifier\Notification\Notification`, этот класс предоставляет базовые методы, которые должен иметь класс уведомления.

> **Читайте больше**
> Читайте больше о классе уведомлений в [официальной документации](https://symfony.com/doc/current/notifier.html#creating-sending-notifications).

```php
use Symfony\Component\Notifier\Notification\Notification;
use Symfony\Component\Notifier\Notification\SmsNotificationInterface;
use Symfony\Component\Notifier\Message\SmsMessage;

class UserBannedNotification extends Notification implements SmsNotificationInterface
{
    public function getChannels(RecipientInterface $recipient): array
    {
        if ($recipient instanceof SmsRecipientInterface) {
            return ['nexmo_sms'];
        }
        
        return ['chat/slack'];
    }

    public function asSmsMessage(SmsRecipientInterface $recipient, string $transport = null): ?SmsMessage
    {
        return SmsMessage::fromNotification($this, $recipient);
    }
}
```

Метод `getChannels()` позволяет указать, на какие каналы должно быть отправлено уведомление. В этом примере уведомление будет отправлено как на SMS, так и на чат-каналы.

Вы можете определить `getImportance()` вместо `getChannels()`. Метод позволяет указать уровень срочности уведомления, в данном случае он установлен как `urgent`. Этот уровень важности может быть определен в секции `policies` файла конфигурации.

```php
use Symfony\Component\Notifier\Notification\Notification;
use Symfony\Component\Notifier\Notification\SmsNotificationInterface;
use Symfony\Component\Notifier\Message\SmsMessage;

class UserBannedNotification extends Notification implements SmsNotificationInterface
{
    public function getImportance(): string
    {
        return 'urgent';
    }

    public function asSmsMessage(SmsRecipientInterface $recipient, string $transport = null): ?SmsMessage
    {
        return SmsMessage::fromNotification($this, $recipient);
    }
}
```

### Отправка уведомления

После создания класса уведомления и класса получателя вы можете отправить уведомление.

```php
use Symfony\Component\Notifier\NotifierInterface;

final class UserBanService {

    public function __construct(
        private readonly UserRepository $repository
        private readonly NotifierInterface $notifier
    ) {}

    public function handle(string $userUuid): void
    {
        $user = $this->repository->findByPK($userUuid);

        $this->notifier->send(
            new UserBannedNotification(subject: 'Your profile banned for activity that violates rules'),
            $user
        );
    }
}
```

Вы также можете отправить уведомление через очередь:

```php
$this->notifier->sendQueued(
    new UserBannedNotification(subject: 'Your profile banned for activity that violates rules'),
    $user
);
```

> **Примечание**
> Уведомления в очереди будут отправлены через `queueConnection` из конфигурации уведомлений.

## Пользовательский транспорт уведомлений

В некоторых случаях вам может понадобиться использовать пользовательские транспорты, которые не предоставляются компонентом `symfony/notifier`. В этом случае вы можете зарегистрировать пользовательский транспорт, используя интерфейс `Spiral\Notifications\NotificationTransportRegistryInterface`.

```php
use Spiral\Boot\Bootloader\Bootloader;
use Spiral\Notifications\NotificationTransportRegistryInterface;
use Spacetab\SmsaeroNotifier\SmsaeroTransportFactory;

class MyBootloader extends Bootloader
{
    public function boot(NotificationTransportRegistryInterface $registry): void
    {
        $registry->registerTransport(new SmsaeroTransportFactory());
    }
}
```
