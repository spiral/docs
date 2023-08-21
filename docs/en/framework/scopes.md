# Framework — IoC Scopes

An essential aspect of developing long-living applications is proper context management. In demonized applications,
you are no longer allowed to treat user requests as a global singleton object and store references to its instance in
your services.

Practically it means that you must explicitly request context while processing user input. Spiral offers a simple way to
manage this by using a global IoC (Inversion of Control) container. This acts as a context carrier that allows you to
request specific instances, scoped to a particular context, as if they were global objects.

## How It Works

When processing a user request, the relevant data for that request is placed in a scoped area of the IoC container. This
data is only available for a limited time while processing that request.

You can use the `Spiral\Core\ScopeInterface->runScope` method to achieve this. The default Spiral
container, `Spiral\Core\Container`, implements this interface.

**Here is an example**

```php
use Psr\Container\ContainerInterface;

$container->runScoped(
    closure: function (ContainerInterface $container) {
        dump($container->get(UserContext::class);
    },
    bindings: [UserContext::class => $user,],
);
```

In this example, the `runScoped` method is creating a temporary scope within which the `UserContext` class is bound to
the `$user` variable. The callback function can then use the container to retrieve the `UserContext` object instance,
which is available within the callback's scope.

> **Note**
> The framework ensures that the scope is cleaned up after execution, even if an exception is thrown.

## Working with Container Scopes

Starting from version **3.8.0 of the Spiral Framework**, a new interface named `Spiral\Core\ContainerScopeInterface` has
been introduced. This interface replaces the previous `Spiral\Core\ScopeInterface` and offers a more convenient and
advanced way to work with scopes in the Inversion of Control (IoC) container.

The Spiral Framework’s Container class now implements this interface. This means that when you are working with a
Container instance in Spiral, you can make use of these new features directly through the container instance.

### Method Signature

This interface allows you to run a specific block of code (a callable) within a uniquely defined and
isolated scope of the IoC container. Here's how it works:

```php
public function runScoped(callable $closure, array $bindings = [], ?string $name = null, bool $autowire = true): mixed
```

**Parameters**

- `$closure`: A callable that represents the block of code you want to run within the isolated scope.
- `$bindings`: An optional array of key-value pairs that define specific bindings for this scope.
- `$name`: An optional string that allows you to give a unique name to this scope.
- `$autowire`: A boolean flag (default is `true`) that specifies whether or not dependencies should be automatically
  resolved (autowired) or not.

The method returns the result of the passed callable.

### Basic Usage

When you call the `runScoped` method, it executes the provided `$closure` (a callable, e.g., an anonymous function) in a
separate, isolated scope. Any dependencies or bindings specified in the `$bindings` array will only be available within
that specific scope.

When `$autowire` is set to `true`, the IoC container will automatically resolve and inject all dependencies that the
given `$closure` requires based on their type-hints.

**Here's an example**

```php
$result = $container->runScoped(closure: function (SomeInterface $instance) {
    // Your code here
}, bindings: [SomeInterface::class => SomeImplementation::class]);
```

When `$autowire` is set to `false`, the container won't automatically resolve the `$closure`'s dependencies. Instead,
the entire container instance itself will be passed as the first argument to the `$closure`. This way, you can manually
fetch or manipulate the necessary dependencies from the container within the closure.

```php
$container->runScoped(closure: function (Contaner $container) {
    $instance = $container->get(SomeInterface::class);
    // Your code here
}, bindings: [SomeInterface::class => SomeImplementation::class], autowire: false);
```

### Configuring Bindings for Named Scopes

You have the option to pre-configure bindings specific to named scopes. There is `getBinder` method in the container
which allows you to fetch a `BinderInterface` instance that is associated with a specific named scope. With this binder,
you can set up default bindings for that scope.

```php
// Configure `root` scope bindings (the current instance)
$container->bindSingleton(Interface::class, Implementation::class);

// Configure `request` scope default bindings
// Prefer way to make many bindings
$binder = $container->getBinder('request');
$binder->bindSingleton(Interface::class, Implementation::class);
$binder->bind(Interface::class, factory(...));
```

#### Scope destroying

