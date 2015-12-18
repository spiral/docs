# Container, Factory, DI
Spiral framework interfaces and components are mostly (but not entirelly) build around constructor, method injections and universal factories, however to provide you better development environment (and, in some cases, for performance reasons) you are able to use functionality of Service Locator or Container. Spiral utilizes [Interop/Container](https://github.com/container-interop/container-interop) agreement for that.

> Framework internally joins all tree interfaces together: ContainerInterface, FactoryInterface and ResolverInterface (resolves arguments for given method/constructor/closure reflection, see below). However, you are still recommended to request each of this interfaces separatelly if you can.

## Container (Service Locator)
Classic container example can be explained using following code:

```php
public function indexAction()
{
    echo $this->container->get('views')->render(...);
}
```

> We will explain how to bind your own services in container below.

This code is identical to method or constructor injection and only used to get shortcut to the most commonly used framework components and interfaces, it still possible to avoid using it:

```php
public function indexAction(ViewsInterface $views)
{
    echo $views->render(...);
}
```

> Attention, a lot of people do not recommend to use Service Locators/Container in your code blindly, it is very important to understand [the difference between them](https://www.google.com/search?q=service+locator+vs+dependency+injection) (especially code testability). 

You can read about how to make container bindings and bootload your application [here](/framework/bootloaders.md).

## Shared/Global container
Framework components and bundle can work using only given dependency injections (however some functionality like shared loggers, easy pagination will be disabled), but in some cases it's easier to construct your application when you have one global instance of container for your enviroment. Such container is automatically set at moment of core initialization and can be used in your code (for example in views) using such options:

```php
\App::sharedContainer()->get('views')->render(...);

//Alternative
spiral('views')->render(...);
```

> In default application flow shared container are identical to container which can be requested by interface using DI. If you can use DI istead of statically accessed instance - do that.

## Default bindings (shared components)
Your application comes pre-configured with some commonly used bingings which are set in framework and application bootloaders.

#### Binded in AppBootload (your application code)

Binding/Alias | Compoment                               
---           | ---                                     
twig          | Twig_Environment                       
app           | App                                    
faker         | Faker\Generator             
 
#### Binded in SpiralBootloader (framework bundle)

Binding/Alias   | Compoment                               
---             | ---      
memory          | Spiral\Core\HippocampusInterface
debugger        | Spiral\Debug\Debugger
console         | Spiral\Console\ConsoleDispatcher
http            | Spiral\Http\HttpDispatcher  
encrypter       | Spiral\Encrypter\Encrypter
files           | Spiral\Files\FileManager
tokenizer       | Spiral\Tokenizer\Tokenizer
locator         | Spiral\Tokenizer\ClassLocator 
translator      | Spiral\Translator\Translator
views           | Spiral\Views\ViewManager 
storage         | Spiral\Storage\StorageManager  
dbal            | Spiral\Database\DatabaseManager 
odm             | Spiral\ODM\ODM
orm             | Spiral\ORM\ORM
cache           | Spiral\Cache\CacheStore *(default cache store)*
db              | Spiral\Database\Entities\Database *(default database)*
mongo           | Spiral\ODM\Entities\MongoDatabase *(default database)*
request         | Psr\Http\Message\ServerRequestInterface *(only in http scope)*
route           | Spiral\Http\Routing\RouteInterface *(featched from scoped request attribute)*
session         | Spiral\Session\SessionStore *(featched from scoped request attribute)*
input           | Spiral\Http\Input\InputManage *(only in http scope)*
cookies         | Spiral\Http\Cookies\CookieManager *(only in http scope)*
router          | Spiral\Http\Routing\Router *(only in http scope)*
responses       | Spiral\Http\Responses\Responder  *(only in http scope)*

### Short/Virtual Bindings (sugar)
Following statement is possible in any of your service, controller or command which does have property 'container' (this classes already delcared constructor injector)

```php
public function indexAction()
{
    echo $this->container->get('views')->render(...);
}
```

As alternative framework provides special trait which can be used to bypass container property and access required binding directly using class `__get` method:

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
protected function container()
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

To better understand how SharedTrait works let look at it source code which might look obvious:

```php
trait SharedTrait
{
    /**
     * Shortcut to Container get method.
     *
     * @see ContainerInterface::get()
     * @param string $alias
     * @return mixed|null|object
     * @throws AutowireException
     * @throws SugarException
     */
    public function __get($alias)
    {
        if ($this->container()->has($alias)) {
            return $this->container()->get($alias);
        }

        throw new SugarException("Unable to get property binding '{$alias}'.");

        //no parent call, too dangerous
    }

    /**
     * @return InteropContainer
     */
    abstract protected function container();
}
```

Following methodic can work very well in combination with good IDE and provides very sufficient way to write or prototype your code:

![Short Bindings](https://raw.githubusercontent.com/spiral/guide/master/resources/virtual-bindings.gif)

At any moment in future, you can simply create needed property in your class and set it's value using dependency injection (see below).

> Attention, ideally you should only use short bindings for the set of components which are stated as supportive/infrastruture (i.e. twig, faker, views, cache etc.) and DI for your business logic.

## FactoryInterface


## Dependency Injection
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

Container will automatically resolse cascade of dependecies and return us valid instance of `UserMailer`. Resolved dependencies with either come from set of bindigs (see below) or container will constuct them automatically. In some cases, when class constructor requires set of dependencies container can not handle (for example some scalar values) we can use another method `construct` to specify set of parameter passed int our class:

```php
$userMailer = $this->container->construct(UserMailer::class, [
    'email' => 'some@email.com'
]);
```

### Method Injections
Besides using class consturtor to inject dependecies, container allow you to apply same procedute for some specified class method. Dependency injection on method pre-implemented for you in spiral `Controller` and `Command` classes:

```php
protected function indexAction(UserMailer $mailer)
{
    //...
}
```

> In additional to that spiral controllers and [services] (/application/services.md) support `init` method where you can put your dependencies without overwriting default service constructor. 

### Shared bindings
Shared bindings provide you ability to use any system component by requesting it like a class property:



TODO update docs

### Bindings
In given example we simply constructed instance of `UserMailer` without any additional operations, however in many cases (especially in spiral core components), 
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

> Container will provide itself as first argument to binded function. To be changed.

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
There are a lot of scenarios where some instance must be preserved across application, this can be archied using the `bindSingleton` method:

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

### Declarative Singleton binding and SINGLETON constant
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
protected function indexAction(MyService $service)
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

### Controllable/Contextual Injections
Spiral Container supports injection feature which might significanlty simplify access to some component instances (databases, cache adapters, storage buckets) in constructor and methods injections (however it will add little bit "magic" in your code):


```php
protected function indexAction(Database $primary, Database $secondary)
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
    public function createInjection(\ReflectionClass $class, $context)
    {
        return new Name($context);
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

You can also use controllable injections in combination with default container implementation this way:

```php
$this->container->get(DatabaseInterface::class, 'default');
```

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

## What is constructed using container?
Spiral container used to resolve every framework component, service,  controller, command or extenal module, this means you can freely declarate dependies in such classes.

> Spiral mainly use container as injector and class constructor rather than container with set of bindings (even more, string bingings like `ContainerInterface->get("something")` are highly forbidden in spiral core components, only named class requests are allowed - `ContainerInterface->get(SomeService::class)`). Spiral application might perfectly work without any binding defined outside of alises and interfaces configured in default spiral core.
