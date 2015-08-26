# Spiral IoC Container
Spiral provides simplistic implementation of IoC container with ability to construct requested classes, bind instance resolvers and singletons. Container fully
support constructor and method injections and can be used in every day development.

## Accessing Container
You can request instance of spiral container by defining `ContainerInterface` depency in constructor of your class. If you working under spiral application
based on framework bundle, you will be able to access global container using binding "core" or requesting instance of `Core` or `Application` due both of this classes
extends generic container implementation.

> Most of spiral services, requests and controllers already have internal property "container" which you can use. In addition to that, spiral core will mount itself as global component container and can be accessible statically using "Component::container()" method. Static functionality used mainly in helper traits and
models as fallback to resolve needed service outside of application context.

### Container injection
Let's say that we want to create simple class to perfom mailing operation:

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

Based on provided example we can see that class requested dependendy "Mailer", we can either construct this class manually using existed instance of Mailer, or
we can dedicate this job to container.

```php
    $userMailer = $this->container->get(UserMailer::class);
```

Container will automatically recolse cascade of dependecies and return us valid instance of `UserMailer`. In some cases, when constructor requires set of 
dependencies container can not handle (for example some scalar values) we can use another method `construct`:

```php
    $userMailer = $this->container->construct(UserMailer::class, [
        'email' => 'some@email.com'
    ]);
```

### Method injections
By default you can declare bindings in any application class which is going to be resolves using Container, in addition to that both `Controller` and `Command` support
method injection:

```php
public function index(UserMailer $mailer) 
{
    //...
}
```

> In additional to that spiral controllers and services support `init` method where you can put your dependencies without overwriting default constructor.

### Bindings
In given example we simply constructed instance of `UserMailer` without any additional operations, hovewer in many cases (especially in spiral core components), 
class may declare dependency with interface rather than class:

```php
public function __construct(ConfiguratorInterface $configurator, ...)
```

In such scenario we can use container to create implementation binding (link interface and it's realization), for example:

```php
$this->container->bind(ConfiguratorInterface::class, MyConfigurator::class);
```

You can also bind closure functions to be used to resolve needed instance:

```php
$this->container->bind(ConfiguratorInterface::class, function(ContainerInterface $container) {
    return new MyConfigurator('...');
});
```

> Container will provide itself as first argument to binded function.

We can also bind one class implementation to another (make sure binded class extends target), this can allow you to mock some functionality, test or
even change application behaviour:

```php
$this->container->bind(ConfiguratorInterface::class, MyConfigurator::class);
//...
$this->container->bind(MyConfigurator::class, NewConfigurator::class);
```

Based on such binding every requested `MyConfigurator` or `ConfiguratorInterface` going to be resovled using `NewConfigurator` instance.
> You can bind not only one class name to another, but class name to some string binding, this might simplify some services access.

```php
$this->container->bind('configurator', NewConfigurator::class);
```

As before we can request needed instance using `get`, `construct` method or via magic `__get`:
```php
dump($this->container->get('configurator'));
dump($this->container->configurator);
```

### Singletons
There is a lot of scenarious where some instance must be preserved across application, this can be archied using `bindSingleton` method:

```php
$this->container->bindSingleton(ConfiguratorInterface::class, function(ContainerInterface $container) {
    return new MyConfigurator('...');
});
```

In this case `ConfiguratorInterface` will be constructed only once and resolved as instance of MyConfigurator for every requested dependency.
You can also bing already constructed instance to become a singleton:

```php
$this->container->bindSingleton(ConfiguratorInterface::class, $this);
```

### Automatic Singleton binding and SINGLETON constant
Spiral Container provides internal contract which allow to state that class must be a singleton using class declaration. To mark your class
as singleton simple define "SINGLETON constant" and implement `SingletonInterface`:

```php
class MyService implements SingletonInterface
{
    const SINGLETON = self::class;

    public function method()
    {
        //...
    }
}
```

SINGLETON constant must be pointing to binding to store constructed instance, by default we are going to use class name, which is identical to:

```php
$this->container->bindSingleton(MyService::class, new MyService(...));
```

This implementation provides us ability to avoid setting up singleton bindings in application bootstrap which can significantly improve performance.

```php
public function index(MyService $service)
{
    dump($this->container->hasInstance(MyService::class));
    dump($service === $this->container->get(MyService::class));
}
```

> Most of spiral components has defined SINGLETON constant.

As in other cases you can replace exsited singleton realization with custom classes:
```php
  $this->container->hasInstance(MyService::class, NewService::class);
```

Once MyServive will be requested, Container will route request to `NewService` implementation and store it under binging defined in `MyService` class. In other
words - every class extends singleton will become singleton to replace it's parent.

> You can always disable singleton behaviour by inheriting class and defining SINGLETON constant as `null`.

### Controllable Injections
Spiral Container supports injection feature which might significanlty simplify access to some component instances (databases, cache adapters, storage buckets) in constructor and methods injections (however it will add little bit "magic" in your code):


```php
public function index(Database $primary, Database $secondary)
{
    dump($primary);
    dump($secondary);
}

public function cached(CacheStore $storeA, MemcacheStore $storeB)
{
    dump($storeA);
    dump($storeB);
}
```

Both of this examples (this is controller code) demonstrates principal of controllable injections which based on dedicating class injection to it's factory using context.

Let's view how our class may look like to defined it's own injectable:

```php
class Name implements InjectableInterface
{
    const INJECTOR = Injector::class;
    
    //This argument can not be automatically set by Container if we trying to resolve
    //this class in our arguments
    public function __constrcut($name)
    {
        //...
    }
}
```

When injector will look like:

```php
class Injector implements InjectorInterface
{
    public function createInjection(\ReflectionClass $class, \ReflectionParameter $parameter)
    {
        return new Name($parameter->getName());
    }
}
```

As result we can request instance of `Name` using method arguments:

```php
public function method(Name $john, Name $bob)
{
    dump($john);
    dump($bob);
}
```

> Attention, i think this feature is cool, but you must be accurate using it.

### Additional Container methods
There is few additional methods in Container you might consider using.

#### Check if Container has given binding
To check if container has specified binding, let's use `has` method:

```php
dump($this->container->has(MyService::class));
```

#### Check if Container has binded singleton
To check if container has binded instance (singleton) we can use `hasInstance` method.

```php
dump($this->container->hasInstance(MyService::class));
```

#### Remove container binding
If you want to remove existed container binding you can do it using `removeBinding` method, this will automatically destroy any class or singleton associated
with such binding if no other references for class instance exists.

```php
$this->container->removeBinding(MyService::class);
```

### Scoping
There is few scenarious when you might want to create some binding only for specific part of code.

```php
$outerBinding = $this->container->replace(SomeClass::class, new SomeClassB());

//Every dependency of `SomeClass` will be resolved with SomeClassB

//We can now restore original binding (if any)
$this->container->restore($outerBinding);
```

> Container scoping used by `HttpDispatcher` to set active instance of `ServerRequestInterface`.

## What is resolved using container?
Spiral container used to resolve every framework component, service,  controller, command or extenal module, this means you can freely declarate dependies in such classes.