Once a scope is finished (destroyed), all the dependencies (including singletons) that were initialized within that
scope are also destroyed. This ensures clean and efficient resource management. The next time a scope with the same name
is created, these bindings will be resolved anew, following the configurations you've set using the `BinderInterface`.

#### Overriding Default Bindings

When you use the `Container::runScoped()` method to create a new scope, you can optionally pass the `$bindings` argument
to override the default bindings for that specific scope.

- These custom bindings will take precedence within the newly created scope.
- This won't affect the default bindings that you've previously defined for that named scope.

**Example:**

```php
$container->bindSingleton(SomeInterface::class, SomeImplementation::class);

$container->runScoped(closure: function ($container) {
    $instance = $container->get(SomeInterface::class);
    // Your code here
}, bindings: [SomeInterface::class => AnotherImplementation::class], name: 'request');
```

In this example, even though the `request` scope might have a default binding for `SomeInterface::class`, this specific
run of the scope is using A`notherImplementation::class`.

### Scope Destroying

When a dependency is resolved within a scope, and that scope is destroyed, the finalize method specified by this
attribute will be called. The purpose of the finalize method is to perform any necessary cleanup or finalization actions
before the scope is destroyed. In this case you can use `Finalize` attribute to control finalizing process. The
attribute provides a convenient way to handle resource cleanup and ensure proper destruction of objects when a scope is
no longer needed.

```php
#[Finalize(method: 'finalize')]
class Foo
{
    public bool $finalized = false;

    public function finalize(Logger $logger): void
    {
        $this->finalized = true;
        $logger->log();
    }
}
```

### Naming a Scope

You can optionally give a name to the scope with the `$name` parameter. This can be useful for tracking or managing
specific scopes.

**Restrictions**

These restrictions ensure proper usage and prevent conflicts within the scope hierarchy. Here are the key restrictions:

1. **Parallel Named Scopes:** Parallel named scopes are allowed.

2. **Default Bindings for Named Scopes:** Named scopes can have default bindings associated with them. It's important to
   note that changing of default bindings does not affect already created container instances with the same scope name,
   except for the root container. Default bindings provide a convenient way to set up common dependencies for a
   particular named scope.

3. **Root Scope Name:** The most parent scope in the hierarchy is named the root scope by default. The root scope serves
   as the top-level scope and provides the foundation for all other scopes. It's the starting point for dependency
   resolution and can have its own set of default bindings.

### Resolving Rules

When a nested scope does not have its own bindings, it will now resolve dependencies from the parent scope. 

### Accessing Scoped Values

You can access the values set in the scope directly from the container or via dependency injection in your services or
controllers. However, this should only be done within the IoC scope.

**Here's an example**

```php
public function doSomething(UserContext $user): void
{
    \dump($user);
}
```

In short, you can receive active context from container or injection inside the IoC scope as you would normally do
for any normal dependency, but you **must not store** it between scopes.

## Context Managers

As mentioned above, you are not allowed to store any reference to the scoped instance, the following code is invalid and
will cause the controller to lock on first scope value:

```php Wrong approach
class HomeController
{
    public function __construct(
        private readonly UserContext $userContext // <====== !!!not allowed!!!
    ) {
    }
    
    public function index(): void
    {
        dump($this->userContext);
    }
}
```

Instead, it is recommended to use objects specifically crafted to provide access to IoC scopes from singletons - Context
Managers. A simple context manager can be written as follows:

```php
final class UserScope
{
    public function __construct(
        private readonly ContainerInterface $container
    ) {
    }

    public function getUserContext(): ?UserContext
    {
        // error checking is omitted
        return $this->container->get(UserContext::class);
    }

    public function getName(): string
    {
        return $this->getUserContext()->getName();
    }
}
```

You can use this manager in any of your services, including singletons.

```php app/src/Endpoint/Web/HomeController.php
class HomeController implements SingletonInterface
{
    public function __construct(
        private readonly UserScope $userManager
    ) {
    }
}
```

> **Note**
> A good example is `Spiral\Http\Request\InputManager`. The manager operates as an accessor
> to `Psr\Http\Message\ServerRequestInterface` available only inside http dispatcher scope.
