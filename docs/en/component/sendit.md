# SendIt

The Spiral Framework provides a simple component for creating and sending emails. The component available by default in
the [application bundle](https://github.com/spiral/app).

## Installation

To enable the component, you just need to add `Spiral\SendIt\Bootloader\BuilderBootloader` and`Spiral\SendIt\Bootloader\MailerBootloader` 
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

The `BuilderBootloader` bootloader registers package views and provides the ability to generate email templates using 
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

## Usage

The component provides the ability to compose content-rich email templates using `Stempler` views:

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
