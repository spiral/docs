# Framework - IoC Scopes

An essential aspect of developing long-living applications is proper context management. In demonized applications,
you are no longer allowed to treat user requests as a global singleton object and store references to its instance in your
services.

Practically it means that you must explicitly request context while processing user input. Spiral framework simplifies
such requests by using a global IoC container as a context carrier which allows you to call request specific instances
as global objects via context bounded scopes.

## Explanation

While processing some user request, the context-specific data is located in the IoC scope, available in the container
only for a limited time. Such operation is performed using `Spiral\Core\ScopeInterface`->`runScope` method. Default
spiral container `Spiral\Core\Container` implements this interface.

```php
use Psr\Container\ContainerInterface;

$container->runScope(
    [
        UserContext::class => $user
    ],
    function (ContainerInterface $container) {
    
        dump($container->get(UserContext::class);

    }
);
```

The first argument of the `runScope` method is an array of values that should be bound to the container within the 
callback's scope.

The callback function receives the container as its argument and can use the container to retrieve the values that were 
bound to it within the callback's scope. This can be useful for creating a temporary scope within which certain values 
are available.

In this example, the `runScope` method is creating a temporary scope within which the `UserContext` class is bound to the 
`$user` variable. The callback function can then use the container to retrieve the `UserContext` object instance, which 
is available within the callback's scope.

> **Note**
> Framework will guarantee that scope is clean after the execution, even in case of any exception.

You can receive values set in scope directly from the container or as method injections in your services/controllers
while calling them **inside the IoC scope**:

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

```php
class HomeController implements SingletonInterface
{
    private UserContext $userContext; // <====== !!!not allowed!!!

    public function __construct(UserContext $userContext)
    {
        $this->userContext = $userContext;
    }
}
```

Instead, it is recommended to use objects specifically crafted to provide access to IoC scopes from singletons - Context
Managers. A simple context manager can be written as follows:

```php
class UserScope
{
    private ContainerInterface $container;

    public function __construct(ContainerInterface $container)
    {
        $this->container = $container;
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

```php
class HomeController implements SingletonInterface
{
    private UserScope $userScope; // allowed

    public function __construct(UserScope $userManager)
    {
        $this->userScope = $userManager;
    }
}
```

> **Note**
> A good example is `Spiral\Http\Request\InputManager`. The manager operates as an accessor
> to `Psr\Http\Message\ServerRequestInterface` available only inside http dispatcher scope.
