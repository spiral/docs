# Container and Dependency Injection
Spiral Container is base class for `Core` and `Application` instances, it provides support for constructor and
method injections, ability to redefine core components, create singletons and etc. Additionally Container supports
"controllable injection" technique, allows to resolve injection via custom code.

## Usage examples
Class dependencies will be resolved only if instance created via `make()` method, requested directly via container or 
declared as dependency in another class (inner dependency). Let's check some simple examples:
```php
namespace Models;

use Spiral\Components\Files\FileManager;
use Spiral\Core\Component;

class MyService extends Component
{
    protected $file = null;

    public function __construct(FileManager $file)
    {
        $this->file = $file;
    }
}
```
Class can be constructed multiple ways:
```php
$service = new MyService($file); //Manual
$service = MyService::make();
$service = Container::get('Models\MyService');
```
Additionally, you can request such class as dependency in other classes.
```php
namespace Controllers;
use Models\MyService;
use Spiral\Core\Controller;

class HomeController extends Controller
{
    public function __construct(MyService $service)
    {
        dump($service);
    }
}
```
In case of controllers, dependency can be declared in action method:
```php
namespace Controllers;
use Models\MyService;
use Spiral\Core\Controller;

class HomeController extends Controller
{
    public function index(MyService $service)
    {
        dump($service);
        
        //Alternative
        $this->core->get('Models\MyService');
    }
}
```

## Custom Binding
Sometimes you need more functionality that simple class injections. In this case you can use `Container` bindings to achieve your goals.

### Registering class Alias
You can redefine any existed binding or class by providing your own implementation:
```php
namespace Models;

class NewService extends MyService
{
}
```
```php
$this->core->bind('Models\MyService', 'Models\NewService');
```
In this case every injection will be resolved as `Models\NewService`. Following can also be used to bind 
implementations to interfaces:
> You can use this technique to extend some of spiral core components.

```php
$this->core->bing('ServiceInterface', 'Models\MyService');
```
```php
class MyController extends Controller 
{
    public function __construct(ServiceInterface $service)
    {
        dump($service);
    }
}
```

### Resolvers
You can also use custom closure function to resolve class instance:
```php
$this->core->bind('Models\MyService', function() {
    return new NewService(FileManager::getInstance(), 'something else');
});
```
In some cases you will need to construct instance only once (singleton), you can use another method:
```php
$this->core->bindSingleton('Models\MyService', function() {
    //This code will be executed only once on demand
    return new NewService(FileManager::getInstance(), 'something else');
});
```

### Binding Instances
You can also bind already constructed instance, in this case every injection will be resolved with that instance.
```php
$this->core->bind('Models\MyService', new NewService(FileManager::getInstance(), 'something else'));
```

## Singletons
Spiral has multiple ways to delcare Singleton classes, two of them (binding instance or singleton resolved) was covered in
previous sections. Another way to declare that class is singleton - let class tell about it via SINGLETON constant
and SingletonTrait (can be applied only to `Component` classes).
```php
use Spiral\Core\Component\SingletonTrait;

class Test extends Component
{
    use SingletonTrait;
    
    const SINGLETON = __CLASS__;
}
```
Now singleton can be received using 4 different methods:
```php
class MyController extends Controller
{
    public function action(Test $testA)
    {
        $testB = Test::make();
        $testC = Test::getInstance();
        $testD = Container::get('Test');
        
        assert($testA === $testB);
        assert($testB === $testC);
        assert($testC === $testD);
        assert($testA instanceof Test);
    }
}
```
You can always destruct singleton by using `Container::removeBinding()` method.
```php
Container::removeBinding('Test'); //No more Test instance
```
> You can use custom `SINGLETON` constant value, however in this case additional core binding are required.

Singletons follow container bindings:
```php
Container::bind("Test", "Test2");
```
```php
class Test2 extends Test
{
}
```
```php
class MyController extends Controller
{
    public function action(Test $testA, Test2 $testE)
    {
        $testB = Test::make();
        $testC = Test::getInstance();
        $testD = Container::get('Test');
        $testF = Container::get('Test2');
    
        assert($testA === $testB);
        assert($testB === $testC);
        assert($testC === $testD);
        assert($testD === $testE);
        assert($testE === $testF);
    
        assert($testA instanceof Test2);
    }
}
```

## "Controllable injections"
Sometimes you may need to receive instance created by parent factory based on some alias or type. Spiral provides convenient
way to pass injection resolution from `Container` to parent factory/manager.

To declare that class should be resolved using external factory/manager, simply define class constant with factory name
```php
class MyClass
{
    class INJECTION_MANAGER = "MyService"
    
    public function __construct($name)
    {
        dump($name);
    }
}
```
Factory/manager should implement `Spiral\Core\Container\InjectionManagerInterface`:
```php
class MyService implements Spiral\Core\Container\InjectionManagerInterface
{
    public static function resolveInjection(\ReflectionClass $class, \ReflectionParameter $parameter)
    {
        dump($class->getName()); //MyClass or it's child
        return new MyClass($parameter->getName());
    }
}
```
Usage (controller method):
```php
public function action(MyClass $myName)
{
    //Will dump "MyClass" and "myName"
}
```

Following techniques was implemented for all spiral databases (using parameter name) and for cache stores 
(using parameter type). Following examples provided for controllers method injections:
```php
public function action(Database $db, RedisClient $redis, MongoDatabase $mongo, MemcacheStore $store)
{
}
```
Provided code is identical to:
```php
public function action()
{
    $db = $this->dbal->db('default');
    $redis = $this->redis->client('default');
    $mongo = $this->odm->db('default');
    $store = $this->cache->store('memcache');
}
```
In provided example `Container` will pass parameter reflection to DBAL, ODM, Redis and Cache components, which will
allow them to construct or fetch required instance based on parameter name or type.
> DBAL, ODM and Redis components has special config section `aliases` which can be used to bind multiple names to one database (for example `db` = `database` = `default`).


## Adding method injection to custom classes
By default method injection can be used only in Controllers and console Commands. To add method injection to your custom
class method, use `Spiral\Core\Container\MethodTrait`.
```php
use Spiral\Core\Container\MethodTrait;

class MyClass 
{
    use MethodTrait;
    
    protected function doAction(array $parameters = array())
    {
        return $this->callMethod('action', $parameters);
    }
}
```
##Additional Container methods
You can use additional methods to check binding state or receive all bindings.

Method          | Description
---             | ---
`hasBinding`    | Check if desired alias or class name binded in Container.        
`removeBinding` | Remove existed binding (will destroy associated singleton). 
`getBindings`   | Get all bindings.                         