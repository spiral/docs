# Container — Overview

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

## Services

### Binder

The framework provides a `Spiral\Core\BinderInterface` that you can use to bind a class to an interface or alias. 
Read more about it in the [Container — Configuration](../container/configuration.md) section.

### Factory

The framework provides a `Spiral\Core\FactoryInterface` that you can use to construct a class without resolving
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

### Resolver

The framework provides the `Spiral\Core\ResolverInterface` that you can use to resolve method arguments to dynamic 
targets, such as controller methods. This can be useful in situations where you want to invoke a method and need to 
resolve its dependencies at runtime.

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

#### Supported types

##### Union types

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

##### Variadic arguments

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

##### Reference arguments

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

##### Default object value

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

#### Arguments validation

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

### Invoker

The framework provides the `Spiral\Core\InvokerInterface` that you can use to invoke a desired method with automatic 
resolution of its dependencies. This can be useful in situations where you want to invoke a method on an object and 
need to resolve its dependencies at runtime.

#### Invoke a class method

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

#### Invoke callable

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

### Scopes

An essential aspect of developing long-living applications is proper context management. In demonized applications, you
are no longer allowed to treat user requests as a global singleton object and store references to its instance in your
services.

Practically it means that you must explicitly request context while processing user input. Spiral offers a simple way 
to manage this by using a global IoC (Inversion of Control) container. This acts as a context carrier that allows you 
to request specific instances, scoped to a particular context, as if they were global objects.

Read more about **scopes** in the [Container — IoC Scopes](../container/scopes.md) section.

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