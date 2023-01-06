# Extensions - Monolog

The Web and GRPC bundles come with default integration with the Monolog https://github.com/Seldaek/monolog for
log management.

## Configuration

The extension can be configured using either a configuration file or a bootloader. The configuration file for this
extension should be located in `app/config/monolog.php`. In this file, you can specify the default Monolog handler,
global logging level, as well as configure handlers and processors.

For example, the configuration file might have the following format:

```php
<?php

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
     * @see https://github.com/Seldaek/monolog/blob/main/doc/01-usage.md#log-levels
     */
    'globalLevel' => \Monolog\Logger::toMonologLevel(env('MONOLOG_DEFAULT_LEVEL', \Monolog\Logger::DEBUG)),

    /**
     * -------------------------------------------------------------------------
     *  Handlers
     * -------------------------------------------------------------------------
     *
     * @see https://github.com/Seldaek/monolog/blob/main/doc/02-handlers-formatters-processors.md#handlers
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
        ]
    ],

    /**
     * -------------------------------------------------------------------------
     *  Processors
     * -------------------------------------------------------------------------
     *
     * Processors allows adding extra data for all records.
     *
     * @see https://github.com/Seldaek/monolog/blob/main/doc/02-handlers-formatters-processors.md#processors
     */
    'processors' => [
        'default' => [
            [
                'class' => \Monolog\Processor\PsrLogMessageProcessor::class,
                'options' => [
                    'dateFormat' => 'Y-m-d\TH:i:s.uP',
                ],
            ],
        ],
    ],
];
```

> **Note**
> 
> Use `MONOLOG_DEFAULT_CHANNEL` env variable to specify the default Monolog logger that should be used in your
> application.

## Register handler

Use `Spiral\Monolog\Bootloader\MonologBootloader` also to declare a handler and a log-formatter for a specific channel.

```php
namespace App\Application\Bootloader;

use Spiral\Boot\Bootloader\Bootloader;
use Spiral\Monolog\Bootloader\MonologBootloader;

class LoggingBootloader extends Bootloader
{
    protected const DEPENDENCIES = [
        MonologBootloader::class
    ];
    
    public function boot(MonologBootloader $monolog): void
    {
        $monolog->addHandler(
            'my-channel',
            $monolog->logRotate(directory('runtime') . 'logs/my-channel.log')
        );
    }
}
```

> **Note**
> 
> Make sure to add your Bootloader `App\Application\Bootloader\LoggingBootloader` at the top of the `Kernel::LOAD` list.


## Register RoadRunner log handler

The `Spiral\Monolog\Bootloader\MonologBootloader` allows registering of different Monolog log handlers. For instance,
you can register a log handler for [RoadRunner](https://roadrunner.dev/docs/plugins-applogger/2.x/en) using the
`Spiral\RoadRunnerBridge\Logger\Handler` class.

The following code sample demonstrates how this can be achieved through the use of the `LoggingBootloader`:

```php
namespace App\Application\Bootloader;

use Spiral\Boot\Bootloader\Bootloader;
use Spiral\Monolog\Bootloader\MonologBootloader;
use Spiral\RoadRunnerBridge\Logger\Handler as RoadRunnerHandler;
use Spiral\RoadRunnerBridge\Bootloader\LoggerBootloader;

class LoggingBootloader extends Bootloader
{
    protected const DEPENDENCIES = [
        MonologBootloader::class,
        LoggerBootloader::class,
    ];
    
    public function boot(MonologBootloader $monolog, RoadRunnerHandler $handler): void
    {
        $monolog->addHandler('roadrunner', $handler);
    }
}
```

And then change the default monolog handler in the `.env` file:

```dotenv
MONOLOG_DEFAULT_CHANNEL=roadrunner
```

> **Note**
>
> Make sure that you have the `spiral/roadrunner-bridge` package installed. This package provides the necessary classes 
> and functions needed to integrate RoadRunner with Monolog.


## Write to Log

You receive the `default` logger using the `Psr\Log\LoggerInterface` dependency:

```php
use Psr\Log\LoggerInterface;

// ...
public function index(LoggerInterface $logger): void
{
    $logger->alert('message');
}
```

To get a logger for a specific channel, use `Spiral\Logger\LogsInterface`:

```php
use Spiral\Logger\LogsInterface;

// ...
public function index(LogsInterface $logs): void
{
    $logs->getLogger('my-channel')->alert('message');
}
```

## LoggerTrait

You can use `Spiral\Logger\Traits\LoggerTrait` to quickly assign a Logger to any class. The class name will be used as
the channel name:

```php
use Spiral\Logger\Traits\LoggerTrait;

class HomeController
{
    use LoggerTrait;

    public function index(): void
    {
        $this->getLogger()->alert('message');
    }
}
```

And assign a logger to it:

```php
public function boot(MonologBootloader $monolog): void
{
    $monolog->addHandler(
        HomeController::class,
        $monolog->logRotate(directory('runtime') . 'logs/home-controller.log') // handler
    );
}
```

> **Note**
> 
> LoggerTrait only works inside the `$app->serve()` method via the global IoC scope.

## Errors

To aggregate all application errors into a single log file, subscribe to the `default` channel:

```php
namespace App\Bootloader;

use Monolog\Logger;
use Spiral\Boot\Bootloader\Bootloader;
use Spiral\Monolog\Bootloader\MonologBootloader;

class LoggingBootloader extends Bootloader
{
    protected const DEPENDENCIES = [
        MonologBootloader::class
    ];

    public function boot(MonologBootloader $monolog): void
    {
        $monolog->addHandler(
            'default',
            $monolog->logRotate(directory('runtime') . 'logs/errors.log', Logger::ERROR) // only ERROR and above
        );
    }
}
```