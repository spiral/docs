# SendIt

The Spiral Framework provides a simple component for creating and sending emails. The component is available by default in
the [application bundle](https://github.com/spiral/app).

## Installation

To enable the component, you just need to add `Spiral\SendIt\Bootloader\BuilderBootloader`
and`Spiral\SendIt\Bootloader\MailerBootloader`
classes to the bootloaders list, which is located in the class of your application.

```php
namespace App;

use Spiral\SendIt\Bootloader\BuilderBootloader;
use Spiral\SendIt\Bootloader\MailerBootloader;

class App extends Kernel
{
    protected const LOAD = [
        // ...
        BuilderBootloader::class,
        MailerBootloader::class,
        // ...
    ];
}
```

The `BuilderBootloader` bootloader registers package views and provides an ability to generate email templates using
the `Stempler` template engine. The `MailerBootloader` configures a queue for sending emails.

> **Note**
> You can disable one of the bootloaders if you don't need some functionality.

## Configuration

The configuration file for this component should be located at `app/config/mailer.php`. Within this file, you may
configure the `dsn`, `from`, `queue`, `queueConnection` parameters.

For example, the configuration file might look like this:

```php
return [
    /**
     * -------------------------------------------------------------------------
     *  Transport setup
     * -------------------------------------------------------------------------
     * 
     * The MAILER_DSN isn't a real address: it's a convenient format that offloads most of the configuration work to mailer.
     * Supported formats are compatible with the symfony/mailer component. 
     */
    'dsn' => env('MAILER_DSN'),
    
    /**
     * -------------------------------------------------------------------------
     *  From
     * -------------------------------------------------------------------------
     * 
     * Instead of calling ->from() on each Email you create, you can configure this value globally. 
     */
    'from' => env('MAILER_FROM'),
    
    /**
     * -------------------------------------------------------------------------
     *  Configuration queue connection
     * -------------------------------------------------------------------------
     * 
     * This section allows you to configure the `queue` and the `driver` for sending emails.
     * The `queueConnection` with the value `sync` allows you to send emails without using a queue.
     */
    'queue' => env('MAILER_QUEUE'), // won't be used for sync queue
    'queueConnection' => env('MAILER_QUEUE_CONNECTION', 'sync'),
];
```

### DSN

The `MAILER_DSN` isn't a real address: it's a convenient format that offloads most of the configuration work to mailer.

| DSN protocol | Example                                | Description                                                                                                                                                                                                                |
|--------------|----------------------------------------|----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| smtp         | `smtp://user:pass@smtp.example.com:25` | Mailer uses an SMTP server to send emails                                                                                                                                                                                  |
| sendmail     | `sendmail://default`                   | Mailer uses the local sendmail binary to send emails                                                                                                                                                                       |
| native       | `native://default`                     | Mailer uses the sendmail binary and options configured in the `sendmail_path setting of `php.ini`. On Windows hosts, Mailer fallbacks to `smtp` and `smtp_port` `php.ini` settings when `sendmail_path` is not configured. |


> **Note**
> You can find out more about DSN transports on the
> official `symfony/mailer` [documentation](https://symfony.com/doc/current/mailer.html#using-built-in-transports)

### Custom mailer transport

Spiral Framework provides an ability to customize mailer transport. You just need to bind your implementation of 
`Symfony\Component\Mailer\Transport\TransportInterface` via container.

**Simple example**

```php
use Spiral\Boot\Bootloader\Bootloader;
use Symfony\Component\Mailer\Transport\TransportInterface;
use Symfony\Component\Mailer\Transport\RoundRobinTransport;
use Symfony\Component\Mailer\Transport;

class AppBootloader extends Bootloader
{
    protected const DEPENDENCIES = [
        \Spiral\SendIt\Bootloader\MailerBootloader::class,
    ];
    
    protected const SINGLETONS = [
        TransportInterface::class => [self::class, 'initTransport'],
    ];
    
    public function initTransport(MailerConfig $config): TransportInterface
    {
        $transports = [];

        foreach($config->getDsns() as $dsn) {
            $transports[] = Transport::fromDsn($dsn);
        }

        return new RoundRobinTransport(
            $transports
        );
}


```

### Queue

The `queueConnection` key uses a specifying queue that will be used for handling mail messages. By default, the messages 
will be sent using `sync` connection.

> **Note**
> Read more about queue connection configuration [here](/queue/configuration.md).

## Usage

The component provides an ability to compose content-rich email templates using `Stempler` views:

```html
<extends:sendit:builder subject="I'm afraid I can't do that"/>
<use:bundle path="sendit:bundle"/>

<email:attach path="{{ $attachment }}" name="attachment.txt"/>

<block:html>
    <p>I'm sorry, {{ $name }}!</p>
    <p><email:image path="path/to/image.png"/></p>
</block:html>
```

To use:

```php
use Spiral\Mailer\MailerInterface;
use Spiral\Mailer\Message;

public function send(MailerInterface $mailer): void
{
    $mailer->send(new Message(
        'template.dark.php', 
        'email@domain.com',
        [
            'name' => 'Dave',
            'attachment' => __FILE__,
        ]
    ));
}
```

### Sending messages with a delay

The component allows sending messages with a delay. The delay time is set using the `setDelay` method:

```php
use Spiral\Mailer\Message;

$message = new Message('test', 'email@domain.com');
$message->setDelay(new \DateTimeImmutable('+ 60 minute'));
// or
$message->setDelay(new \DateInterval('PT60S'));
// or
$message->setDelay(100);
```

## Events

| Event                              | Description                                          |
|------------------------------------|------------------------------------------------------|
| Spiral\SendIt\Event\MessageSent    | The Event will be fired `after` sending the message. |
| Spiral\SendIt\Event\MessageNotSent | The Event is fired if the message could not be sent. |
