# Framework — Bootloaders

Bootloaders are the central piece in Spiral Framework and your application. These objects are responsible
for [Container](../framework/container.md) configuration, default configuration, etc.

Bootloaders are only executed once while loading your application. Since the app will stay in memory for long - you can
add as much code to your bootloaders as you want. It will not cause any performance effect on runtime.

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

Every Bootloader must be activated in your application kernel. Add the class reference into `defineBootloaders` or `defineAppBootloaders` functions of
your `App\App` class:

```php
namespace App;

use Spiral\Framework\Kernel;
use App\Bootloader\RoutesBootloader;
use App\Bootloader\LoggingBootloader;
use App\Bootloader\MyBootloader;

class App extends Kernel
{
    public function defineBootloaders(): array
    {
        return [
            // ...
           RoutesBootloader::class,
        ]
    }

    public function defineAppBootloaders(): array
    {
        return [
           LoggingBootloader::class,
           MyBootloader::class,
           
           // anonymous bootloader via object instance
           new class () extends Bootloader {
               public const BINDINGS = [
                 // ...
               ];
               public const SINGLETONS = [
                 // ...
               ];
               
               public function init(BinderInterface $binder): void
               {
                 // ...
               }
               
               public function boot(BinderInterface $binder): void
               {
                  // ...
               }
           },
       ];
    }
}
```

> **Note**
> `defineAppBootloaders` is always loaded after `defineBootloaders`, keep domain-specific bootloaders in it.

Currently, your Bootloader doesn't do anything. A little bit later, we will add some functionality to it.

## Available methods

Bootloaders provide two methods `init` and `boot` that are executed when the application is initialized.

### Init

This method will be run **first**. It's a good idea to set default values for the config files before proceeding. You 
can use the special bootloader methods to modify the config files as needed. After that, you can go ahead and run any 
other logic that doesn't require reading the config files and isn't dependent on code execution in the `init` and `boot` 
methods of other bootloaders. This is also a good time to add initialization callbacks and configure container bindings, 
as long as it doesn't require accessing the app config.

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

> **Note**
> 1. Learn more about the `Spiral\Boot\EnvironmentInterface`, in the 
> [Configuration](../start/configuration.md) section.
> 2. Learn more about the `Spiral\Boot\AbstractKernel` class (also known as the 'Kernel'), in
> the [Kernel and Environment](../framework/kernel.md) section.
> 3. Learn more about the `Spiral\Config\ConfiguratorInterface` class, in
> the [Config Objects](../framework/config.md) section.

### Boot

This method will be run after the `init` method in the bootloaders have been executed. The reason for this is that
you might need the results of bootloader initialization in order to proceed. For example, compiled configuration files.

Just keep in mind that it should be run after all those init methods have completed.

```php
use Spiral\Boot\Bootloader\Bootloader;
use Spiral\Session\Config\SessionConfig;

final class SessionBootloader extends Bootloader
{
    public function boot(
        SessionConfig $session,
        CookiesBootloader $cookies
    ): void {
        $cookies->whitelistCookie($session->getCookie());
    }
}
```
## Configuring Container

Bootloaders are usually used to set up a DI container, like if we want to link multiple implementations to their 
interfaces or create some service. We can use the `init` or `boot` method for this, which lets us request any services we 
need using method injection.

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
        
        $container->bindSingleton(MyService::class, static function(MyClass $myClass) {
            return new MyService($myClass); 
        });
    }
}
```

> **Note**
> The closure is provided as an argument to the `bindSingleton` method will be called by the dependency injection 
> (DI) container when it needs to create an instance of `MyService`. When the closure is called, the DI container will 
> automatically resolve and inject any dependencies that are required by the closure. 
> 
> If you want to learn more about DI, you can check out the [Container and Factories](../framework/container.md) section
> of the documentation. It should have all the info you need.

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
        MyClassInterface::class => MyClass::class
    ];

    public function boot(Container $container): void
    {
        $container->bindSingleton(MyService::class, static function(MyClass $myClass) {
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
declare a new route for our application or module:

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
> You are only able to use bootloaders to configure your components during the bootstrap phase (a.k.a. via another
> bootloader). The framework would not allow you to change any configuration value after component initialization.

## Depending on other Bootloaders

Depending on other bootloaders can be really useful in certain situations. For example, if you want to make sure a 
certain bootloader is initialized before yours, you can use one of two main approaches: injecting the bootloader class 
into the init or boot method as an argument, or using the `Bootloader::DEPENDENCIES` constant in your bootloader class. 

This can be a good way to manage the initialization of your app and make sure all the necessary resources and 
dependencies are available when you need them. Just keep in mind that dependent bootloaders will only be initialized 
once, even if they are depended on by multiple other bootloaders.

Some framework bootloaders can be used as a simple way to configure application settings. For example, we can
use `Spiral\Bootloader\Http\HttpBootloader` to add global PSR-15 middleware:

**There are two ways to define dependent bootloaders:**

1. Injecting the bootloader class into the `init` or `boot` method as an argument of your bootloader class

**For example:**
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

2. Using the `Bootloader::DEPENDENCIES` constant in your bootloader class. This can be a convenient way to define
   dependent bootloaders when you don't need to access them directly in your bootloader class.

**For example:**
```php
namespace App\Bootloader;

class MyBootloader extends Bootloader 
{
    protected const DEPENDENCIES = [
        \Spiral\Bootloader\Http\HttpBootloader::class
    ];
    
    public function boot(): void
    {
        // ...
    }
}
```

The Spiral Framework will automatically resolve and initialize the dependent bootloader before the depending bootloader 
is initialized.

Both of these approaches allow you to define dependent bootloaders in a declarative way, which can make it easier to 
manage the initialization of your application and ensure that all necessary resources and dependencies are available
when they are needed.

> **Note**
> Dependent bootloaders will be only initialized once, even if multiple other bootloaders depend on them.

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
