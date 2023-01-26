# Queue — Interceptors

Spiral Framework provides a way for developers to customize the behavior of their job processing pipeline through the
use of interceptors. An interceptor is a piece of code that is executed before or after a job is pushed or consumed, and
which allows developers to hook into the job processing pipeline to perform some action.

> **See more**
> Read more about interceptors in the [Framework — Interceptors](../framework/interceptors.md) section.

Interceptors can be useful for a variety of purposes, such as handling errors, adding additional context to the job, or
performing some other action based on the job being processed.

To use interceptors you will need to implement the `Spiral\Core\CoreInterceptorInterface` interface. This interface
requires you to implement method `process`.

## Interceptor for Push

Here is an example of a simple interceptor that logs a message before and after a job is pushed:

```php
namespace App\Application\Job\Interceptor;

use Spiral\Core\CoreInterceptorInterface;
use Spiral\Core\CoreInterface;
use Spiral\Queue\OptionsInterface;

final class LogInterceptor implements CoreInterceptorInterface
{
    public function __construct(
        private readonly \Psr\Log\LoggerInterface $core,
    ) {
    }
    
    /**
     * @param array{options: ?OptionsInterface, payload: array} $parameters
     */
    public function process(string $name, string $action, array $parameters, CoreInterface $core): string
    {
        $this->logger->info('Job pushing...', [
            'name' => $name,
            'action' => $action,
        ]);
        
        $id = $core->callAction($name, $action, $parameters);
        
        $this->logger->info('Job pushed', [
            'id' => $id,
            'name' => $name,
            'action' => $action,
        ]);

        return $id;
    }
}
```

To use this interceptor, you will need to register it in the configuration file `app/config/queue.php`.

```php app/config/queue.php
use App\Application\Job\Interceptor\LogInterceptor;

return [    
    'interceptors' => [
        'push' => [
            LogInterceptor::class,
        ],
        'consume' => [
            //...
        ],
    ],
    // ...
];
```

> **Note**
> The `callAction` method will push and return the `id` string.

## Interceptor for Consume

To create an interceptor, you need to create a class and implement the interface `Spiral\Core\CoreInterceptorInterface`.
The `callAction` method will execute a consuming job, it doesn't return anything.

Let's create an `JobExceptionsHandlerInterceptor`. It's a class that provides 3 attempts to execute a job in a consumer 
before cancelling it.

```php
namespace App\Application\Job\Interceptor;

use Spiral\Core\CoreInterceptorInterface;
use Spiral\Core\CoreInterface;
use Spiral\Exceptions\ExceptionReporterInterface;

final class JobExceptionsHandlerInterceptor implements CoreInterceptorInterface
{
    public function __construct(
        private readonly ExceptionReporterInterface $reporter,
        private readonly int $maxAttempts = 3,
    ) {
    }
    /**
     * @param array{
     *     driver: non-empty-string, 
     *     queue: non-empty-string, 
     *     id: non-empty-string, 
     *     payload: array, 
     *     headers: array
     * } $parameters
     */
    public function process(string $name, string $action, array $parameters, CoreInterface $core): mixed
    {
        try {
            return $core->callAction($name, $action, $parameters);
        } catch (\Throwable $e) {
             $attempts = (int)($parameters['headers']['attempts'] ?? 0);
             $this->reporter->report($e);
             
            if ($attempts > $this->maxAttempts) {
                throw new \Spiral\Queue\Exception\FailException(reason: $e->getMessage());
            }
            
            throw new \Spiral\Queue\Exception\RetryException(
                reason: $e->getMessage(),
                options: (new Options())->withDelay(5)->withHeader('attempts', (string)($attempts - 1))
            );
        }
    }
}
```

To use this interceptor, you will need to register it in the configuration file `app/config/queue.php`.

```php app/config/queue.php
use App\Application\Job\Interceptor\JobExceptionsHandlerInterceptor;
use Spiral\Core\Container\Autowire;

return [    
    'interceptors' => [
        'consume' => [
            new Autowire(
                JobExceptionsHandlerInterceptor::class, [
                'maxAttempts' => 5
            ]),
        ],
        //...
    ],
    // ...
];
```

## Registering a new Interceptor

There are several ways to add a new interceptor.

### Via configuration

Add it to the configuration file `app/config/queue.php`.

```php app/config/queue.php
use App\CustomInterceptor;
use Spiral\Core\Container\Autowire;

return [    
    /**
     * -------------------------------------------------------------------------
     *  List of all interceptors
     * -------------------------------------------------------------------------
     */
     'interceptors' => [
        // interceptors for push
        'push' => [
            // via fully qualified class name
            CustomInterceptor::class,
        
            // via Autowire
            new Autowire(CustomInterceptor::class),     
            
            // via interceptor instance
            new CustomInterceptor(),
        ],
        
        // interceptors for consume
        'consume' => [
            // via fully qualified class name
            CustomInterceptor::class,
        
            // via Autowire
            new Autowire(CustomInterceptor::class),     
            
            // via interceptor instance
            new CustomInterceptor(),
        ],
     ],
];
```

#### Via QueueBootloader

Call the method `addPushInterceptor` to add an interceptor for the `push` action. Or call the
method `addConsumeInterceptor` to add an interceptor for the `consume` action in 
the `Spiral\Queue\Bootloader\QueueBootloader` class.

```php app/src/Application/Bootloader/AppBootloader.php
namespace App\Application\Bootloader;

use App\Interceptor\CustomInterceptor;
use Spiral\Boot\Bootloader\Bootloader;
use Spiral\Core\Container\Autowire;
use Spiral\Queue\Bootloader\QueueBootloader;

class AppBootloader extends Bootloader
{
    public function boot(QueueBootloader $queue): void
    {
        // via fully qualified class name
        $queue->addPushInterceptor(CustomInterceptor::class);
        $queue->addConsumeInterceptor(CustomInterceptor::class);

        // via Autowire
        $queue->addPushInterceptor(new Autowire(CustomInterceptor::class));
        $queue->addConsumeInterceptor(new Autowire(CustomInterceptor::class));
        
        // via interceptor instance
        $queue->addPushInterceptor(new CustomInterceptor());
        $queue->addConsumeInterceptor(new CustomInterceptor());
    }
}
```
