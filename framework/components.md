
## Shared/Global container
Framework components and bundles can work using only dependency injections (however some functionality like shared loggers, easy pagination will be disabled), but in some cases it's easier to construct your application when you have one global instance of container for your environment. Such container only available when your application receives incoming request in order to avoid collision with other applications in a same runtime process.

```php
\App::sharedContainer()->get('views')->render(...);

//Alternative
spiral('views')->render(...);
```

You can extend your classes from `Spiral\Core\Component` which will automatically define `iocContainer()` method capable of automatic routing container request between local (object specific) and global (shared/static) containers.

## Scoping
There is few scenarious when you might want to create some binding only for specific part of code. Such way used a lot inside `HttpDispatcher` and it's middlewares as it provides ability to isolate application request:

```php
$outerBinding = $this->container->replace(SomeClass::class, new SomeClassB());

try {
    //Every dependency of `SomeClass` will be resolved with SomeClassB
} finally {
    //We can now restore original binding (if any)
    $this->container->restore($outerBinding);
}
```

> Almost every framework components/class created using DI or factory, please **do not construct** framework classes directly in your application, use FactoryInterface/Container/DI instead (to prevent breaking changes in future and make your application more flexible).





### Short/Virtual Bindings (sugar)
Following statement is possible in any of your service, controller or command which does have property 'container' (this classes already delcared constructor injector)

```php
public function indexAction()
{
    echo $this->container->get('views')->render(...);
}
```

As alternative, framework provides special trait which can be used to bypass container property and access required binding directly using class `__get` method:

```php
//Already used by Command, Service and Controllers
use SharedTrait;

public function indexAction(ViewsInterface $views)
{
    dump($this->views === $this->container->get('views'));
    dump($this->views === $views);
    
    echo $this->views->render(...);
}
```

> Attention, `SharedTrait` only requires method `container()` to be defined. Most of spiral classes extends `Spiral\Core\Component` which provides ability to route container request to local container (property `container`) first and then to global/shared container.

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

> Try to make sure that every class which uses SharedTrait declares and sets `container` property so you can easily test it.

Following methodic can work very well in combination with good IDE and provides very sufficient way to write or prototype your code:

![Short Bindings](https://raw.githubusercontent.com/spiral/guide/master/resources/virtual-bindings.gif)

At any moment in future, you can simply create needed property in your class and set it's value using dependency injection (see below).