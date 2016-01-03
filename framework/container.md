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
memory          | Spiral\Core\ContainerInterface *(Factory + Interop Container)*
debugger        | Spiral\Debug\Debugger
console         | Spiral\Console\ConsoleDispatcher
http            | Spiral\Http\HttpDispatcher  
encrypter       | Spiral\Encrypter\EncrypterInterface
files           | Spiral\Files\FilesInterface
tokenizer       | Spiral\Tokenizer\TokenizerInterface
locator         | Spiral\Tokenizer\ClassLocatorInterface
translator      | Spiral\Translator\TranslatorInterface
views           | Spiral\Views\ViewManager
storage         | Spiral\Storage\StorageInterface 
dbal            | Spiral\Database\DatabaseManager 
odm             | Spiral\ODM\ODM
orm             | Spiral\ORM\ORM
cache           | Spiral\Cache\StoreInterface *(default cache store)*
db              | Spiral\Database\Entities\Database *(default database)*
mongo           | Spiral\ODM\Entities\MongoDatabase *(default database)*
request         | Psr\Http\Message\ServerRequestInterface *(only in http scope)*
route           | Spiral\Http\Routing\RouteInterface *(featched from scoped request attribute)*
session         | Spiral\Session\SessionInterface *(featched from scoped request attribute)*
input           | Spiral\Http\Input\InputManager *(only in http scope)*
cookies         | Spiral\Http\Cookies\CookieManager *(only in http scope)*
router          | Spiral\Http\Routing\RouterInterface *(only in http scope)*
response        | Spiral\Http\Responses\Responder  *(only in http scope)*

> Attention, some bindings are pointing to concrete implementations!

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

Following methodic can work very well in combination with good IDE and provides very sufficient way to write or prototype your code:

![Short Bindings](https://raw.githubusercontent.com/spiral/guide/master/resources/virtual-bindings.gif)

At any moment in future, you can simply create needed property in your class and set it's value using dependency injection (see below).

> Ideally you should only use short bindings for the set of components which are stated as supportive/infrastruture (i.e. twig, faker, views, cache etc.) and DI for your business logic.

## FactoryInterface
In some cases you might need to construct needed class without specifying each of it's dependencies, in this case you can use specific interface `FactoryInterface` which can help you to handle this task:

```php
public function indexAction(FactoryInterface $factory)
{
    dump($factory->make(MyClass::class, [
        'parameter' => 'value'
    ]); 
}
```

Factory will automatically construct needed class by injecting given parameters into it's constructor and filling missing parameters with other depencies (which provides you ability to modify your class constructor by declaring more depencies without breaking application code).

> Factory classes widely used inside spiral to construct different adapters and databases, since factory merged with container in default implementation you can even bind your own factory methods or replace one class with another. Always prefer DI over factory.

## Dependency Injection
As any other framework spiral support dependecy injection in constucted classes, we can demonstrate following principle using this code:

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

Based on provided example we can see that class requests dependendy "Mailer", we can either construct it manually using existed instance of Mailer or dedicate this job to factory or DI.

```php
$userMailer = $factory->make(UserMailer::class);

//Due auto-wiring principles of spiral you can use this alternative
$userMailer = $this->container->get(UserMailer::class);
```

Container will automatically resolse cascade of dependecies and return us valid instance of `UserMailer`. 

> Default spiral container is ment to be fully autowiring (see below) container and all framework code can work without setting up every needed injection, if you really need to use strict wiring - try to replace container by implementing `Spiral\Core\ContainerInterface`.

### Method Injections
Besides using class consturtor to inject dependecies, container allow you to apply same procedute for some specific class methods. Dependency injection on method pre-implemented for you in spiral `Controller` (actions), `Command` (perform) and `Bootloader` (boot) classes:

```php
protected function indexAction(UserMailer $mailer)
{
    //...
}
```

By default spiral classes use `ResolverInterface` to retrieve arguments for a specified method or funtion, as in case with factory you can combine resolved and user parameters together:

```php
$parameters = [
    'name' => 'value'
];

$reflection = new \ReflectionMethod($this, 'perform');
$reflection->setAccessible(true);

$resolver = $this->container->get(ResolverInterface::class);
return $reflection->invokeArgs($this, $resolver->resolveArguments(
    $reflection, 
    $parameters
));
```

### Autowiring
Spiral container and DI trying to be as ivisible for your code as it possible (some applications can be written with zero configuration). By default spiral will resolve only class constructor arguments based on their declared type, scalar arguments can not be resolved without using `FactoryInterface` hovewer container might use argument default value if it presented.

```php
class SampleClass
{
    public function __construct(OtherClass $class, $value = '', AnotherClass $classB = null)
    {
        //Both class and classB will be delivered by container
    }
}
```

> Spiral will try to resolve EVERY constructor/method argument even if it's stated as optional!

### Singletons
In many cases you might want to use only one instance of your class across application, you can either configure container with singleton binding (see below), or simply state your class as singleton by declaring `SINGLETON` constant and implementing `SingletonInterface`:

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

`SINGLETON` constant must be pointing to the binding, id or alias which has to be used in container to store constructed instance, usualy you can use class name by itself as such alias as in given example.

This implementation provides us ability to avoid setting up singleton bindings in application bootstrap which can significantly improve performance (both application and yours :)).

