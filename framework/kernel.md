# Framework - Kernel and Environment
Each spiral build will contain domain specific kernel with the set of application specific services. Unlike Symfony, 
Spiral need only one Kernel for all the dispatching methods (HTTP, Queue, GRPC, Console). Kernel will select dispatching
method automatically, based on connected `Spiral\Boot\DispatcherInterface` instances.

The base kernel implementation is located in `spiral/boot` repository.

## Kernel Responsibilities
The `Spiral\Boot\Kernel` class only responsible for the following aspects of your application:
- container initialization via set of application specific bootloaders
- environment and directory structure initialization
- selection of appropriate dispatching method

To create your own kernel extend `Spiral\Boot\AbstractKernel`. 

```php
namespace App;

use Spiral\Boot\AbstractKernel;
use Spiral\Boot\Exception\BootException;

class MyApp extends AbstractKernel
{
    protected const LOAD = [
        // bootloaders to initialize
    ];

    public function get(string $service)
    {
        return $this->container->get($service);
    }

    protected function bootstrap()
    {
        // custom initialization code
        // invoked after all bootloaders are loaded
    }

    protected function mapDirectories(array $directories): array
    {
        if (!isset($directories['root'])) {
            throw new BootException('Missing required directory `root`');
        }

        if (!isset($directories['app'])) {
            $directories['app'] = $directories['root'] . '/app/';
        }

        return array_merge(
            [
                // public root
                'public'    => $directories['root'] . '/public/',

                // vendor libraries
                'vendor'    => $directories['root'] . '/vendor/',

                // data directories
                'runtime'   => $directories['root'] . '/runtime/',
                'cache'     => $directories['root'] . '/runtime/cache/',

                // application directories
                'config'    => $directories['app'] . '/config/',
                'resources' => $directories['app'] . '/resources/',
            ],
            $directories
        );
    }
}
```

> The `Spiral\Framework\Kernel` defines the default directory map.

To init the kernel invoke static method `init`:

```php
$myapp = MyApp::init(
    [
        'root' => __DIR__
    ],
    null, // use default env 
    false // do not mount error handler
);      

dump($myapp->get(\Spiral\Boot\DirectoriesInterface::class)->getAll());
```

## Environment
Use `Spiral\Boot\EnviromentInterface` to access list of ENV variables. By default framework relies on system level 
environment values. To redefine env values while initializing the kernel pass custom `EnviromentInterface` to `init` method.

```php
$myapp = MyApp::init(
    [
        'root' => __DIR__
    ],
    new Environment(['key' => 'value']),
    false // do not mount error handler
);

dump($myapp->get(\Spiral\Boot\EnvironmentInterface::class)->getAll());
```

> Such approach can be used to bootstrap application for testing purposes.