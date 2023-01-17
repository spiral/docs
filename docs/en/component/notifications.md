# Component — Notifications

The Spiral Framework has a pretty cool notifications system that lets you send all kinds of messages to your users
through different channels. You can send emails, SMS messages, Slack messages, push notifications, and more all through
the [spiral-packages/notifications](https://github.com/spiral-packages/notifications) package. It's powered by a
component called the Symfony Notifier, and it's super easy to use.

You just need to install the package, register it in your application and you're good to go!

## Installation

To install the package, you can use the composer command:

```terminal
composer require spiral-packages/notifications
```

Once the package is installed, you will need to register the bootloader in your application's bootloader list:

```php app/src/Application/Kernel.php
protected const LOAD = [
    // ...
    \Spiral\Notifications\Bootloader\NotificationsBootloader::class,
];
```

With these steps completed, you will have fully integrated the notifications package into your application.

## Configuration

To fully utilize the capabilities of the package, you'll need to set up the various channels, transports, and policies
in a configuration file.

This file located at `app/config/notifications.php`, allows you to specify how and where notifications should be sent,
as well as any additional options that might be needed.

Here is an example of configuration file:

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
            'transport' => ['smtp', 'smtp_1'], // will be used roundrobin algorithm
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

### Channel types

The package supports a variety of notification channels, each with its own unique capabilities and integration options.

#### These channels include:

- **SMS channel**: allows you to send SMS messages to phone numbers. It can be integrated with providers such as Twilio
  to send the messages.
- **Chat channel**: sends notifications to chat services such as Slack and Telegram.
- **Email channel**: integrates with the Symfony Mailer to send email notifications.
- **Browser channel**: uses flash messages to send notifications to the user's browser.
- **Push Channel**: allows you to send push notifications to mobile devices and web browsers.

> **Note**
> Read more about channel types in the official [Symfony documentation](https://symfony.com/doc/current/notifier.html)

In the configuration file, all the channels that can be used for sending notifications are registered in
the `typeAliases`section. This section is an array where the **key** represents the type of the channel and
the **value** represents the class that will handle the channel.

The keys in the `typeAliases` array must match the `type` defined in the `channels` section of the config file, so that
the system knows which class to use for each channel.

### Transports

The `transports` section defines the various services that can be used to send notifications, such as nexmo, smtp,
slack, etc. Each transport has a unique key, and the value is the connection string which contains the credentials
needed to connect to the service.

For example:

```php
'transports' => [
    'nexmo' => 'nexmo://KEY:SECRET@default?from=FROM',
    'smtp' => 'smtp://user:pass@smtp.example.com:25',
    'smtp_1' => 'smtp://user:pass@smtp.example.com:25',
    'slack' => 'slack://TOKEN@default?channel=CHANNEL'
],
```

> **Note**
> Full list of available transports you can see by
> following [link](https://symfony.com/doc/current/notifier.html#channels-chatters-texters-email-browser-and-push).

### Channels

In the configuration file, you can register as many channels as you need for your application. Each channel is defined
in an array with the key being the name of the channel.

Let's take a look on an example:

```php app/config/notifications.php
'email' => [
    'type' => 'email',
    'transport' => 'smtp',
],
```

- The `email` key is the name of the channel that you can use to route notifications in your notification class.
- The `type` key is the type of channel where the notification will be sent.
- The `transport` key specifies the transport that will be used to send the notifications. If you set the value
  as an array, the package will use a round-robin algorithm for selecting the transport. This means that the package
  will cycle through the array of transports, using one transport for one notification, then the next transport for the
  next notification and so on.

For example, if you have the following configuration:

```php app/config/notifications.php
'roundrobin_email' => [
  'type' => 'email',
  'transport' => ['smtp', 'smtp_1'],
],
```

### Policies

The `policies` section defines different notification policies, which specify which channels should be used for
different types of notifications.

```php app/config/notifications.php
'policies' => [
    'urgent' => ['sms', 'chat/slack', 'email'],
    'high' => ['chat/slack', 'push/firebase'],
],
```

For example, the `urgent` policy specifies that `SMS`, `chat/slack`, and `email` notifications should be used.

## Usage

To send a notification using the package, you need to have both a recipient and a notification.

### Recipient

The recipient should implement the `Symfony\Component\Notifier\Recipient\RecipientInterface` interface.

If the recipient should receive SMS notifications, they should also implement
the `Symfony\Component\Notifier\Recipient\SmsRecipientInterface` interface which defines additional methods specific to
SMS notifications. Similarly, for email notifications, the recipient should implement
the `Symfony\Component\Notifier\Recipient\EmailRecipientInterface` interface.

This ensures that the package has all the necessary information to send the notification to the correct recipient
through the correct channel.

Here is an example of a user class that can be a recipient for notifications:

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

### Notification

A notification class should extend the `Symfony\Component\Notifier\Notification\Notification` class, this class provides
basic methods that a notification class should have.

> **Note**
> Read more about notification class in
> the [official documentation](https://symfony.com/doc/current/notifier.html#creating-sending-notifications).

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

The `getChannels()` method allows you to specify which channels the notification should be sent to. In this example, the
notification will be sent to both SMS and chat channels.

You can define `getImportance()` instead of `getChannels()`. The method allows you to specify the urgency level of the
notification, in this case it's set to `urgent`. This importance level can be defined in the config file's `policies`
section.

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

### Sending a notification

Once you have created a notification class and a recipient class, you can send the notification.

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

You can also send a notification via queue:

```php
$this->notifier->sendQueued(
    new UserBannedNotification(subject: 'Your profile banned for activity that violates rules'),
    $user
);
```

> **Note**
> Queued notification will be sent via `queueConnection` from notification config.

## Custom notification transport

In some cases, you may need to use custom transports that are not provided by the `symfony/notifier` component, In this
case, you can register a custom transport by using the `Spiral\Notifications\NotificationTransportRegistryInterface`
interface.

```php
use Spiral\Boot\Bootloader\Bootloader;
use Spiral\Notifications\NotificationTransportResolver;
use Spacetab\SmsaeroNotifier\SmsaeroTransportFactory;

class MyBootloader extends Bootloader
{
    public function boot(NotificationTransportRegistryInterface $registry): void
    {
        $resolver->registerTransport(new SmsaeroTransportFactory());
    }
}
```