# Queue and Jobs - Interceptors

Interceptors allow to intercept the `push` or `consume` job and execute some logic before or after the push or
consume execution.

## Creating an Interceptor for Push

To create an interceptor, you need to create a class and implement an interface `Spiral\Core\CoreInterceptorInterface`.
Calling the `callAction` method will push and return the `id` string.

```php
namespace App\Interceptor;

use Spiral\Core\CoreInterceptorInterface;
use Spiral\Core\CoreInterface;
use Spiral\Queue\OptionsInterface;

class CustomInterceptor implements CoreInterceptorInterface
{
    /**
     * @param array{options: ?OptionsInterface, payload: array} $parameters
     */
    public function process(string $name, string $action, array $parameters, CoreInterface $core): string
    {
        // ...

        $id = $core->callAction($name, $action, $parameters);

        // ...
        
        return $id;
    }
}
```

## Creating an Interceptor for Consume

To create an interceptor, you need to create a class and implement an interface `Spiral\Core\CoreInterceptorInterface`.
The `callAction` method will execute a consuming job, it doesn't return anything.

```php
namespace App\Interceptor;

use Spiral\Core\CoreInterceptorInterface;
use Spiral\Core\CoreInterface;

class CustomInterceptor implements CoreInterceptorInterface
{
    /**
     * @param array{
     *     driver: non-empty-string, 
     *     queue: non-empty-string, 
     *     id: non-empty-string, payload: array
     * } $parameters
     */
    public function process(string $name, string $action, array $parameters, CoreInterface $core): mixed
    {
        // ...
        
        $core->callAction($name, $action, $parameters);

        // ...

        return null;
    }
}
```

## Registering a new Interceptor

For an interceptor works, it must be registered in the application. There are several ways to add a new interceptor.

### Via configuration

Add it to the configuration file `app/config/queue.php`.

```php
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

Call method `addPushInterceptor` for adding an interceptor for `push` action. Or call method `addConsumeInterceptor` 
for adding an interceptor for `consume` action in the `Spiral\Queue\Bootloader\QueueBootloader` class.

```php
namespace App\Bootloader;

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
