# Framework - Container and Factories
The framework includes a set of interfaces and components intended to simplify dependency injection and object construction
in your application.

> Container implementation is fully compatible with [PSR-11 Container](https://github.com/php-fig/container).

## PSR-11 Container
You can always access container directly in your code by requesting `Psr\Container\ContainerInterface`:

```php
public function testMethod(ContainerInterface $container)
{
    dump($container->get(App\App::class));
}
```

## Dependency Injection
Spiral support both method and constructor injections for your classes:

```php
class UserMailer
{
    protected $mailer = null;

    public function __construct(Mailer $mailer)
    {
        $this->mailer = $mailer;
    }
    
    public function do()
    {
        $this->mailer->sendMail(...);
    }
}
```

The `Mailer` dependency will be automatically delivered by the container. Read more in [Auto Wiring](/framework/auto-wiring.md).
 
> Note, your controllers, commands, and jobs support method injection.

## Configuring Container
You can configure container by creating a set of bindings between aliases or interfaces to concrete implementations.   
Use [Bootloaders](/framework/bootloaders.md) to define bindings.

We can either use `Spiral\Core\BinderInterface` or `Spiral\Core\Container` to configure the application container.

To bind interface to concrete implementation:

```php
public function boot(Container $container)
{
    $container->bind(MyInterface::class, MyImplementation::class);
}
```

To bind singleton:

```php
public function boot(Container $container)
{
    $container->bindSingleton(MyImplementation::class, MyImplementation::class);
}
```

To bind with specific parameters:

```php
public function boot(Container $container)
{
    $container->bindSingleton(MyImplementation::class, bind(MyImplementation::class, [
        'param' => 'value'
    ]));
}
```

> Read about [Config Objects](/framework/config.md) to see how to manage config dependencies.

You can also use `closure` to automatically configure your class:

```php
public function boot(Container $container)
{
    $container->bindSingleton(MyImplementation::class, function() {
        return new MyImplementation('some-value');
    });
}
```

Closures also support dependencies:

```php
public function boot(Container $container)
{
    $container->bindSingleton(MyImplementation::class, function(SomeClass $class) {
        return new MyImplementation($class);
    });
}
```

To check if container has binding use:

```php
dump($container->has(MyImplementation::class));
```

To remove the container binding:

```php
$container->removeBinding(MyService::class);
```

## Lazy Singletons
You can skip singleton binding by implementing `piral\Core\Container\SingletonInterface` in your class:

```php
class MyService implements SingletonInterface
{
    public function method()
    {
        //...
    }
}
```

Now, the container will automatically treat this class as a singleton in your application:

```php
protected function index(MyService $service)
{
    dump($this->container->get(MyService::class) === $service);
}
```

## FactoryInterface
In some cases, you might want to construct desired class without resolving all of it's `__constructor` dependencies.
You can use `Spiral\Core\FactoryInterface` for that purpose:

```php
public function makeClass(FactoryInterface $factory)
{
    return $factory->make(MyClass::class, [
        'parameter' => 'value'
        // other dependencies will be resolved automatically
    ]); 
}
```
