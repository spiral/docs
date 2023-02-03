# Framework — Kernel and Environment

Spiral utilizes a kernel object that contains a set of application-specific services. Unlike Symfony, Spiral only 
requires one kernel for all dispatching methods, such as HTTP, Queue, GRPC Console, etc. The kernel automatically 
selects the appropriate dispatching method based on the connected [Dispatcher](../framework/dispatcher.md).

> **Note**
> The base kernel implementation is located in `spiral/boot` repository.

## Kernel Responsibilities

The Spiral\Boot\AbstractKernel class is responsible for the following aspects of the application:

- Initializing the container through a set of application-specific bootloaders
- Initializing the bootloaders
- Initializing the environment and directory structure
- Initializing the exception handler (if required)
- Selecting the appropriate dispatcher

To create an application kernel, one must extend the `Spiral\Boot\AbstractKernel` class. An example of this can be seen
in the following code snippet:

```php app/src/Application/MyApp.php
namespace App\Application;

use Spiral\Boot\AbstractKernel;
use Spiral\Boot\Exception\BootException;

final class MyApp extends AbstractKernel
{
    protected const LOAD = [
        // bootloaders to initialize
    ];

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

To initialize the kernel, the static method `create` should be invoked. An example of this can be seen in the following
code snippet:

```php app.php
$myapp = MyApp::create(
    directories: [
        'root' => __DIR__,
    ],
    handleErrors: false // do not mount error handler
);

$myapp->run(environment: null); // use default env 

\dump($myapp->get(\Spiral\Boot\DirectoriesInterface::class)->getAll());
```

> **Note**
> During initialization, `MyApp` will be bound to `Spiral\Boot\KernelInterface` in the container as a singleton.

### Callbacks

The `Spiral\Boot\AbstractKernel` class provides several callbacks that are executed at different stages of application
initialization. These callbacks are `running`, `booting`, `booted`, and `bootstrapped`. The `Spiral\Framework\Kernel`
class, which extends `AbstractKernel`, adds additional callbacks, `appBooting` and `appBooted`. This allows developers
to perform custom actions at specific stages of the application initialization process.

> **Note**
> In the application bundle, the default `App\Application\Kernel` class extends the `Spiral\Framework\Kernel` class and
> makes use of these callbacks.

#### Running

The `running` callback is the first callback to be executed during the application initialization process. It is
executed when the `run` method is called, immediately after binding the `EnvironmentInterface` in the application
container.

Here is an example of the `running` callback:

```php app.php
$app = MyApp::create(directories: ['root' => __DIR__]);

$app->running(static function (): void {
    // Do something
});

$app->run();
```

> **Note**
> Callbacks can be called multiple time to register multiple callbacks, they will be invoked in the order they have
> been registered.

#### Booting

The `booting` callback is executed before all the framework bootloaders in the `LOAD` section are booted.

There are two ways to register a callback for the `booting` stage:

:::: tabs

::: tab Kernel initialization

The `booting` method can be called on the application instance after it has been created.

```php app.php
$app = MyApp::create(
    directories: ['root' => __DIR__]
);

$app->booting(function () {
    // ...
});

$app->run();
```

:::

::: tab Kernel bootloaders

The `init` method of a bootloader can also be used to register a booting callback. This method is executed before the
callback fired.

```php app/src/Application/Bootloader/AppBootloader.php
namespace App\Application\Bootloader;

use Spiral\Boot\Bootloader\Bootloader;
use Spiral\Boot\KernelInterface;

class AppBootloader extends Bootloader
{
    public function init(KernelInterface $app): void
    {
        $app->booting(function () {
            // ...
        });
    }
}
```

:::

::::

#### Booted

The `booted` callback is executed after all the framework bootloaders in the `LOAD` section have completed their
initialization process.

```php
$app->booted(function () {
    // ...
});
```

#### AppBooting

The `appBooting` callback is executed before all the application bootloaders in the `APP` section are booted.

```php
$app->appBooting(function () {
    // ...
});
```

#### AppBooted

The `appBooted` callback is executed after all the application bootloaders in the `APP` section have completed their
initialization process.

```php
$app->appBooted(function () {
    // ...
});
```

## Environment

Spiral integrates with [Dotenv](https://github.com/vlucas/phpdotenv) through the
`Spiral\DotEnv\Bootloader\DotenvBootloader` class. This bootloader is responsible for loading the environment variables
from the `.env` file and making them available to the application.

### Environment variables

The `Spiral\Boot\EnvironmentInterface` is used to access a list of environment variables (ENV vars). By default, the
framework relies on system-level environment values. However, it is possible to redefine these values while initializing
the kernel by passing a custom `Spiral\Boot\Environment` object to the `run` method.

> **See more**
> Read more about application environments in the [Getting started — Configuration](../start/configuration.md) section.

An example of this can be seen in the following code snippet:

```php app.php
use \Spiral\Boot\Environment;

// Create an application instance ...

$app->run(new Environment(['DEBUG' => true]));

\dump($app->get(\Spiral\Boot\EnvironmentInterface::class)->getAll());
```

> **Note**
> This approach can be used to bootstrap the application for testing purposes.

### .env file location

By default, the bootloader looks for the `.env` file in the root of the project, but you can change its location by
defining the `DOTENV_PATH` environment variable when running the Kernel:

```php app.php
use Spiral\Boot\Environment;

$app = App\Application\Kernel::create(...);

$app->run(new Environment(['DOTENV_PATH' => __DIR__ . '/.env.production']));
```

> **Note**
> In addition, you can also create your own implementation of the `DotenvBootloader` class. This allows you to customize
> the behavior of loading environment variables, such as changing the location where the .env file is searched for, or
> adding additional functionality. This can be useful in cases where the default bootloader does not meet the specific
> requirements of your application.

### Overwriting variables

By default, Spiral does not overwrite previously set environment variables when loading new ones from the `.env` file. 
However, this behavior can be changed by setting the `overwrite` parameter to `true` when initializing the
`Environment` class.

```php app.php
use Spiral\Boot\Environment;

$app = App\Application\Kernel::create(...);

$app->run(new Environment([
    'APP_ENV' => 'production'
], overwrite: true));
```

## Events

| Event                                | Description                                                                                                        |
|--------------------------------------|--------------------------------------------------------------------------------------------------------------------|
| Spiral\Boot\Event\Bootstrapped       | The Event will be fired `after` all bootloaders from `SYSTEM`, `LOAD` and `APP` sections initialized.              |
| Spiral\Boot\Event\Serving            | The Event will be fired `before` looking for a dispatcher for handling incoming requests in a current environment. |
| Spiral\Boot\Event\DispatcherFound    | The Event will be fired when a dispatcher for handling incoming requests in a current environment is found.        |
| Spiral\Boot\Event\DispatcherNotFound | The Event will be fired when an application dispatcher is not found.                                               |
| Spiral\Boot\Event\Finalizing         | The Event will be fired when finalizer are executed `before` running finalizers.                                   |

> **Note**
> To learn more about dispatching events, see the [Events](../advanced/events.md) section in our documentation.
 
