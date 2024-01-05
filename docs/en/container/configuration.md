# Container — Configuration

The Spiral container allows you to configure it by creating bindings between interfaces or aliases to specific
implementations. You can use [Bootloaders](../framework/bootloaders.md) to define these bindings.

## Overview

There are two ways to configure the container, either by using the `Spiral\Core\BinderInterface` or
the `Spiral\Core\Container`.

### Bind interface to implementation

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

### Bind interface to singleton

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

### Check if container has binding

To check if a container has binding use:

```php
use Spiral\Core\Container;

public function boot(Container $container): void
{
    $container->has(UserRepositoryInterface::class)
}
```

### Remove binding

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

## Advanced bindings

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
container `$container->get('some-binding')`, the `DeferredFactory` will resolve an instance of `MyClass` and then invoke
the handle method on that instance to produce the required value.

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

An inflector allows you to manipulate an object after creating it in the container. This is particularly useful for
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

In this case, any object implementing `LoggerAwareInterface`
will have its logger set based on the specified configuration.

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

> **Note**
> Read more about container attributes in the [Container - Attributes](./attributes.md) section.

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

