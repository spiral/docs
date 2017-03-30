# Logging
Spiral supplied with simple PSR-3 compatible bridge to get access to a channel specific logger instance.

Default `LogsFactory` implementation is based on Monolog Logger and Monolog Handlers.

## Access global LoggerInterface
You can always write information into global instance of logger by requesting it as dependency:

```php
public function indexAction(LoggerInterface $logs)
{
    $logs->alert('alert');
}
```

## Class specific Loggers
If you wish to use custom logger for your classes you can either create it manually using `LogsInterface` as factory:

```php
public function indexAction(LogsInterface $logs)
{
    $logs->getLogger('channel')->alert('alert');
}
```

Or use `Spiral\Debug\Traits\LoggerTrait` to automatically receive class specific logger:

```php
use LoggerTrait;

public function indexAction()
{
    $this->getLogger()->alert('alert');
}
```

> Note, LoggerTrait will only work inside application IoC scope.

## Configure Loggers
You can configure any of your log channel to write it's content using Monolog compatible handlers, do that using "monolog" configuration file:

```php
return [
    /*
     * This is default spiral logger, all error messages are passed into it.
     */
    \Spiral\Debug\LogManager::DEBUG_CHANNEL => [
        [
            'handler' => \Monolog\Handler\RotatingFileHandler::class,
            'format'  => "[%datetime%] %level_name%: %message%\n",
            'options' => [
                'level'          => \Psr\Log\LogLevel::ERROR,
                'maxFiles'       => 1,
                'filename'       => directory('runtime') . 'logs/errors.log',
                'bubble'         => false
            ],
        ],
        /*{{handlers.debug}}*/
    ],

    /*
     * Such middleware provides ability to isolate ClientExceptions into nice error pages. You can
     * use it's log to collect http errors.
     */
    HttpErrors::class                       => [
        [
            'handler' => \Monolog\Handler\RotatingFileHandler::class,
            'format'  => "[%datetime%] %message%\n",
            'options' => [
                'level'    => \Psr\Log\LogLevel::ERROR,
                'maxFiles' => 7,
                'filename' => directory('runtime') . 'logs/http.log'
            ],
        ],
        /*{{handlers.http}}*/
    ],

    /*{{handlers}}*/
];
```

For example, we can specify that all error messages created by IndexController must be stored in a log file:

```php
class IndexController extends Controller
{
    use LoggerTrait;

    public function indexAction()
    {
        $this->getLogger()->alert('alert');
    }
}
```

You can use your class name as channel in "monolog" configuration:

```php
\Controllers\IndexController::class     => [
    [
        'handler' => \Monolog\Handler\RotatingFileHandler::class,
        'format'  => "[%datetime%] %level_name%: %message%\n",
        'options' => [
            'level'    => \Psr\Log\LogLevel::ERROR,
            'filename' => directory('runtime') . 'logs/index-controller.log',
            'maxFiles' => 1,
        ],
    ],
],
```

> Feel free to create your own implementation of `LogsInterface`.