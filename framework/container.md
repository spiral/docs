# Container, Factory, DI
Spiral framework includes set of implementations to enable inversion of control and dependency injection in your application. In many cases you can use same functionality in a form of Service Locators.

> Note that IoC implementation is based on [Interop/Container](https://github.com/container-interop/container-interop), you are free to replace default spiral container or delegate it's functionality to 3rd part component.


## Bindings
Container binding(s) used to link aliases (i.e. shortcuts) and interfaces to conrete implementations or factory methods. Basic example (code can be located in controller or application bootloader):

```php
$container->bind(SomeInterface::class, MyClass::class);
```

Bind to Closure to construct class manually:

```php
$container->bind(SomeInterface::class, function(ContainerInterface $container) {
    return new MyClass('...');
});
```

More optimally you can point binding to a specific factory method:

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

> Note that such factory will be constructed inside container, it makes you able to use constructor injections and define such class as singleton.

Many of framework or application functionality can be altered by replacing default interfaces/classes with custom code:

```php
$container->bind(MyInterface::class, MyClass::class);
//...
$container->bind(MyClass::class, NewClass::class);
```

Now, every requested `MyClass` or `MyInterface` will be resolved by `NewClass` instance. 

Use simple stings as binding name (alias) to create shortcuts to your classes and services:

```php
$container->bind('myClass', NewClass::class);
```

```php
public function indexAction()
{
    dump($this->myClass);
}
```

> Read more about shortcuts [here](shortcuts.md).

## Singletons
If you want your class to be constructed only once in a term of application life cycle, use alternative method `bingSingleton`:

```php
$container->bindSingleton('binding', function () {
    return new SomeClass();
});
```

```php
assert($container->get('binding') === $container->make('binding'));
```

> You can re-bind any of created singletons at any moment or destroy them.

You can freely extend singletons:

```php
class MyClass implements SingletonInterface 
{
}
```

```php
class OtherClass extends MyClass
{
    
}
```

```php
$container->bind(MyClass::class, OtherClass::class);
```

Let's check our bindings:

```php
public function indexAction(MyClass $class, OtherClass $otherClass)
{
    dump($class instanceof OtherClass);
    dump($class === $otherClass);
}
```

## Service Locator
The basic example of container can be based around Service Locator invocations:

```php
public function indexAction()
{
    echo $this->container->get('views')->render(...);
}
```

Such code will receive instance associated with alias "views" and call method "render".

> See below how to create associations.

Due to support of method and constructor injections you can use alternative approach:

```php
public function indexAction(ViewsInterface $views)
{
    echo $views->render(...);
}
```

> Make sure to know when to switch between Service Locator and Dependency Injections and to understand [the difference between them](https://www.google.com/search?q=service+locator+vs+dependency+injection). 

You can read about how to make container bindings and bootload your application [here](/framework/bootloaders.md).

## FactoryInterface
In many cases you might want to create an instance with automatically resolved dependencies, use `FactoryInterface` for such purposes:

```php
public function indexAction(FactoryInterface $factory)
{
    dump($factory->make(MyClass::class, [
        'parameter' => 'value'
    ])); 
}
```

## Dependency Injection
Spiral support both method and constructor injections for your classes:

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

We can automatically resolve "Mailer" value by creating such class using factory, or simply requesting it as dependency:

```php
$userMailer = $factory->make(UserMailer::class);

//Due auto-wiring principles of spiral you can use this alternative
$userMailer = $this->container->get(UserMailer::class);
```

Container will automatically resolve cascade of dependencies and return us valid instance of `UserMailer`. 

### Auto-wiring
Default spiral container is intended to be as invisible for your code as possible, in order to achieve that container is set to be auto-wiring by default. You have to acknowledge that framework will try to resolve EVERY constructor/method argument even if it's stated as optional, unless you have manually defined binding or factory method:

> Replace default container by implementing `Spiral\Core\ContainerInterface` or delegate lookup functions using parent container.

```php
class SampleClass
{
    public function __construct(OtherClass $class, string $value = '', AnotherClass $classB = null)
    {
        //Both class and classB will be delivered by container
    }
}
```

## Lazy Singletons
You can skip singleton bindings by implementing `SingletonInterface` in your class:

```php
class MyService implements SingletonInterface
{
    public function method()
    {
        //...
    }
}
```

Now, container will automatically treat this class as singleton in your application:

```php
protected function indexAction(MyService $service)
{
    dump($this->container->get(MyService::class) === $service);
}
```

> Most of spiral components has defined as singleton. You can always disable singleton behaviour by creating your custom factory method in bootloader.

## Controllable/Contextual Injections
Spiral Container in addition to regular method injections provides ability to create more intelligent contextual injections. Such technique provide us ability to request multiple databases using following statement:

```php
protected function indexAction(Database $primary, Database $secondary)
{
    dump($primary);
    dump($secondary);
}
```

Implement `InjectableInterface` to declare such behaviour:

```php
class Name implements InjectableInterface
{
    //Binding name/alias
    const INJECTOR = Injector::class;
    
    //This argument can not be automatically set by Container if we trying to resolve
    //this class in our arguments
    public function __constrcut(string $name)
    {
        //...
    }
}
```

Now we have to define class responsible for such injections:

```php
class Injector implements InjectorInterface
{
    public function createInjection(\ReflectionClass $class, string $context = null)
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

You can always bypass contextual injection using factories:

```php
dump($factory->make(Name::class, ['name' => 'abc']));
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

## Partial Bindings
You can also create bindings with partially pre-defined class parameters:

```php
$container = new Container();
$container->bind('abc', new Autowire(ClassWithName::class, ['name' => 'Fixed']));

$abc = $container->get('abc');

$this->assertSame('Fixed', $abc->getName());
```

## Container Delegation
You are able to delegate container functionality to external PSR compatible implementation using container constructor.

```php             
$container = new \Pimple\ContainerContainer();

//Initiating shared container, bindings, directories and etc
$application = App::init([
    'root'        => $root,
    'runtime'     => $root . 'runtime/',
    'libraries'   => $root . 'vendor/',
    'application' => $root . 'app/',
    //other directories calculated based on default pattern, @see Core::__constructor()
], null, $container);
```

> Note that container like that will not be available in your application directly thought `ContainerInterface` but rather composited inside of `Spiral\Core\ContainerInterface`. Spiral container
will work as auto-wiring layer at of your Pimple container.