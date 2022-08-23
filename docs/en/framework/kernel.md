# Framework - Kernel and Environment

Each spiral build will contain a kernel object with a set of application-specific services. Unlike Symfony,
The Spiral needs only one Kernel for all the dispatching methods (HTTP, Queue, GRPC, Console, etc). The kernel will
select the dispatching method automatically, based on connected `Spiral\Boot\DispatcherInterface` instances.

> **Note**
> The base kernel implementation located in `spiral/boot` repository.

## Kernel Responsibilities

The `Spiral\Boot\AbstractKernel` class only responsible for the following aspects of your application:

- container initialization via a set of application-specific bootloaders
- environment and directory structure initialization
- selection of appropriate dispatching method

To create your kernel extends `Spiral\Boot\AbstractKernel`.

```php
namespace App;

use Spiral\Boot\AbstractKernel;
use Spiral\Boot\Exception\BootException;

class MyApp extends AbstractKernel
{
    protected const LOAD = [
        // bootloaders to initialize
    ];

    public function get(string $service): mixed
    {
        return $this->container->get($service);
    }

    protected function bootstrap(): void
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

        return \array_merge(
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

> **Note**
> The `Spiral\Framework\Kernel` defines the default directory map.

## Kernel initialization

To init the kernel invoke static method `create`:

```php
$myapp = MyApp::create(
    [
        'root' => __DIR__
    ],
    false // do not mount error handler
);

$myapp->run(null); // use default env 

\dump($myapp->get(\Spiral\Boot\DirectoriesInterface::class)->getAll());
```

### Initialization callbacks

The `Spiral\Boot\AbstractKernel` class provides `running`, `booting`, `booted` callbacks that are executed at 
various times during application initialization.

#### Running

`Running` is the first callbacks to be executed. They are executed when the `run` method is called, immediately after 
binding the `EnvironmentInterface` in the application container.

To register a callback, call the `running` method:

```php
$app = App::create(
    directories: ['root' => __DIR__]
);

$app->running(function () {
    // ...
});

$app->run();
```

#### Booting

`Booting` is the callbacks that's will be executed *before* all framework bootloaders in the `LOAD` section will be booted.

To register a callback, call the `booting` method:

```php
$app = App::create(
    directories: ['root' => __DIR__]
);

$app->booting(function () {
    // ...
});

$app->run();
```

#### Booted

`Booted` is the callbacks that's will be executed *after* all framework bootloaders in the `LOAD` section will be booted.

To register a callback, call the `booted` method:

```php
$app = App::create(
    directories: ['root' => __DIR__]
);

$app->booted(function () {
    // ...
});

$app->run();
```

The `Spiral\Framework\Kernel` class provides additional `appBooting` and `appBooted` callbacks.
In an [application bundle](https://github.com/spiral/app), the default `App\App` class inherits from this `Kernel` 
class and has access to this functionality.

#### AppBooting

`AppBooting` is the callbacks that's will be executed *before* all application bootloaders in the `APP` section 
will be booted.

To register a callback, call the `appBooting` method:

```php
$app = App::create(
    directories: ['root' => __DIR__]
);

$app->appBooting(function () {
    // ...
});

$app->run();
```

#### AppBooted

`AppBooted` is the callbacks that's will be executed *after* all application bootloaders in the `APP` section
will be booted.

To register a callback, call the `appBooted` method:

```php
$app = App::create(
    directories: ['root' => __DIR__]
);

$app->appBooted(function () {
    // ...
});

$app->run();
```

## Environment

Use `Spiral\Boot\EnvironmentInterface` to access the list of ENV variables. By default, the framework relies on
system-level environment values. To redefine env values while initializing the kernel pass custom `EnvironmentInterface`
to the `create` method.

```php
use \Spiral\Boot\Environment;

$myapp = MyApp::create(
    [
        'root' => __DIR__
    ],
    false // do not mount error handler
);

$myapp->run(new Environment(['key' => 'value']));

\dump($myapp->get(\Spiral\Boot\EnvironmentInterface::class)->getAll());
```

> **Note**
> Such an approach can be used to bootstrap the application for testing purposes.
