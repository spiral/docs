# Extensions - Monolog
Web and GRPC bundles include default integration with https://github.com/Seldaek/monolog to manage logs.

## Configuration
Extension does not require any default configuration. Use `Spiral\Monolog\Bootloader\MonologBootloader` to
declare handler and formatter for specific channel:

```php
namespace App\Bootloader;

use Spiral\Boot\Bootloader\Bootloader;
use Spiral\Monolog\Bootloader\MonologBootloader;

class LoggingBootloader extends Bootloader
{
    protected const DEPENDENCIES = [
        MonologBootloader::class
    ];
    
    public function boot(MonologBootloader $monolog)
    {
        $monolog->addHandler(
            'my-channel',
            $monolog->logRotate(directory('runtime') . 'logs/my-channel.log')
        );
    }
}
``` 

> You can use any monolog handler as the second argument.

## Write to Log
You receive `default` logger using `Psr\Log\LoggerInterface` dependency:

```php
public function index(LoggerInterface $logger)
{
    $logger->alert('message');
}
```

To get logger for specific channel use `Spiral\Logger\LogsInterface`:

```php
use Spiral\Logger\LogsInterface;

// ...
public function index(LogsInterface $logs)
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

    /**
     * @param MonologBootloader $monolog
     */
    public function boot(MonologBootloader $monolog)
    {
        $monolog->addHandler(
            'default',
            $monolog->logRotate(directory('runtime') . 'logs/debug.log')
        );
    }
}
```

## LoggerTrait
You can use `Spiral\Logger\Traits\LoggerTrait` to quickly assign Logger to any class, the class name will be used as 
channel name:

```php
use Spiral\Logger\Traits\LoggerTrait;

class HomeController
{
    use LoggerTrait;

    public function index()
    {
        $this->getLogger()->alert('message');
    }
}
```

And assign logger to it:

```php
public function boot(MonologBootloader $monolog)
{
    $monolog->addHandler(
        HomeController::class,
        $monolog->logRotate(directory('runtime') . 'logs/home-controller.log') // handler
    );
}
```

> LoggerTrait works only inside `$app->serve()` method via global IoC scope.

## Errors
To aggregate all application errors into single log file subscribe to `default` channel:

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

    /**
     * @param MonologBootloader $monolog
     */
    public function boot(MonologBootloader $monolog)
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
