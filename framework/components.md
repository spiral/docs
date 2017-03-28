# Shared/Global container
In order to simplify access to container in your application and enable some of sugar functionality Spiral relays on global static instance of IoC container:

```php
\App::sharedContainer()->get('views')->render(...);

//Alternative
spiral('views')->render(...);
```

Please note that such container is only available inside your application and usually initiated by following constructions:

```php
$scope = self::staticContainer($container);
try {
    //Execute your application
} finally {
    self::staticContainer($scope);
}
```

Following approach provides ability to minimize amount of static instances and allow multiple spiral applications co-exists in one session.

> Shared container is widely used for support functionality such as loggers and shortcuts.

You can always get access to active container by extending `Component` in your code and utilizing function `iocContainer`:

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

> Note that some of spiral traits (i.e. LoggerTrait, PaginatorTrait, TranslatorTrait) require such method.

## Scoping
In some cases (usually inside middlewares and HMVC cores) you might want to define IoC scope only one isolated part of your application, use `replace`/`restore` methods of `ScoperInterface`/`ContainerInterface` for that:

```php
$outerBinding = $this->container->replace(SomeClass::class, new SomeClassB());
try {
    //Every dependency of `SomeClass` will be resolved with SomeClassB
} finally {
    //We can now restore original binding (if any)
    $this->container->restore($outerBinding);
}
```