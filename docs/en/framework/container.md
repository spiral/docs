# Framework — Container and DI

Spiral includes a set of tools that makes it easier to manage dependencies and create objects in your code. One of the
main tools is the container, which helps you handle class dependencies and automatically "injects" them into the class.
This means that instead of creating objects and setting up dependencies manually, the container takes care of it for
you.

## PSR-11 Container

The Spiral container also follows a set of [PSR-11 Container](https://github.com/php-fig/container) standards, which
ensures compatibility with other libraries.

In your code, you can access the container by asking for the `Psr\Container\ContainerInterface`.

```php app/src/Endpoint/Web/UserController.php
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

```php app/src/Endpoint/Web/UserController.php
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

```php app/src/Endpoint/Web/UserController.php
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

## Advanced Binding

Starting from version **3.8.0 of the Spiral Framework** we have replaced the previous array-based structure for storing
information about bindings within the container. The new approach employs Data Transfer Objects (DTO), providing a more
structured and organized way to store binding information. With this change, developers now can easily configure
container bindings using these objects.

To demonstrate the enhanced container functionality, here's an example showcasing the new binding configuration:

```php
use Spiral\Core\Config\Factory;

$container->bind(LoggerInterface::class, new Factory(
    callable: static function() {
        return new Logger(....);
    }, 
    singleton: true
))
```

Here are the available binding DTOs:

### Alias

This simplified DTO allows you to create a link to another binding within the container.

```php
use Spiral\Core\Config\Alias;

$container->bind(
    \DatetimeImmutable::class, 
    static fn() => new \DatetimeImmutable()
);

$container->bind(
    'now', 
    new Alias(alias: \DatetimeImmutable::class)
);
```

In this example, we first bind the `\DatetimeImmutable` class to a factory function that creates a new instance
of `\DatetimeImmutable` every time it's requested. Then, we create an Alias binding named now and associate it with
the `\DatetimeImmutable` class binding.

Now you can request `\DatetimeImmutable` class from the container by alias $container->get('now')

The Alias also provides a second argument - `singleton`. By setting singleton to `true`, the aliased class becomes a
singleton. This means that whenever you request `$container->get('now')` from the container, it will return the same
instance every time. On the other hand, calling `$container->get(\DatetimeImmutable::class)` will return a new instance
of `\DatetimeImmutable` with the current time on each request.

### Autowire

The `Spiral\Core\Config\Autowire` binding serves as a wrapper for the `Spiral\Core\Container\Autowire` class, providing
an automated way to resolve and instantiate classes by injecting their dependencies.

```php
use Spiral\Core\Config\Autowire;
use Spiral\Core\Container\Autowire as AutowireAlias;

$container->bind(MyClass::class, new Autowire(
    autowire: new AutowireAlias(MyClass::class, ['foo' => 'bar']),
    singleton: true
));
```

The `singleton` argument, set to true in this example, indicates that the container should treat `MyClass` as a
singleton. When you request an instance of MyClass from the container `$container->get(MyClass::class)`, it will return
the same instance every time.

### Factory

The `Spiral\Core\Config\Factory` binding serves as a simple factory for creating mixed types using a closure.

```php
use Spiral\Core\Config\Factory;

$container->bind('time', new Factory(
    callable: static fn() => time(),
));
```

Every time when you request current time from the container `$container->get('time')` it will return current timestamp.

Additionally, the `Factory` binding can be configured as a singleton:

```php
$container->bind('time', new Factory(
    callable: static fn() => time(),
    singleton: true,
));
```

By setting the `singleton` argument to `true`. This means that whenever you request the time value from the
container `$container->get('time')`, it will return the same value across multiple invocations.

### DeferredFactory

The `Spiral\Core\Config\DeferredFactory` binding enables you to bind deferred factory to the container using a
special array callable.

```php
use Spiral\Core\Config\DeferredFactory;

$container->bind('some-binding', new DeferredFactory(
    factory: [MyClass::class, 'handle'],
));
```

In the example above, we bind the key some-binding to a `DeferredFactory` instance. The factory property is set
to `[MyClass::class, 'handle']`. When the some-binding key is requested from the
container `$container->get('some-binding')`, the `DeferredFactory` will create an instance of `MyClass` and then invoke
the handle method on that instance.

Additionally, binding can be configured as a singleton:

```php
$container->bind('some-binding', new DeferredFactory(
    factory: ...,
    singleton: true
));
```

### Scalar

The Scalar binding is a new functionality introduced for the container. It provides a convenient way to store and
retrieve static scalar values within the container. It can be useful for configuring paths, constants, or any other
scalar values needed by your application.

```php
use Spiral\Core\Config\Scalar;

$container->bind('app-path', new Scalar(value: '/var/www/my-app'));
```

### Shared

The Shared binding allows you to bind a persistent object to a key in the container. Once the object is created, it will
be reused every time the key is requested, and custom arguments cannot be provided during subsequent requests.

```php
use Spiral\Core\Config\Shared;

$container->bind(MyClass::class, new Shared(value: new MyClass(...)));
```

It's important to note that with the `Shared` binding, custom arguments cannot be provided during subsequent requests.
The object will be created with the initial arguments and reused as is.

It is particularly useful when you want to ensure that the same instance of an object is used throughout the
application. It provides persistence and prevents the creation of multiple instances with different arguments.

### Inflector

An inflector allows you to manipulate an object before returning it from the container. This is particularly useful for
applying common modifications or injections to objects of a specific type.

```php
use Spiral\Core\Config\Inflector;

$container->bind(LoggerAwareInterface::class, new Inflector(
    inflector: static function (LoggerAwareInterface $obj, LoggerInterface $logger): LoggerAwareInterface {
        $obj->setLogger($logger);

        return $obj;
    }
));
```

By configuring the Inflector binding, you can apply modifications or injections to objects of a specific type
automatically whenever they are requested from the container. In this case, any object
implementing `LoggerAwareInterface` will have its logger set based on the specified configuration.

The Inflector binding is a powerful tool for applying common modifications or injections consistently across objects in
your application. It simplifies the process of configuring and customizing objects retrieved from the container.

### WeakReference

The WeakReference feature allows you to work with weak reference objects within the container. Weak references are
references to an object that do not prevent the object from being garbage-collected when there are no strong references
to it.

```php
se Spiral\Core\Config\WeakReference;

$obj = new MyClass();

$container->bind(MyClass::class, new WeakReference(
    reference: new \WeakReference($obj)
));

$obj === $container->get(MyClass::class); // true

unset($obj);

$obj1 = $container->get(MyClass::class); // A new object will be created
$obj1 === $container->get(MyClass::class); // true
```

When you retrieve MyClass from the container `$container->get(MyClass::class)`, the container returns the original
object because it still exists. However, when you unset the `$obj` variable, removing the strong reference to the
object, it becomes eligible for garbage collection. Subsequent calls to `$container->get(MyClass::class)` will create a
new instance of `MyClass` because the original object has been garbage-collected.

Using weak references can be beneficial in certain scenarios where you want to have control over the object's lifecycle
and allow it to be garbage-collected when there are no more strong references to it.

## Lazy Singletons

The framework also allows you to use "lazy singletons" which are classes that are automatically treated as
singletons by the container without the need to explicitly bind them as such.

:::: tabs

::: tab Attribute
`Spiral\Core\Attribute\Singleton` allows you to mark a class as a singleton. By applying this attribute to a class, you
indicate that only a single instance of the class should be created and shared across the application. This attribute
can be used as an alternative to interfaces for specifying singleton behavior.

```php
use Spiral\Core\Attribute\Singleton;

#[Singleton]
final class UserService
{
    public function store(User $user): void
    {
        //...
    }
}
```

:::

::: tab SingletonInterface

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

:::
::::

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

## Attributes

Spiral provides a set of attributes that can be used to provide additional control over dependency resolution:

### Singleton

Using the `#[Singleton]` attribute can be handy in situations where we want to ensure that only one instance of a class
is created and used throughout the application, such as with configuration managers, database connections, or loggers.

#### Example

Suppose you have a configuration manager in your application that reads from a configuration file and provides access to
various configuration settings. It would be wasteful to reload and parse the configuration file every time you need to
access a setting. Using the #[Singleton] attribute ensures that once the configuration file is loaded and parsed, it
remains in memory for the duration of the application's lifetime.

```php
use Spiral\Core\Attribute\Singleton;

#[Singleton]
final class ConfigurationManager
{
    private readonly array $config;

    public function __construct()
    {
        $this->config = parse_ini_file('config.ini');
    }

    public function get(string $key)
    {
        return $this->config[$key] ?? null;
    }
}
```

### Scope

Allows you to set a specific scope in which a dependency can be resolved. If the dependency is attempted to be resolved
in a different scope, an exception will be thrown, indicating a scope mismatch. This attribute helps enforce strict
scoping rules and prevents dependencies from being mistakenly resolved in unintended scopes.

#### Example

Consider an application where certain resources or operations are restricted to authenticated users. For such resources,
you'd want to:

1. Ensure that the user is authenticated.
2. Make the authenticated user's data available throughout the application during the current request.

This class holds information about the authenticated user. This data is only meant to be available and resolved when the
user is authenticated (i.e., within an auth scope).

```php
use Spiral\Core\Attribute\Scope;

#[Scope('auth')]
final readonly class AuthenticatedUser
{
    public function __construct(
        private int $id,
        private string $name, 
        private string $email,
    ) {
    }
}
```

The middleware checks if the user is authenticated. If they are, it sets up an auth scope in the IoC container and binds
the authenticated user's data to the AuthenticatedUser class.

```php;
use Psr\Http\Server\MiddlewareInterface;
use Psr\Http\Message\ResponseInterface as Response;
use Psr\Http\Message\ServerRequestInterface as Request;
use Psr\Http\Server\RequestHandlerInterface as Handler;

final class AuthMiddleware implements MiddlewareInterface
{
    public function __construct(
        private readonly ContainerInterface $container
    ) {
    }

    public function process(Request $request, Handler $handler): Response
    {
        // This is a simplified authentication check.
        // In a real application, this might involve checking session data, JWT tokens, etc.
        if ($request->hasHeader('Authorization')) {
            // Fetch user data based on the authorization header. 
            // For simplicity, we're hardcoding user data here.
            $user = new AuthenticatedUser(1, 'John Doe', 'john.doe@example.com');

            // Set up the auth scope
            return $this->container->runScoped(
                closure: function (ContainerInterface $container) use ($next, $request) {
                    // Now, within this scope, you can get the authenticated user instance.
                    $authenticatedUser = $container->get(AuthenticatedUser::class);
                    
                    // Continue processing the request.
                    return $handler($request);
                },
                bindings: [AuthenticatedUser::class => $user],
                name: 'auth'
            );

        } else {
            // No authentication header found.
            // Return an unauthorized response or simply continue processing.
            return $handler($request);
        }
    }
}
```

Let's say we have a controller or service that requires the authenticated user's data. By attempting to resolve the
`AuthenticatedUser` class from the container, we can be sure we are either getting the authenticated user or that we are
within the auth scope (thanks to the #[Scope('auth')] attribute).

```php
final class UserProfileController
{
    public function getProfile(AuthenticatedUser $user)
    {
        // Use the $user data to fetch and return the profile.
    }
}
```

> **Note**
> Read more about container scopes in the [Framework — IoC Scopes](../framework/scopes.md) section.

### Finalize

Allows you to define a finalize method for a class. When a dependency is resolved within a scope, and that scope is
destroyed, the finalize method specified by this attribute will be called. The purpose of the finalize method is to
perform any necessary cleanup or finalization actions before the scope is destroyed. This attribute provides a
convenient way to handle resource cleanup and ensure proper destruction of objects when a scope is no longer needed.

#### Example

Consider you're working with a database connection. Once you're done with it, especially within a specific scope, you
might want to close the connection or release other resources.

```php
use Spiral\Core\Attribute\Finalize;

#[Finalize(method: 'closeConnection')]
final class DatabaseConnection
{
    private $connection;

    public function __construct()
    {
        // Initialize the database connection
    }

    public function query($sql)
    {
        // Execute the query on the database
    }

    public function closeConnection(): void
    {
        // Close the connection
    }
}
```

When using this class within a scope, once the scope ends, the closeConnection method will be invoked, ensuring that
resources are released:

```php
$container->runScoped(
    closure: function (DatabaseConnection $db) {
        // Execute some database operations
        $users = $db->query('SELECT * FROM users');
        // ...  
    },
    bindings: [DatabaseConnection::class => new DatabaseConnection()],
);

// Once the scope is destroyed, the connection is automatically closed.
```

> **Warning**
> The object can be leaked but finalized. You should avoid such situations.

```php
$root = new Container();
$obj = $root->get(Foo::class);
unset($root); // The Foo finalizer will be called

// Here we have a leaked finalized object. It is `$obj`.
```

All the attributes — `#[Finalize]`, `#[Singleton]`, and `#[Scope]` — are fully compatible with each other. This means
that you can use these attributes simultaneously on the same class, allowing for fine-grained control over the behavior
and lifecycle of your dependencies.

#### Example

Imagine you have a caching service where:

1. Only a single instance of the caching service should be created throughout the application's lifecycle (singleton).
2. The service should only be available within certain parts of your application, like within an HTTP request handling (
   scoped).
3. When the application is shutting down, or when the scope ends, you want to ensure all pending cache operations (like
   write-backs) are finalized and any resources (like open file handles or network connections) are closed.

```php
namespace App\Services;

use Psr\Log\LoggerInterface;
use Spiral\Core\Attribute\Finalize;
use Spiral\Core\Attribute\Scope;
use Spiral\Core\Attribute\Singleton;

#[Singleton]
#[Scope('http')]
#[Finalize(method: 'shutdown')]
final class CacheService
{
    private array $cache = [];
    public function __construct(
        private readonly LoggerInterface $logger,
    ) {
    }

    public function get(string $key): ?string
    {
        return $this->cache[$key] ?? null;
    }

    public function set(string $key, string $value): void
    {
        $this->cache[$key] = $value;
    }

    public function shutdown(): void
    {
        $this->logger->info("CacheService is finalizing.");
        
        // Flush the cache to a persistent storage, close any resources, etc.
        $this->cache = [];
    }
}
```

## Auto Wiring

Spiral attempts to hide container implementation and configuration from your domain layer by providing rich auto-wiring
functionality. Though auto-wiring rules are straightforward, it's essential to learn them to avoid framework
misbehavior.

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

> **See more**
> Read more about injectors in the [Advanced — Injectors](../advanced/injectors.md) section.

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

## Fibers support

The container supports PHP 8.2 fibers. The static class `\Spiral\Core\ContainerScope::getContainer()` will return the
correct container instance for the current fiber and scope, enabling seamless integration with fibers.