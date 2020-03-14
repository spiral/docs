# Framework - Container and Factories
The framework includes a set of interfaces and components intended to simplify dependency injection and object construction
in your application.

> Container implementation is fully compatible with [PSR-11 Container](https://github.com/php-fig/container).

## PSR-11 Container
You can always access the container directly in your code by requesting `Psr\Container\ContainerInterface`:

```php
use Psr\Container\ContainerInterface;

class HomeContoller
{
    public function index(ContainerInterface $container)
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

The `Mailer` dependency will be automatically delivered by the container (auto-wiring).
 
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

You can also use `closure` to configure your class automatically:

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
You can skip singleton binding by implementing `Spiral\Core\Container\SingletonInterface` in your class:

```php
use Spiral\Core\Container\SingletonInterface;

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
In some cases, you might want to construct a desired class without resolving all of it's `__constructor` dependencies.
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

## ResolverInterface
If you want to resolve method arguments to dynamic target (i.e., controller method) use `Spiral\Core\ResolverInterface`:

```php
abstract class Handler
{
    /** @var ResolverInterface */
    protected $resolver;

    public function __construct(ResolverInterface $resolver)
    {
        $this->resolver = $resolver;
    }

    public function run(array $params)
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
    public function do(SomeClass $some)
    {
        // ...    
    }
}
```

## Auto Wiring
Spiral Framework attempts to hide the container implementation and configuration from your domain layer by providing rich auto-wiring functionality. Though auto-wiring rules are straightforward, it's essential to learn them to avoid framework misbehavior.

### Automatic Dependency Resolution
Framework container can automatically resolve the constructor or method dependencies by providing instances
of concrete classes.

```php
class MyController
{
    public function __construct(OtherClass $class, SomeInterface $some)
    {
    }
}
```

In a provided example, the container will attempt to give the instance of `OtherClass` by automatically constructing it. However,
`SomeInterface` would not be resolved unless you have the proper binding in your container.

```php
$container->bind(SomeInterface::class, SomeClass::class); 
```

Please note, the container will try to resolve *all* constructor dependencies (unless you manually provide some values). It means that
all class dependencies must be available, or parameter must be declared as optional:

```php
// will fail if `value` dependency not provided
__construct(OtherClass $class, $value)

// will use `null` as `value` if no other value provided
__construct(OtherClass $class, $value = null) 

// will fail if SomeInterface does not point to the concrete implemenation
__construct(OtherClass $class, SomeInterface $some) 

// will use null as value of `some` if no conrete implemation is provided
__construct(OtherClass $class, SomeInterface $some = null) 
```

### Contextual Auto Wiring
In addition to regular method injections, the container can resolve the injection context automatically. Such a technique provides us the ability to request multiple databases using the following statement:

```php
protected function index(Database $primary, Database $secondary)
{
    dump($primary);
    dump($secondary);
}
```

> Where `primary` and `secondary` are database names.

Implement `Spiral\Core\Container\InjectorInterface` to create injection factory:

Now we have to define class responsible for such injections:

```php
class Injector implements InjectorInterface
{
    public function createInjection(\ReflectionClass $class, string $context = null)
    {
        return new MyClass($context);
    }
}
```

Where MyClass is:

```php
class MyClass 
{
    public function __construct(string $name)
    {
        //...
    }
}
```

Make sure to register inject in the container:

```php
$container->bindInjector(MyClass::class, Injector::class);
```

As result we can request instance of `MyClass` using method arguments:

```php
public function method(MyClass $john, MyClass $bob)
{
    dump($john);
    dump($bob);
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
The framework provides multiple ways to declare a class object as a singleton. At first, you can create Bootloader in which
you can bind class under its name:

```php
class ServiceBootloader extends Bootloader
{
    protected const SIGNLETONS = [
        Service::class => Service::class
    ];
}
```

Now, you will always receive the same instance from the IoC container by doing the injection.

The alternative approach does not require Bootloader and can be defined via class itself, implement interface `Spiral\Core\Container\SingletonInterface`
to declare to the container that class must be constructed only once:

```php
use piral\Core\Container\SingletonInterface;

class Service implements SingletonInterface
{
    // ...
}
``` 

> You can implement a singleton interface in services, controllers, middleware, etc. Do not implement it in Repositories
and mappers as these classes state are managed by ORM.

### Limitations
By keeping your services in memory between requests, you can avoid doing some complex initializations and computations
over and over. However, you must remember that your services must be designed in **stateless** fashion and do not contain
any user data.

You can not:
- store user information in your singleton
- store reference to PSR-7 request (use `InputManager` instead)
- store reference to session (use `SessionScope` instead)
- store RBAC actor (use `GuardScope` instead) 

### Pre-Heating
You are allowed to store data in services that can not change between user requests. For example, if you application
relies on heavy XML as configuration source:


```php
class Service implements SingletonService 
{
    private $configCache;


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
