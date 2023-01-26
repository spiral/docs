# Advanced — Mailer

The Spiral Framework provides a simple email API powered by the symfony/mailer component.

> **Note**
> The component is available by default in the [application bundle](https://github.com/spiral/app).


Emails are delivered via a "transport". Out of the box, you can deliver emails over SMTP by configuring the DSN in your
`.env` file (the user, pass and port parameters are optional).

## Installation

To enable the component, you need to add the `Spiral\SendIt\Bootloader\MailerBootloader` class to the bootloaders list.

> **See more**
> Read more about bootloaders in the [Framework — Bootloaders](../framework/bootloaders.md) section.
>

```php app/src/Application/Kernel.php
protected const LOAD = [
    // ...
    \Spiral\SendIt\Bootloader\MailerBootloader::class,
    // ...
];
```

This bootloader configures the base bindings, default settings, and a queue for sending emails. It will also
automatically register the `Spiral\SendIt\Bootloader\BuilderBootloader`, which registers the `spiral/views` component
and
provides the ability to create email templates using the **Stempler** template engine.

## Configuration

:::: tabs

::: tab Environment
You can configure the component via `.env` variables:

```dotenv
MAILER_DSN=smtp://username:password@example.com:25
MAILER_FROM=John Smith

MAILER_QUEUE_CONNECTION=roadrunner
MAILER_QUEUE=emails
```

:::

::: tab Config file
You can also configure the component via the `app/config/mailer.php` file.

Here is an example of the configuration file:

```php
<?php

declare(strict_types=1);

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

:::

::::

### DSN

The `MAILER_DSN` parameter is used to configure the transport for delivering emails. The component uses
the `symfony/mailer` component to send emails, and it supports several built-in transports, including `SMTP`,
`sendmail`, and the PHP `mail()` function.

> **Note**
> The DSN (Data Source Name) is a string that specifies the transport to be used, and can include options such as the
> user, password, and port. Here is an example: `smtp://username:password@example.com:25`

| DSN protocol | Example                                | Description                                                                                                                                                                                                                |
|--------------|----------------------------------------|----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| smtp         | `smtp://user:pass@smtp.example.com:25` | Mailer uses an SMTP server to send emails                                                                                                                                                                                  |
| sendmail     | `sendmail://default`                   | Mailer uses the local sendmail binary to send emails                                                                                                                                                                       |
| native       | `native://default`                     | Mailer uses the sendmail binary and options configured in the `sendmail_path setting of `php.ini`. On Windows hosts, Mailer fallbacks to `smtp` and `smtp_port` `php.ini` settings when `sendmail_path` is not configured. |

> **Note**
> You can find more information about the different types of transports supported by the `symfony/mailer` component, and
> how to configure them, in the
> official [Symfony documentation](https://symfony.com/doc/current/mailer.html#using-built-in-transports)

### Custom mailer transport

In addition to the built-in transports supported by the `symfony/mailer` component, the Spiral Framework also allows you
to use custom transports.

To use a custom transport, you will need to create a class that implements the
`Symfony\Component\Mailer\Transport\TransportInterface` which is a part of `symfony/mailer` component. This class should
handle the logic of sending an email using the desired transport.

Let's imagine that you have multiple DSN transports, and you want to use the Round Robin algorithm to select one of them
for sending emails.

Here is an example of how to use custom transport:

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
    
    public function initTransport(): TransportInterface
    {
        $transports = [];
        
        $dsns = [
            'smtp://...',
            'smtp://...',
            // ...
        ];

        foreach($dsns as $dsn) {
            $transports[] = Transport::fromDsn($dsn);
        }

        return new RoundRobinTransport($transports);
    }
}
```

> **Note**
> It's important to note that creating and configuring custom transports can be complex and may require deep
> understanding of Symfony Mailer component and its Transport interface

### Queue

Sending emails can be a time-consuming task, and if your application sends a large number of emails, it can slow down
the overall performance of your application. One way to mitigate this issue is to use queue jobs to send emails.

<details>
    <summary>Click to show benefits of using queue jobs to send emails</summary>

- **Improved performance:** By offloading the task of sending emails to a queue job, you can prevent the main
  application from being blocked while waiting for the email to be sent. This can improve the overall performance of
  your application.
- **Scalability:** If your application needs to send a large number of emails, using a queue job allows you to scale the
  number of worker processes to handle the load, without affecting the performance of the main application.
- **Retry mechanism:** If there is an error sending an email, the job can be set to retry at a later time, or after a
  certain number of attempts. This can be useful for handling temporary failures, such as when the email server is down.
- **Prioritizing:** Depending on the email priority you can sort the queue and process most important emails first.
- **Logging:** The queue job provide a logging mechanism to keep track of the status of the emails (sent, failed,
  retrying).

</details>

The Spiral Framework provides an easy way to set up a queue connection and pipeline for sending emails.

In the `.env` file, you can configure the queue connection and pipeline to use for sending emails by setting the
`MAILER_QUEUE_CONNECTION` and `MAILER_QUEUE` variables.

You can also configure the email queue in the `app/config/mailer.php` configuration file.

> **See more**
> Read more about queue connection configuration in
> the [Queue — Installation and Configuration](../queue/configuration.md) section.

## Usage

The component provides an ability to compose content-rich email templates using `Stempler` views:

```html

<extends:sendit:builder subject="I'm afraid I can't do that"/>
<use:bundle path="sendit:bundle"/>

<email:attach path="{{ $attachment }}" name="attachment.txt"/>

<block:html>
    <p>I'm sorry, {{ $name }}!</p>
    <p>
        <email:image path="path/to/image.png"/>
    </p>
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

> **Note**
> To learn more about dispatching events, see the [Events](../advanced/events.md) section in our documentation.
