# Framework — Container and DI

The Spiral framework includes a set of tools that makes it easier to manage dependencies and create objects in your
code. One of the main tools is the container, which helps you handle class dependencies and automatically "injects" them
into the class. This means that instead of creating objects and setting up dependencies manually, the container takes
care of it for you.


> **Note**
> Container implementation is fully compatible with .

## PSR-11 Container

The Spiral container also follows a set of [PSR-11 Container](https://github.com/php-fig/container) standards, which
ensures compatibility with other libraries.

In your code, you can access the container by asking for the `Psr\Container\ContainerInterface`.

```php app/src/Interface/Controller/UserController.php
final class UserController
{
    public function __construct(
        private readonly \Psr\Container\ContainerInterface $container
    ) {}

    public function show(string $id): void
    {
       $repository = $this->container->get(UserRepository::class);
       // ...
    }
}
```

## Dependency Injection

The Spiral container allows you to use both **constructor** and **method** injections for your classes. This means that
you can have dependencies automatically **injected** into the class through the constructor, or through a specific
method.

:::: tabs

::: tab Constructor

For example, in the `UserController` class, the `UserRepository` dependency is injected through the `__construct()`
method. This means that the container will automatically create and deliver the UserRepository object when the method is
called.

```php app/src/Interface/Controller/UserController.php
use Psr\Container\ContainerInterface;

final class UserController
{
    public function __construct(
        private readonly UserRepository $users
    ) {}

    public function show(string $id): void
    {
       $user = $this->users->findOrFail($id);
       // ...
    }
}
```

:::

::: tab Method

For example, in the `UserController` class, the `UserRepository` dependency is injected through the `show()` method.
This means that the container will automatically create and deliver the UserRepository object when the method is
called.

```php app/src/Interface/Controller/UserController.php
final class UserController
{
    public function show(UserRepository $users, string $id): void
    {
       $user = $users->findOrFail($id);
       // ...
    }
}
```

:::
::::

> **Note**
> This feature is available for classes like controllers, console commands, and queue jobs.
> Additionally, the container also supports advanced features like union types, variadic arguments, referenced
> parameters and default object values for the auto-wiring.

## Configuring Container

The Spiral container allows you to configure it by creating bindings between interfaces or aliases to specific
implementations. You can use [Bootloaders](../framework/bootloaders.md) to define these bindings.

There are two ways to configure the container, either by using the `Spiral\Core\BinderInterface` or
the `Spiral\Core\Container`.

To bind an interface to a concrete implementation, you can use the following code snippet:

:::: tabs

::: tab BinderInterface

```php app/src/Application/Bootloader/AppBootloader.php
use Spiral\Core\BinderInterface;

public function boot(BinderInterface $binder): void
{
    $binder->bind(
        UserRepositoryInterface::class, 
        CycleUserRepository::class
    );
}
```

:::

::: tab Container

```php app/src/Application/Bootloader/AppBootloader.php
use Spiral\Core\Container;

public function boot(Container $container): void
{
    $container->bind(
        UserRepositoryInterface::class, 
        CycleUserRepository::class
    );
}
```

:::
::::

To bind a singleton, you can use the following code snippet:

```php app/src/Application/Bootloader/AppBootloader.php
use Spiral\Core\BinderInterface;

public function boot(BinderInterface $binder): void
{
    $binder->bindSingleton(
        UserRepositoryInterface::class, 
        CycleUserRepository::class
    );
}
```

You can also bind with specific parameters by using the `Autowire` class like this:

```php app/src/Application/Bootloader/AppBootloader.php
use Spiral\Core\BinderInterface;
use Spiral\Core\Container\Autowire;

public function boot(BinderInterface $binder): void
{
    $binder->bindSingleton(
        UserRepositoryInterface::class, 
        new Autowire(CycleUserRepository::class, ['table' => 'users'])
    );
}
```

Lastly, you can use closures to configure your class automatically by passing a closure to the bind or `bindSingleton`
method like this:

```php app/src/Application/Bootloader/AppBootloader.php
use Spiral\Core\BinderInterface;

public function boot(BinderInterface $binder): void
{
    $binder->bindSingleton(
        UserRepositoryInterface::class, 
        static fn() => new CycleUserRepository(table: users)
    );
}
```

Closures also support dependencies:

```php app/src/Application/Bootloader/AppBootloader.php
use Spiral\Core\BinderInterface;
use Spiral\Core\Container\Autowire;

public function boot(BinderInterface $binder): void
{
    $binder->bindSingleton(
        UserRepositoryInterface::class, 
        static fn(UserConfig $config) => new CycleUserRepository(table: $config->getTable())
    );
}
```

When this closure is executed, the container will automatically resolve an instance of `UserConfig` and pass it as an
argument to the closure. This allows you to easily configure your classes with dependencies without having to manually
instantiate and manage them.

To check if a container has binding use:

```php
use Spiral\Core\Container;

public function boot(Container $container): void
{
    $container->has(UserRepositoryInterface::class)
}
```

To remove the container binding:

```php
use Spiral\Core\BinderInterface;

public function boot(BinderInterface $binder): void
{
    $binder->removeBinding(UserRepositoryInterface::class)
}
```

Container supports `WeakReference` binding:

:::: tabs

::: tab String alias

```php app/src/Application/Bootloader/AppBootloader.php
use Spiral\Core\Container;
use WeakReference;

public function boot(Container $container): void
{
    $object = new stdClass();
    $hash = \spl_object_hash($object);
    $reference = WeakReference::create($object);

    $container->bind('test-alias', $reference);
    
    dump($hash === \spl_object_hash($container->get('test-alias'))); // true
    
    unset($object);
    // New object can't be created because classname has not been stored
    dump($container->get('test-alias')); // null
}
```

:::

::: tab Class name alias

```php app/src/Application/Bootloader/AppBootloader.php
use Spiral\Core\Container;
use WeakReference;

public function boot(Container $container): void
{
    $object = new stdClass();
    $hash = \spl_object_hash($object);
    $reference = WeakReference::create($object);

    $container->bind(stdClass::class, $reference);
    
    dump($hash === \spl_object_hash($container->get(stdClass::class))); // true
    
    unset($object);
    // new instance created using alias class
    dump($hash === \spl_object_hash($container->get(stdClass::class))); // false
}
```

:::
::::

## Lazy Singletons

The framework also allows you to use "lazy singletons" which are classes that are automatically treated as
singletons by the container without the need to explicitly bind them as such.

To use this feature, you can simply implement the `Spiral\Core\Container\SingletonInterface` interface in your class,
like this:

```php
use Spiral\Core\Container\SingletonInterface;

final class UserService implements SingletonInterface
{
    public function store(User $user): void
    {
        //...
    }
}
```

Now, the container will automatically treat this class as a singleton and only create one instance of it for the entire
application, regardless of how many times it is requested.

```php
protected function index(UserService $service): void
{
    dump($this->container->get(UserService::class) === $service);
}
```

## FactoryInterface

The framework also provides a `Spiral\Core\FactoryInterface` that you can use to construct a class without resolving
all of its constructor dependencies. This can be useful in situations where you only need a specific subset of a class's
dependencies or if you want to pass in specific values for some of the dependencies.

```php
public function makeClass(FactoryInterface $factory): MyClass
{
    return $factory->make(UserService::class, [
        'table' => 'users'
    ]); 
}
```

In the example above, the `make()` method is used to create an instance of `UserService` and pass in the value `table`
for the parameter constructor dependency. The other constructor dependencies will be resolved automatically by the
container.

This allows you to have more control over the construction of your classes, and can be particularly useful in situations
where you need to create multiple instances of a class with different constructor dependencies.

## ResolverInterface

The framework also provides the `Spiral\Core\ResolverInterface` that you can use to resolve method arguments to
dynamic targets, such as controller methods. This can be useful in situations where you want to invoke a method and need
to resolve its dependencies at runtime.

```php
abstract class Handler
{
    public function __construct(
        protected readonly ResolverInterface $resolver
    ) {
    }

    public function run(array $params): bool
    {
        $method = new \ReflectionMethod($this, 'do');

        return $method->invokeArgs(
            $this, 
            $this->resolver->resolveArguments($method, $params) // resolve missing arguments
        );
    }
}
```

The `run()` method uses the `ResolverInterface` to invoke the `do` method with method injection. Now, the `do` method
can request method injection:

```php
class UserStoreHandler extends Handler
{
    public function do(UserService $service): bool
    {
        // Store user
    }
}
```

This way, you can easily resolve the dependencies of a method at runtime and invoke it with the required arguments,
regardless of whether the dependencies are passed in as parameters or if they need to be resolved by the container.

### Supported types

#### Union types

The default implementation of `ResolverInterface` supports Union types. One of the available dependencies of
the needed type will be passed:

```php
use Doctrine\Common\Annotations\Reader;
use Spiral\Attributes\ReaderInterface;

final class Entities
{
    public function __construct(
        private Reader|ReaderInterface $reader
    ) {
    }
}
```

#### Variadic arguments

```php
$resolver = $this->container->get(ResolverInterface::class);
$function = static fn(int ...$bar) => $bar;

// array passed by parameter name
$args = $resolver->resolveArguments(
    new \ReflectionFunction($function),
    ['bar' => [1, 2]]
);

dump($args); // [1, 2]

// array passed by parameter name with named arguments inside
$args = $resolver->resolveArguments(
    new \ReflectionFunction($function),
    ['bar' => ['ab' => 1, 'bc' => 2]]
);

dump($args); // ['ab' => 1 'bc' => 2]

// value passed by parameter name
$args = $resolver->resolveArguments(
    new \ReflectionFunction($function),
    ['bar' => 1]
);

dump($args); // [1]

```

#### Reference arguments

```php
$resolver = $this->container->get(ResolverInterface::class);
$function = static fn(int $bar) => $bar;

$bar = 1;

$args = $resolver->resolveArguments(
    new \ReflectionFunction($function),
    ['bar' => &$bar]
);

$bar = 42;
dump($args); // [42]
```

#### Default object value

```php
$resolver = $this->container->get(ResolverInterface::class);
$function = static fn(stdClass $std = new \stdClass()) => $std;

$args = $resolver->resolveArguments(new \ReflectionFunction($function));

dump($args); 

// array(1) {
//   [0] =>
//   class stdClass#369 (0) {
//   }
// }
```

### Arguments validation

In some cases, you may want to validate a function or method arguments. To do this, you can use the public
`validateArguments` method, in which you need to pass `ReflectionMethod` or `ReflectionFunction` and
`array of arguments`. If you received the arguments using the `resolveArguments` method and didn't pass `false` in the
`$validate` parameter, they don't need additional validation. They will be checked automatically.
If the arguments are not valid, `Spiral\Core\Exception\Resolver\InvalidArgumentException` will be thrown.

```php
$resolver = $this->container->get(ResolverInterface::class);
$function = static fn(int $bar) => $bar;

$resolver->validateArguments(new \ReflectionFunction($function), [42]);
```

## InvokerInterface

The framework also provides the `Spiral\Core\InvokerInterface` that you can use to invoke a desired method with
automatic resolution of its dependencies. This can be useful in situations where you want to invoke a method on an
object and need to resolve its dependencies at runtime.

### Invoke a class method

The InvokerInterface has a `invoke()` method that you can use to invoke a method on an object and pass in specific
values for its dependencies.

Here is an example of how you can use it:

```php
use Spiral\Core\InvokerInterface;

abstract class Handler
{
    public function __construct(
        protected readonly InvokerInterface $invoker
    ) {
    }

    public function run(array $params): bool
    {
        return $this->invoker->invoke([$this, 'do'], $params)
    }
}
```

if you pass as first callable array value a string `['user-service', 'store']`, it (`user-service`) will be requested
from the container.

```php
$container->bind('user-service', UserService::class);
// ...
$invoker->invoke(
    ['user-service', 'store'], 
    $params
);
```

The container will resolve the class `user-service` and call the method `store` on it.

This allows you to easily invoke methods on classes that are managed by the container without having to manually
instantiate them. This is particularly useful in situations where you want to use a class as a service and invoke its
methods from multiple parts of your codebase.

> **Note**
> A method can have any visibility (public, protected, or private) and still be invoked.

### Invoke callable

The `InvokerInterface` can also be used to invoke closures and automatically resolve their dependencies.

```php
$invoker->invoke(
    function (MyClass $class, string $parameter) {
        return new MyClassService($class);
    },
    [
        'parameter' => 'value',
    ]
); 
```

This allows you to easily invoke closures with the required dependencies, regardless of whether the dependencies are
passed in as parameters or if they need to be resolved by the container.

## Auto Wiring

Spiral Framework attempts to hide container implementation and configuration from your domain layer by providing
rich auto-wiring functionality. Though auto-wiring rules are straightforward, it's essential to learn them to avoid
framework misbehavior.

### Automatic Dependency Resolution

A framework container can automatically resolve the constructor or method dependencies by providing instances
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

In the provided example, the container will attempt to give the instance of `OtherClass` by automatically constructing
it. However, `SomeInterface` would not be resolved unless you have proper binding in your container.

```php
$container->bind(SomeInterface::class, SomeClass::class); 
```

Please note that the container will try to resolve *all* constructor dependencies (unless you manually provide some
values). It means that all class dependencies must be available, or parameter must be declared as optional:

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

### Autowire class

The `Spiral\Core\Container\Autowire` class allows you to delegate options to the container and pass specific
configuration values to your classes without hardcoding them. This can be useful for keeping your configuration separate
from your code and for making it easier to modify your application's behavior without changing the code itself.

```php app/config/session.php
use Spiral\Core\Container\Autowire;
use Spiral\Session\Handler\FileHandler;

return [
    // ...
    'handler' => new Autowire(
        FileHandler::class,
        [
            'directory' => directory('runtime') . 'session',
            'lifetime' => (int)env('SESSION_LIFETIME', 86400),
        ]
    ),
];
```

When the container tries to resolve the `Autowire`, it will automatically create an instance of the `FileHandler`
class and pass the `directory` and `lifetime` options to the constructor.

This allows you to easily configure your classes and pass in specific options without having to hardcode them in your
code, and can be particularly useful for configuring classes that have many options or that are used in multiple parts
of your codebase.

### Contextual Auto Wiring

In addition to regular method injections, a container can resolve the injection context automatically. Such a
technique provides us with an ability to request multiple databases using the following statement:

```php
protected function index(Database $primary, Database $secondary): void
{
    dump($primary);
    dump($secondary);
}
```

Where `primary` and `secondary` are database names.

> **Note**
> Read more about injectors in [Advanced — Injectors](../advanced/injectors.md) section.

## Singletons

A lot of internal application services reside in a memory in the form of singleton objects. Such objects do not
implement static `getInstance`, but are rather configured to remain in the container **between requests**.

Declaring your service or controller as a singleton is the shortest path to get small performance improvement, however,
some rules must be applied to avoid memory and state leaks.

### Define the Singleton

The framework provides multiple ways to declare a class object as a singleton. At first, you can create Bootloader in
which you can bind a class under its name:

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

## Replacing an application container

In some cases, you might want to replace a container instance in the application. You can do this when you create
an application instance.

```php app.php
use Spiral\Core\Container;
use App\Application\Kernel;

$container = new Container();
$container->bind(...);

$app = Kernel::create(
    directories: ['root' => __DIR__],
    container: $container
)
```