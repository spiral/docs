# The Basics — Logging

Spiral offers `spiral/logger` component that is compliant with the [PSR-3](https://www.php-fig.org/psr/psr-3/) standard.
This component can be utilized to log various types of information, such as errors, warnings, and debugging messages, 
which can assist in identifying and resolving issues within the application.

By default, the framework does not provide its own implementation, however, the `spiral/monolog-bridge` component is
available, which fully integrates with the [Seldaek/monolog](https://github.com/Seldaek/monolog) package and offers
support for various powerful log handlers.

The framework makes it simple to configure these handlers, allowing for customization of log handling through the use of
different channels.

## Configuration

To configure this component, it can be set up to your preference via a config file or a bootloader. The config file for
this component is typically located in `app/config/monolog.php`. Through this file, you can select a default handler,
set a global logging level, and customize handlers and processors to meet your specific needs.

Here is an example of a config file:

```php app/config/monolog.php
use Monolog\Handler\ErrorLogHandler;
use Monolog\Logger;
use Monolog\Processor\PsrLogMessageProcessor;

return [
    /**
     * -------------------------------------------------------------------------
     *  Default Monolog handler
     * -------------------------------------------------------------------------
     */
    'default' => env('MONOLOG_DEFAULT_CHANNEL', 'default'),

    /**
     * -------------------------------------------------------------------------
     *  Global logging level
     * -------------------------------------------------------------------------
     *
     * Monolog supports the logging levels described by RFC 5424.
     *
     * @see https://seldaek.github.io/monolog/doc/01-usage.html#log-levels
     */
    'globalLevel' => Logger::toMonologLevel(
        env('MONOLOG_DEFAULT_LEVEL', \Monolog\Logger::DEBUG)
    ),

    /**
     * -------------------------------------------------------------------------
     *  Handlers
     * -------------------------------------------------------------------------
     *
     * @see https://seldaek.github.io/monolog/doc/02-handlers-formatters-processors.html#handlers
     */
    'handlers' => [
        'default' => [
            [
                'class' => 'log.rotate',
                'options' => [
                    'filename' => directory('runtime') . 'logs/app.log',
                    'level' => \Monolog\Logger::DEBUG,
                ],
            ],
        ],
        'stderr' => [
            ErrorLogHandler::class,
        ],
        'stdout' => [
            [
                'class' => SyslogHandler::class,
                'options' => [
                    'ident' => 'app',
                    'facility' => LOG_USER,
                ],
            ],
        ],
    ],

    /**
     * -------------------------------------------------------------------------
     *  Processors
     * -------------------------------------------------------------------------
     *
     * Processors allows adding extra data for all records.
     *
     * @see https://seldaek.github.io/monolog/doc/02-handlers-formatters-processors.html#processors
     */
    'processors' => [
        'default' => [
            [
                'class' => PsrLogMessageProcessor::class,
                'options' => [
                    'dateFormat' => 'Y-m-d\TH:i:s.uP',
                ],
            ],
        ],
    ],
];
```

> **Note**
> Use `MONOLOG_DEFAULT_CHANNEL` env variable to specify the default Monolog logger that should be used in your
> application.

## Register handler

In addition to configuring the logging component through a config file or bootloader, you can also register handlers via
the `Spiral\Monolog\Bootloader\MonologBootloader` class.

### Log rotate handler

```php app/src/Application/Bootloader/LoggingBootloader.php
namespace App\Application\Bootloader;

use Spiral\Boot\Bootloader\Bootloader;
use Spiral\Monolog\Bootloader\MonologBootloader;

final class LoggingBootloader extends Bootloader
{
    public function boot(MonologBootloader $monolog): void
    {
        $monolog->addHandler(
            'my-channel',
            $monolog->logRotate(directory('runtime') . 'logs/my-channel.log')
        );
    }
}
```

> **Warning**
> Don't forget to add this bootloader to the top of bootloaders list in `app/src/Application/Kernel.php`:

```php app/src/Application/Kernel.php
protected const LOAD = [
    \App\Application\Bootloader\LoggingBootloader::class,
    // ...
];
```

### RoadRunner handler

The RoadRunner bridge package provides `Spiral\RoadRunnerBridge\Logger\Handler` handler for sending logs to 
the [RoadRunner app logger](https://roadrunner.dev/docs/plugins-applogger).

You just need to add `Spiral\RoadRunnerBridge\Bootloader\LoggerBootloader` to the top of bootloaders list:

```php app/src/Application/Kernel.php
protected const LOAD = [
    \Spiral\RoadRunnerBridge\Bootloader\LoggerBootloader::class,
    // ...
];
```

> **Warning**
> Make sure that you have the [spiral/roadrunner-bridge](../start/server.md#roadrunner-bridge) package installed.
> This package provides the necessary classes to integrate RoadRunner with Monolog.

And change the default channel to `roadrunner`:

:::: tabs

::: tab Environment
```dotenv .env
MONOLOG_DEFAULT_CHANNEL=roadrunner
```
:::

::: tab Config
```php app/config/monolog.php
return [
    'default' => 'roadrunner',
    // ...
];
```
:::

::::

By default, the handler will format the log message using the following structure `%message% %context% %extra%\n`

If you want to change the log message structure, you can use the `LOGGER_FORMAT` evn variable:

```dotenv .env
LOGGER_FORMAT=[%datetime%] %channel%.%level%: %message% %context% %extra%\n
```

> **See more**
> Read more about the available placeholders in 
> the [Monolog documentation](https://seldaek.github.io/monolog/doc/message-structure.html).

## Usage

### Send logs to the default channel

In order to use the logging component, the framework utilizes the `Psr\Log\LoggerInterface` class, which can be used to
log messages to the default channel.

```php
use Psr\Log\LoggerInterface;

final class UserService
{
    public function __construct(
        private readonly LoggerInterface $logger
    ) {
    }

    public function register(string $email, string $password): void
    {
        // Register user ...
        
        $this->logger->info('User has been registered', ['email' => $email]);
    }
}
```

### Send logs to a specific channel

In case you want to log to a specific channel other than the default channel, you can use
the `Spiral\Logger\LogsInterface` class to get a logger for that specific channel.

```php
use Psr\Log\LoggerInterface;
use Spiral\Logger\LogsInterface;

final class UserService
{
    private readonly LoggerInterface $logger;
    public function __construct(LogsInterface $logs) 
    {
        $this->logger = $logs->channel('my-channel');
    }

    public function register(string $email, string $password): void
    {
        // Register user ...
        
        $this->logger->info('User has been registered', ['email' => $email]);
    }
}
```

### Logger trait

Spiral provides a convenient way to quickly assign a Logger to any class through the use of the
`Spiral\Logger\Traits\LoggerTrait` trait. By simply including this trait in a class you can easily access a Logger
instance and log messages.

> **Warning**
> The **channel name** used for logging will be the class name by default. This trait enables you to log messages from
> any class without the need to explicitly inject the Logger in the constructor.

```php
use Spiral\Logger\Traits\LoggerTrait;

final class UserService
{
    use LoggerTrait;

    public function register(string $email, string $password): void
    {
        // Register user ...
        
        $this->getLogger()->info('User has been registered', ['email' => $email]);
    }
}
```

And assign a logger to it:

```php app/src/Application/Bootloader/LoggingBootloader.php
namespace App\Application\Bootloader;

use Spiral\Boot\Bootloader\Bootloader;
use Spiral\Monolog\Bootloader\MonologBootloader;

final class LoggingBootloader extends Bootloader
{
    // ...
    
    public function boot(MonologBootloader $monolog): void
    {
        $monolog->addHandler(
            UserService::class,
            $monolog->logRotate(directory('runtime') . 'logs/user-dervice.log')
        );
    }
}
```

> **Warning**
> LoggerTrait only works inside the global [IoC scope](../framework/scopes.md).

### Handling only specific log levels

In some cases, you may want to log only specific log levels. For example, you may want to aggregate only application
errors into a single log file.

**To do this, you can subscribe to the default channel:**

```php app/src/Application/Bootloader/LoggingBootloader.php
namespace App\Application\Bootloader;

use Monolog\Logger;
use Spiral\Boot\Bootloader\Bootloader;
use Spiral\Monolog\Bootloader\MonologBootloader;

final class LoggingBootloader extends Bootloader
{
    // ...
    
    public function boot(MonologBootloader $monolog): void
    {
        $monolog->addHandler(
            'default',
            $monolog->logRotate(directory('runtime') . 'logs/errors.log', Logger::ERROR) // only ERROR and above
        );
    }
}
```