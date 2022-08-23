# Framework - Bootloaders

Bootloaders are the central piece in Spiral Framework and your application. These objects are responsible
for [Container](../framework/container.md) configuration, default configuration, etc.

Bootloaders only executed once while loading your application. Since the app will stay in memory for long - you can
add as many code to your bootloaders as you want. It will not cause any performance effect on runtime.

![Application Control Phases](https://user-images.githubusercontent.com/773481/180768689-c711e6f0-3523-4330-a496-f78088504b29.png)

## Simple Bootloader

You can create simple Bootloader by extending `Spiral\Boot\Bootloader\Bootloader` class:

```php
namespace App\Bootloader;

use Spiral\Boot\Bootloader\Bootloader;

class MyBootloader extends Bootloader 
{
}
```

Every Bootloader must be activated in your application kernel. Add the class reference into `LOAD` or `APP` lists of
your `App\App` class:

```php
namespace App;

use Spiral\Framework\Kernel;
use App\Bootloader\RoutesBootloader;
use App\Bootloader\LoggingBootloader;
use App\Bootloader\MyBootloader;

class App extends Kernel
{
    protected const LOAD = [
        // ...
    ];

    protected const APP = [
        RoutesBootloader::class,
        LoggingBootloader::class,
        MyBootloader::class
    ];
}
```

Currently, your Bootloader doesn't do anything. A little bit later, we will add functionality to it.

## Available methods

Bootloaders provide two methods `init` and `boot` that are executed when the application is initialized.

### Init

This method is executed *first*. It's recommended to set default values for configuration files. Modify configuration 
files using special bootloader methods. Execute other logic that doesn't require reading configuration files 
and doesn't depend on the execution of code in the `init` and `boot` methods of other bootloaders.
In this method, you can add initialization callbacks, configure container bindings if this does not require 
access to the application configuration.

```php
use Spiral\Boot\AbstractKernel;
use Spiral\Boot\Bootloader\Bootloader;
use Spiral\Boot\EnvironmentInterface;
use Spiral\Config\ConfiguratorInterface;
use Spiral\Queue\Config\QueueConfig;

final class QueueBootloader extends Bootloader
{
    public function __construct(
        private readonly ConfiguratorInterface $config
    ) {
    }

    public function init(
        EnvironmentInterface $env, 
        AbstractKernel $kernel
    ): void {
        $this->config->setDefaults(
            QueueConfig::CONFIG,
            [
                'default' => $env->get('QUEUE_CONNECTION', 'sync'),
                // ...
            ]
        );
        
        $kernel->booting(function () {
            // ...
        });
    }
}
```

### Boot

This method is executed *after* executing method `init` in all bootloaders. It can be used if you need the result 
of the `init` methods in all bootloaders. For example, compiled configuration files.

```php
use Spiral\Boot\Bootloader\Bootloader;
use Spiral\Config\ConfiguratorInterface;
use Spiral\Session\Config\SessionConfig;

final class SessionBootloader extends Bootloader
{
    public function boot(
        ConfiguratorInterface $config,
        CookiesBootloader $cookies
    ): void {
        $session = $config->getConfig(SessionConfig::CONFIG);

        $cookies->whitelistCookie($session['cookie']);
    }
}
```

> **Note**
> `APP` bootloader namespace always loaded after `LOAD`, keep domain-specific bootloaders in it.

## Configuring Container

The most common use-case of bootloaders is to configure DI container, for example, we might want to bind multiple
implementations to their interfaces or construct some service.

We can use the method `init` or `boot` for these purposes. Method support method injection, so we can request any services we
need:

```php
namespace App\Bootloader;

use Spiral\Core\Container;
use App\MyClassInterface;
use App\MyClass;
use App\MyService;

class MyBootloader extends Bootloader 
{
    public function boot(Container $container): void
    {
        $container->bind(MyClassInterface::class, MyClass::class);
        
        $container->bindSingleton(MyService::class, function(MyClass $myClass) {
            return new MyService($myClass); 
        });
    }
}
```

Bootloaders provide the ability to simplify container binding definition using constants `BINDINGS` and `SINGLETONS`.

```php
namespace App\Bootloader;

use Spiral\Core\Container;
use App\MyClassInterface;
use App\MyClass;
use App\MyService;

class MyBootloader extends Bootloader 
{
    const BINDINGS = [
        MyInterface::class => MyClass::class
    ];

    public function boot(Container $container): void
    {
        $container->bindSingleton(MyService::class, function(MyClass $myClass) {
            return new MyService($myClass); 
        });
    }
}
```

You can also replace factory closures with factory methods:

```php
namespace App\Bootloader;

use App\MyClassInterface;
use App\MyClass;
use App\MyService;

class MyBootloader extends Bootloader 
{
    const BINDINGS = [
        MyInterface::class => MyClass::class
    ];

    const SINGLETONS = [
        MyService::class => [self::class, 'myService']
    ];

    protected function myService(MyClass $myClass): MyService
    {
        return new MyService($myClass); 
    }
}
```

## Configuring Application

Another common use case of bootloaders is to configure the framework before the application launch. For example, we can
declare new route for our application or module:

```php
namespace App\Bootloader;

use Spiral\Router\RouterInterface;
use Spiral\Router\Target\Controller;
use Spiral\Router\Route;

class MyBootloader extends Bootloader 
{
    public function boot(RouterInterface $router): void
    {
        $router->setRoute(
            'my-route',
            new Route('/<action>', new Controller(MyController::class))
        );
    }
}
```

> **Note**
> Identically, you can mount middleware, change tokenizer directories, and much more.

## Depending on other Bootloaders

Some framework bootloaders can be used as a simple path to configure application settings. For example, we can
use `Spiral\Bootloader\Http\HttpBootloader` to add global PSR-15 middleware:

```php
namespace App\Bootloader;

use Spiral\Bootloader\Http\HttpBootloader;
use App\Middleware\MyMiddleware;

class MyBootloader extends Bootloader 
{
    public function boot(HttpBootloader $http): void
    {
        $http->addMiddleware(MyMiddleware::class);
    }
}
```

If you want to ensure that `HttpBootloader` has always been initiated before `MyBootloader` use
constant `DEPENDENCIES`:

```php
namespace App\Bootloader;

use Spiral\Bootloader\Http\HttpBootloader;
use App\Middleware\MyMiddleware;

class MyBootloader extends Bootloader 
{
    const DEPENDENCIES = [
        HttpBootloader::class
    ];

    public function boot(HttpBootloader $http): void
    {
        $http->addMiddleware(MyMiddleware::class);
    }
}
```

> **Note**
> Note, you are only able to use bootloaders to configure your components during the bootstrap phase (a.k.a. via another
> bootloader). The framework would not allow you to change any configuration value after component initialization.

## Cascade bootloading

You can control the bootload process using Bootloader itself, simply request `Spiral\Boot\BootloadManager`:

```php
namespace App\Bootloader;

use Spiral\Boot\Bootloader\Bootloader;
use Spiral\Boot\BootloadManager;
use Spiral\Bootloader\DebugBootloader;
use Spiral\Boot\EnvironmentInterface;

class AppBootloader extends Bootloader
{
    public function boot(BootloadManager $bootloadManager, EnvironmentInterface $env): void
    {
        if ($env->get('DEBUG')) {
            $bootloadManager->bootload([
                DebugBootloader::class
            ]);
        }
    }
}
```
