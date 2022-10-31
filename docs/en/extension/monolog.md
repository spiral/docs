# Extensions - Monolog

Web and GRPC bundles include default integration with https://github.com/Seldaek/monolog to manage logs.

## Configuration

The extension can configure using a configuration file or a bootloader. 

The configuration file for this extension should be located at `app/config/monolog.php`. Within this file, you may
configure the default Monolog handler, global logging level, handlers and processors.

For example, the configuration file might look like this:

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
    'globalLevel' => Logger::toMonologLevel(env('MONOLOG_DEFAULT_LEVEL', Logger::DEBUG)),

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
                    'level' => Logger::DEBUG,
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
     * @see https://github.com/Seldaek/monolog/blob/main/doc/02-handlers-formatters-processors.md#processors
     */
    'processors' => [
        'default' => [
            // ...
        ],
        'stdout' => [
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

Use `Spiral\Monolog\Bootloader\MonologBootloader` to declare handler and log-formatter for specific channel:

```php
namespace App\Bootloader;

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
> You can use any monolog handler as the second argument. Make sure to add your Bootloader at the top of the LOAD list.

## Write to Log

You receive `default` logger using `Psr\Log\LoggerInterface` dependency:

```php
use Psr\Log\LoggerInterface;

// ...
public function index(LoggerInterface $logger): void
{
    $logger->alert('message');
}
```

To get logger for specific channel use `Spiral\Logger\LogsInterface`:

```php
use Spiral\Logger\LogsInterface;

// ...
public function index(LogsInterface $logs): void
{
    $logs->getLogger('my-channel')->alert('message');
}
```

Make sure to enable default handler to save content to the file:

```php
namespace App\Bootloader;

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
            $monolog->logRotate(directory('runtime') . 'logs/debug.log')
        );
    }
}
```

## LoggerTrait

You can use `Spiral\Logger\Traits\LoggerTrait` to assign Logger to any class quickly. The class name will be used as a
channel name:

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

And assign logger to it:

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
> LoggerTrait works only inside `$app->serve()` method via global IoC scope.

## Errors

To aggregate all application errors into a single log file subscribe to `default` channel:

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

## Debug

Use the second argument of `dump` function to dump directly into `default` log channel:

```php
dump('hello', \Spiral\Debug\Dumper::LOGGER); 
```