```php
protected function indexAction(MyService $service)
{
    dump($this->container->get(MyService::class) === $service);
}
```

> Most of spiral components has defined SINGLETON constant. You can always disable singleton behaviour by inheriting class and defining SINGLETON constant as `null`.

You can freely extend singleton classes or even replace one implementation with another by creating container binding (see below).

> Attention, declarative singletons are not real singletons as you can drop them from container at any moment (plus it's development sugar), you have to remember that other containers you might use - possibly ignore such constant and force you to declare singleton explicitly. 

### Controllable/Contextual Injections
Spiral Container in addition to regular method injections provides ability to create more intelligent contextual injections. Such technique provide us ability to request multiple databases using following statement:

```php
protected function indexAction(Database $primary, Database $secondary)
{
    dump($primary);
    dump($secondary);
}
```

Both of this examples demonstrates principal of controllable injections which based on dedicating class injection to it's factory using context.

Let's view how our class may look like to define it's own as injectable:

```php
class Name implements InjectableInterface
{
    //Binding name/alias
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

> Also, there going to be as alternative way of creating and using contextual injections by declaring special variable in your factory method or class arguments (come in future releases).

You can always bypass contextual injection using factory with non default set of paramerters:

```php
dump($factory->make(Name::class, ['name' => 'abc']));
```

> You can also use type reflection as well to resolve needed instance in your dependencies like used for injectable configs.

## Bindings (see `Spiral\Core\ContainerInterface`)
As you might notice in a previos sections, there is a lot of mentions for so called bindings. Bindings provide you alibity to link short alias with specific class name or factory, also it can be used to redefine exsited implementation or interface.

The simpliest example of using bindings might look like:

```php
$container->bind(SomeInterface::class, MyClass::class);
```

You can also bind closure functions to be used to resolve needed instance:

```php
$container->bind(SomeInterface::class, function(ContainerInterface $container) {
    return new MyClass('...');
});
```

Or point your container to factory which is going to be resolved on demand:

```php
$container->bind(SomeInterface::class, [MyFactory::class, 'someClass']);
```

Where MyFactory is:

```php
class MyFactory 
{
    public function someClass(... factory dependencies ...)
    {
        return new SomeClass(...);
    }
}
```

We can also bind one class implementation to another (make sure binded class extends target), this can allow you to mock some functionality, test or even change application behaviour:

```php
$container->bind(MyInterface::class, MyClass::class);
//...
$container->bind(MyClass::class, NewClass::class);
```

Based on such binding every requested `MyClass` or `MyInterface` going to be resovled using `NewClass` instance. You wish to create short/virtual bindings described above, use following code:

```php
$container->bind('myClass', NewClass::class);
```

```php
public function indexAction()
{
    dump($this->myClass);
}
```

You can also use alternative method `bingSingleton` which is identical to `bind` by syntax but only resolves dependency once.

```php
$container->bindSingleton('binding', function () {
    return new SomeClass();
});
```

```php
assert($container->get('binding') === $container->make('binding'));
```

> You can also bind one singleton class (with SINGLETON constant) to another which will automatically replace original class.

```php
class MyClass implements SingletonInterface 
{
    const SINGLETON = self::class;
}
```

```php
class OtherClass extends MyClass
{
    
}
```

```php
$container->bing(MyClass::class, OtherClass::class);
```

Let's check our bindings:

```php
public function indexAction(MyClass $class, OtherClass $otherClass)
{
    dump($class instanceof OtherClass);
    dump($class === $otherClass);
}
```

## Additional Container methods
There is few additional methods in `Spiral\Core\ContainerInterface` you might consider using.

### Check if Container has given binding
To check if container has specified binding, let's use `has` method:

```php
dump($this->container->has(MyService::class));
```

### Check if Container has binded instance
To check if container has binded instance (singleton) we can use `hasInstance` method.

```php
dump($this->container->hasInstance(MyService::class));
```

### Remove container binding
If you want to remove existed container binding you can do it using `removeBinding` method, this will automatically destroy any class or singleton associated
with such binding if no other references for class instance exists.

```php
$this->container->removeBinding(MyService::class);
```

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

> Almost every framework components/class created using DI or factory, please **do not construct** framework classes directly in your application, use FactoryInterface/Container/DI instead (to prevent breaking changes in future and make your applicationl more flexible).
