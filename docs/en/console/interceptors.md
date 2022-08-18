# Console - Interceptors

Interceptors allow to intercept the execution of a console command and execute some logic before or after the command execution.

## Creating an Interceptor

To create an interceptor, you need to create a class and implement an interface `Spiral\Core\CoreInterceptorInterface`.

```php
namespace App;

use Spiral\Console\Command;
use Spiral\Core\CoreInterceptorInterface;
use Spiral\Core\CoreInterface;
use Symfony\Component\Console\Input\InputInterface;
use Symfony\Component\Console\Output\OutputInterface;

class Interceptor implements CoreInterceptorInterface
{

    /**
     * @param array{input: InputInterface, output: OutputInterface, command: Command}|array<empty, empty> $parameters
     */
    public function process(string $commandClass, string $method, array $parameters, CoreInterface $core): int
    {
        // ...

        $result = $core->callAction($commandClass, $method, $parameters);

        // ...

        return $result;
    }
}
```

## Adding an Interceptor

For an interceptor works, it must be registered in the application. There are several ways to add a new interceptor.

### Via configuration

Add it to the configuration file `app/config/console.php`.

```php
use App\Interceptor;
use Spiral\Core\Container\Autowire;

return [    
    /**
     * -------------------------------------------------------------------------
     *  List of all interceptors
     * -------------------------------------------------------------------------
     */
    'interceptors' => [
        // via fully qualified class name
        Interceptor::class,
        
        // via Autowire
        new Autowire(Interceptor::class),
    ],
];
```

### Via ConsoleBootloader

Call method `addInterceptor` in the `Spiral\Console\Bootloader\ConsoleBootloader` class.

```php
namespace App\Bootloader;

use App\Interceptor;
use Spiral\Boot\Bootloader\Bootloader;
use Spiral\Console\Bootloader\ConsoleBootloader;

class AppBootloader extends Bootloader
{
    public function boot(ConsoleBootloader $console): void
    {
        $console->addInterceptor(Interceptor::class);
    }
}
```
