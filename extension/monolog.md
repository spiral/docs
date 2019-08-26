# Extensions - Monolog
Web and GRPC bundles include default integration with https://github.com/Seldaek/monolog to manage logging.

## Configuration
Extension does not require any default configuration. Use `Spiral\Monolog\Bootloader\MonologBootloader` to
declare handler and formatter for specific channel:

```php
public function boot(MonologBootloader $monolog)
{
    $monolog->addHandler(
        'my-channel',
        $monolog->logRotate(directory('runtime') . 'logs/my-channel.log')
    );
}
``` 

> You can use any monolog handler as second argument.

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

## LoggerTrait
You can use `Spiral\Logger\Traits\LoggerTrait` to quickly assign Logger to any class, class name will be used as 
channel name:

```php
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
        $monolog->logRotate(directory('runtime') . 'logs/home-controller.log')
    );
}
```

> LoggerTrait works only inside `$app->serve()` method.