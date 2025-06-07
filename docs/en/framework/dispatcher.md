# Framework — Dispatchers

One of the key features of Spiral is its support for multiple kernel dispatchers, which are responsible for routing 
incoming requests to the appropriate handler based on the current environment.

Let's imagine that we have two dispatchers: `console` and `http`.

Here is an example of `http` dispatcher:

```php
use Spiral\Boot\EnvironmentInterface;
use Spiral\Boot\DispatcherInterface;

final class HttpDispatcher implements DispatcherInterface
{
    public function __construct(
        private readonly EnvironmentInterface $env
    ) {
    }
    
    public function canServe(): bool
    {
        return $this->env->get('RR_MODE') === 'http';
    }
    
    public function serve(): void
    {
        // Handle HTTP requests
    }
}
```

And an example of `console` dispatcher:

```php
use Spiral\Boot\EnvironmentInterface;
use Spiral\Boot\DispatcherInterface;

final class ConsoleDispatcher implements DispatcherInterface
{
    public function __construct(
        private readonly EnvironmentInterface $env
    ) {
    }
    
    public function canServe(): bool
    { 
        return (PHP_SAPI === 'cli' && $this->env->get('RR_MODE') === null);
    }
    
    public function serve(InputInterface $input = null, OutputInterface $output = null): int
    {
        // Handle console command
    }
}
```

Now we can register these dispatchers in our application:

```php
use Spiral\Boot\Bootloader\Bootloader;
use Spiral\Boot\KernelInterface;

class AppBootloader extends Bootloader
{
    public function boot(
      KernelInterface $kernel, 
      HttpDispatcher $http,
      ConsoleDispatcher $console,
    ): void  {
        $kernel->addDispatcher($http)
        $kernel->addDispatcher($console);
    }
}
```

And an entry point `app.php` of our application will look like this:

```php app.php
use App\Application\Kernel;

\mb_internal_encoding('UTF-8');
\error_reporting(E_ALL | E_STRICT ^ E_DEPRECATED);
\ini_set('display_errors', 'stderr');

require __DIR__ . '/vendor/autoload.php';
$app = Kernel::create(
    directories: ['root' => __DIR__],
)->run();

$code = (int)$app->serve();  // <========== Will run the appropriate dispatcher based on the current environment
exit($code);
```

When we run our application, the appropriate dispatcher will be chosen based on the current environment. For example, if
we run the following command:

```terminal
php app.php db:migrate
```

The framework will iterate through the list of registered dispatchers and call the `canServe` method on each
dispatcher. This method is a way for the dispatcher to tell the framework whether it can handle request based on the
current environment. The framework will use the first dispatcher that returns `true`.

The `ConsoleDispatcher` will return `true` if the current environment is `cli`. When RoadRunner starts the HTTP plugin,
the plugin will start a worker process and pass the `RR_MODE=http` environment variable to the worker.
The `HttpDispatcher` will be selected in this case.

If no dispatcher returns `true`, the framework will throw an exception.

## Available Dispatchers

Spiral comes with several built-in dispatchers:

- [Console dispatcher](https://github.com/spiral/framework/blob/master/src/Framework/Console/ConsoleDispatcher.php):
  responsible for handling console commands within your application. This is useful if you want to create custom
  commands that can be run from the command line.

- [RoadRunner HTTP dispatcher](https://github.com/spiral/roadrunner-bridge/blob/4.x/src/Http/Internal/Dispatcher.php):
  responsible for handling incoming HTTP requests and routing them to the appropriate controller action or function.
  This is the dispatcher that is used when your application is running as an HTTP service.

- [RoadRunner GRPC dispatcher](https://github.com/spiral/roadrunner-bridge/blob/4.x/src/GRPC/Internal/Dispatcher.php):
  responsible for handling incoming GRPC requests and routing them to the appropriate services.
  This is the dispatcher that is used when your application is running as a GRPC service.

- [RoadRunner TCP dispatcher](https://github.com/spiral/roadrunner-bridge/blob/4.x/src/Tcp/Internal/Dispatcher.php):
  Handles responsible for handling incoming TCP connections and routing them to the appropriate handler.
  This is the dispatcher that is used when your application is running as a TCP service.

- [Temporal dispatcher](https://github.com/spiral/temporal-bridge/blob/2.0/src/Dispatcher.php): responsible for handling
  incoming workflow activities. Temporal is a distributed, scalable, and fault-tolerant workflow engine that is used to
  build and orchestrate long-running business logic.

- [RoadRunner Queue dispatcher](https://github.com/spiral/roadrunner-bridge/blob/4.x/src/Queue/Internal/Dispatcher.php):
  allows you to consume messages from a queue and route them to the appropriate handler.
  This is useful if you want to build a system that is based on message-driven architecture.
  This is the dispatcher that is used when your application is running as a Queue consumer service.

> **Note**
> Read how to create custom dispatcher [here](../cookbook/custom-dispatcher.md)

Spiral kernel dispatchers provide a flexible and powerful way to route incoming requests to the appropriate handler. 
They are an essential component and are an important part of its overall design.
