# Framework - Container and Factories

The framework includes a set of interfaces and components intended to simplify dependency injection and object
construction in your application.

> **Note**
> Container implementation is fully compatible with [PSR-11 Container](https://github.com/php-fig/container).

## PSR-11 Container

You can always access the container directly in your code by requesting `Psr\Container\ContainerInterface`:

```php
use Psr\Container\ContainerInterface;

class HomeController
{
    public function index(ContainerInterface $container): void
    {
        dump($container->get(App\App::class));
    }
}
```

## Dependency Injection

Spiral support both method and constructor injections for your classes:

```php
class UserMailer
{
    protected Mailer $mailer;

    public function __construct(Mailer $mailer)
    {
        $this->mailer = $mailer;
    }
    
    public function do(): void
    {
        $this->mailer->sendMail(...);
    }
}
```

The `Mailer` dependency will be automatically delivered by the container (auto-wiring).

> **Note**
> Note, your controllers, commands, and jobs support method injection.

## Configuring Container

You can configure container by creating a set of bindings between aliases or interfaces to concrete implementations.   
Use [Bootloaders](/framework/bootloaders.md) to define bindings.

We can either use `Spiral\Core\BinderInterface` or `Spiral\Core\Container` to configure the application container.

To bind interface to concrete implementation:

```php
use Spiral\Core\Container;

//...
public function boot(Container $container): void
{
    $container->bind(MyInterface::class, MyImplementation::class);
}
```

To bind singleton:

```php
use Spiral\Core\Container;

//...
public function boot(Container $container): void
{
    $container->bindSingleton(MyImplementation::class, MyImplementation::class);
}
```

To bind with specific parameters:

```php
use Spiral\Core\Container;

//...
public function boot(Container $container): void
{
    $container->bindSingleton(MyImplementation::class, bind(MyImplementation::class, [
        'param' => 'value'
    ]));
}
```

> **Note**
> Read about [Config Objects](/framework/config.md) to see how to manage config dependencies.

You can also use `closure` to configure your class automatically:

```php
use Spiral\Core\Container;

//...
public function boot(Container $container): void
{
    $container->bindSingleton(MyImplementation::class, function() {
        return new MyImplementation('some-value');
    });
}
```

Closures also support dependencies:

```php
use Spiral\Core\Container;

//...
public function boot(Container $container): void
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

You can skip singleton binding by implementing `Spiral\Core\Container\SingletonInterface` in your class:

```php
use Spiral\Core\Container\SingletonInterface;

class MyService implements SingletonInterface
{
    public function method(): void
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

In some cases, you might want to construct a desired class without resolving all of it's `__constructor` dependencies.
You can use `Spiral\Core\FactoryInterface` for that purpose:

```php
public function makeClass(FactoryInterface $factory): MyClass
{
    return $factory->make(MyClass::class, [
        'parameter' => 'value'
        // other dependencies will be resolved automatically
    ]); 
}
```

## ResolverInterface

If you want to resolve method arguments to dynamic target (i.e., controller method) use `Spiral\Core\ResolverInterface`:

```php
abstract class Handler
{
    protected ResolverInterface $resolver;

    public function __construct(ResolverInterface $resolver)
    {
        $this->resolver = $resolver;
    }

    public function run(array $params): bool
    {
        $method = new \ReflectionMethod($this, 'do'); // the method to invoke with method injection
        $method->setAccessible(true);

        return $method->invokeArgs(
            $this, 
            $this->resolver->resolveArguments($method, $params) // resolve missing arguments
        );
    }
}
```

Method `do` now can request method injection:

```php
class MyHandler extends Handler
{
    public function do(SomeClass $some): bool
    {
        // ...
    }
}
```

## InvokerInterface

In some cases, you might want to invoke a desired method with auto wiring all of it's dependencies.
You can use `Spiral\Core\InvokerInterface` for that purpose:

### Invoke object's instance method

```php
public function invokeMethod(\Spiral\Core\InvokerInterface $invoker): mixed
{
    return $invoker->invoke([$instance, 'handle'], [
        'parameter' => 'value'
        // other dependencies will be resolved automatically
    ]); 
}
```

if you pass as first callable array value a string `['foo', 'bar']`, so it will be requested from the container.

```php
$container->bind('some-job', SomeJob::class);

// ...
public function invokeMethod(\Spiral\Core\InvokerInterface $invoker): mixed
{
    return $invoker->invoke(['some-job', 'handle'], [ // some-job will be requested from the container
        'parameter' => 'value'
        // other dependencies will be resolved automatically
    ]); 
}
```

> **Note**
> Invokable method can have as `public` as `protected` and `privite` visibility.

### Invoke callable

In some cases, you might want to invoke a desired closure with auto wiring all of it's dependencies.

```php
public function invokeMethod(\Spiral\Core\InvokerInterface $invoker): mixed
{
    return $invoker->invoke(function (MyClass $class, string $parameter) {
        return new MyClassService($class);
    }, [
        'parameter' => 'value'
        // other dependencies will be resolved automatically
    ]); 
}
```

## Auto Wiring

Spiral Framework attempts to hide the container implementation and configuration from your domain layer by providing
rich auto-wiring functionality. Though auto-wiring rules are straightforward, it's essential to learn them to avoid
framework misbehavior.

### Automatic Dependency Resolution

Framework container can automatically resolve the constructor or method dependencies by providing instances
of concrete classes.

```php
class MyController
{
    public function __construct(
        OtherClass $class, 
        SomeInterface $some
    ) {
    }
}
```

In a provided example, the container will attempt to give the instance of `OtherClass` by automatically constructing it.
However, `SomeInterface` would not be resolved unless you have the proper binding in your container.

```php
$container->bind(SomeInterface::class, SomeClass::class); 
```

Please note, the container will try to resolve *all* constructor dependencies (unless you manually provide some values).
It means that all class dependencies must be available, or parameter must be declared as optional:

```php
// will fail if `value` dependency not provided
__construct(OtherClass $class, $value)

// will use `null` as `value` if no other value provided
__construct(OtherClass $class, $value = null) 

// will fail if SomeInterface does not point to the concrete implementation
__construct(OtherClass $class, SomeInterface $some) 

// will use null as value of `some` if no concrete implementation is provided
__construct(OtherClass $class, SomeInterface $some = null) 
```

### Contextual Auto Wiring

In addition to regular method injections, the container can resolve the injection context automatically. Such a
technique provides us the ability to request multiple databases using the following statement:

```php
protected function index(Database $primary, Database $secondary): void
{
    dump($primary);
    dump($secondary);
}
```

> **Note**
> Where `primary` and `secondary` are database names.

Implement `Spiral\Core\Container\InjectorInterface` to create injection factory and define class responsible for such
injections:

```php
class MyClassInjector implements InjectorInterface
{
    public function createInjection(\ReflectionClass $class, string $context = null)
    {
        return new MyClass($context);
    }
}
```

Where `MyClass` is:

```php
class MyClass 
{
    public string $name;
    public function __construct(string $name)
    {
        $this->name = $name;
    }
}
```

Make sure to register inject in the container:

```php
$container->bindInjector(MyClass::class, MyClassInjector::class);
```

As a result we can request instance of `MyClass` using method arguments:

```php
public function method(MyClass $john, MyClass $bob): void
{
    dump($john); // $john->name === 'john'
    dump($bob);  // $bob->name === 'bob'
}
```

You can always bypass contextual injection in `Spiral\Core\FactoryInterface`:

```php
dump($factory->make(MyClass::class, ['name' => 'abc']));
```

## Singletons

A lot of internal application services reside in a memory in the form of singleton objects. Such objects do not
implement static `getInstance`, but rather configured to remain in container **between requests**.

Declaring your service or controller as a singleton is the shortest path to get small performance improvement, however,
some rules must be applied to avoid memory and state leaks.

### Define the Singleton

The framework provides multiple ways to declare a class object as a singleton. At first, you can create Bootloader in
which you can bind class under its name:

```php
class ServiceBootloader extends Bootloader
{
    protected const SINGLETONS = [
        Service::class => Service::class
    ];
}
```

Now, you will always receive the same instance from the IoC container by doing the injection.

The alternative approach does not require Bootloader and can be defined via class itself, implement
interface `Spiral\Core\Container\SingletonInterface` to declare to the container that class must be constructed only
once:

```php
use Spiral\Core\Container\SingletonInterface;

class Service implements SingletonInterface
{
    // ...
}
``` 

> **Note**
> You can implement a singleton interface in services, controllers, middleware, etc. Do not implement it in Repositories
> and mappers as these classes state are managed by ORM.

### Limitations

By keeping your services in memory between requests, you can avoid doing some complex initializations and computations
over and over. However, you must remember that your services must be designed in **stateless** fashion and do not
contain any user data.

You can not:

- store user information in your singleton
- store reference to PSR-7 request (use `Spiral\Http\Request\InputManager` instead)
- store reference to session (use `Spiral\Session\SessionScope` instead)
- store RBAC actor (use `Spiral\Security\GuardScope` instead)

### Pre-Heating

You are allowed to store data in services that can not change between user requests. For example, if your application
relies on heavy XML as configuration source:

```php
class Service implements SingletonService 
{
    private ?array $configCache = null;

    public function getConfig(): array
    {
        if ($this->configCache !== null) {
            return $this->configCache;
        }
    
        $this->configCache = $this->readConfig(); // heavy operation
    
        return $this->configCache;
    }
}   
```

Using such an approach, you can perform complex computations only once and rely on RAM cache on later user requests.

## Replacing a container instance

In some cases, you might want to replace `container instance` in the application. You can do this when you create 
an application instance.

```php
use Spiral\Core\Container;

$app = App::create(
    directories: ['root' => __DIR__],
    container: new Container()
)
```
