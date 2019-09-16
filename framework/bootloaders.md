# Framework - Bootloaders
Bootloaders are the central piece in Spiral Framework and your application. This objects are responsible for [Container](/framework/container.md)
configuration, default configuration and etc.

Bootloaders only executed once while loading your application. Since your application will stay in memory for long you can
add as many code to your bootloaders as you want, it will not cause any performance effect on runtime.

![Application Control Phases](https://user-images.githubusercontent.com/796136/64906478-e213ff80-d6ef-11e9-839e-95bac78ef147.png)

## Simple Bootloader
You can create simple bootloader by extending `Spiral\Boot\Bootloader\Bootloader`:

```php
namespace App\Bootloader;

use Spiral\Boot\Bootloader\Bootloader;

class MyBootloader extends Bootloader 
{

}
```

Every bootloader must be activated in your application kernel. Add the class reference into `LOAD` or `APP` lists:

```php
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

Currently, your bootloader doesn't do anything. We can start by adding some container bindings.

> `APP` bootloader namespace is always loaded after `LOAD`, keep domain-specific bootloaders in it.

## Configuring Container
The most common use-case of bootloaders is to configure DI container, for example, we might want to bind multiple
implementations to their interfaces and/or construct some service.

We can use the method `boot` for these purposes. Method support method injection, so we can request any services we need:

```php
use Spiral\Core\Container;

class MyBootloader extends Bootloader 
{
    public function boot(Container $container) 
    {
        $container->bind(MyInterface::class, MyClass::class);
        
        $container->bindSingleton(MyService::class, function(MyClass $myClass) {
            return new MyService($myClass); 
        });
    }
}
```

Bootloaders provide the ability to simplify container binding definition using constants `BINDINGS` and `SINGLETONS`. 

```php
use Spiral\Core\Container;

class MyBootloader extends Bootloader 
{
    const BINDINGS = [
        MyInterface::class => MyClass::class
    ];

    public function boot(Container $container) 
    {
        $container->bindSingleton(MyService::class, function(MyClass $myClass) {
            return new MyService($myClass); 
        });
    }
}
```

You can also replace factory closures with factory methods:

```php
class MyBootloader extends Bootloader 
{
    const BINDINGS = [
        MyInterface::class => MyClass::class
    ];

    const SINGLETONS = [
        MyService::class => [self::class, 'myService']
    ];

    public function myService(MyClass $myClass): MyService
    {
        return new MyService($myClass); 
    }
}
```

## Configuring Application
Another common use case of bootloaders is to configure the framework prior to application launch. For example, we can declare
new route for our application or module:

```php
class MyBootloader extends Bootloader 
{
    public function boot(RouterInterface $router)
    {
        $router->addRoute(
            'my-route',
            new Route('/<action>', new Controller(MyController::class))
        );
    }
}
```

> Identically you can mount middleware, change tokenizer directories and much more.

## Depending on other Bootloaders
Some framework bootloaders can be used as a simple path to configure application settings. For example, we can
use `Spiral\Bootloader\Http\HttpBootloader` to add global PSR-15 middleware:

```php
class MyBootloader extends Bootloader 
{
    public function boot(HttpBootloader $http)
    {
        $http->addMiddleware(MyMiddleware::class);
    }
}
```

If you want to ensure that `HttpBootloader` has always been initiated prior to `MyBootloader` use constant `DEPENDENCIES`:


```php
class MyBootloader extends Bootloader 
{
    const DEPENDENCIES = [HttpBootloader::class];

    public function boot(HttpBootloader $http)
    {
        $http->addMiddleware(MyMiddleware::class);
    }
}
```

> Note, you are only able to use bootloaders to configure your components during the bootstrap phase (a.k.a. via another bootloader). The framework would not allow you to change any configuration value after component initialization.
