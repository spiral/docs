# Shared/Global container
In order to simplify access to IoC container in your application and enable some of "magick" functionality Spiral relays on global static instance of `ContainerInterface`:

```php
\App::sharedContainer()->get('views')->render(...);

//Alternative
spiral('views')->render(...);
```

Please note that such container is only available inside your application (run method) and usually initiated as scope by following constructions:

```php
$scope = self::staticContainer($container);
try {
    //Execute your application
} finally {
    self::staticContainer($scope);
}
```

This approach provides ability to avoid global state and allow multiple spiral applications co-exists in one session.

> Shared container support lazy functinonality for paginators, loggers and debug, all components can work without it.

You can always get access to active container by extending `Component` in your code and calling function `iocContainer`:

```php
protected function iocContainer()
{
    if (
        property_exists($this, 'container')
        && isset($this->container)
        && $this->container instanceof ContainerInterface
    ) {
        return $this->container;
    }

    //Fallback!
    return self::$staticContainer;
}
```

## Dependency Scoping
In some cases (usually inside middlewares or HMVC cores) you might want to create IoC scope for some of your classes/interfaces, use `replace`/`restore` methods of `ScoperInterface`/`ContainerInterface` to do that:

```php
$outerBinding = $this->container->replace(SomeClass::class, new SomeClassB());
try {
    //Every dependency of `SomeClass` will be resolved with SomeClassB
} finally {
    //We can now restore original binding (if any)
    $this->container->restore($outerBinding);
}
```
