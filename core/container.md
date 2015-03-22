# Container and Dependency Injection
Spiral Container is base class for `Core` and `Application` instances. Contantainer provides support for contructor and
method injections, ability to redefine core components, create singletons and etc. Additionally Contantainer supports
"controllable injection" technique, allows to resolve injection via custom code.

## Usage examples
Class dependencies will be resolved only if instance created via `make()` method, requested directly via container or 
declared as dependecly in another class (inner dependency). Let's check some simple examples:
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
Additionally, you can request such class as dependendcy in other classes.
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
You can redefine any existed biniging or class by providing your own implementation:
```php
namespace Models;

class NewService extends MyService
{
}
```
```php
$this->core->bind('Models\MyService', 'Models\NewService');
```
In this case every class injection will be resolved as `Models\NewService`. Following can also be used to bind 
implementations to interfaces:
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
    return new NewService(FileManager::getInstace(), 'someting else');
});
```
In some cases you will need to construct instance only once (singleton), you can use another method:
```php
$this->core->bindSingleton('Models\MyService', function() {
    //This code will be executed only once on demand
    return new NewService(FileManager::getInstace(), 'someting else');
});
```
### Binding Instances
You can also bind already constructed instance, in this case every injection will be resolved with that instance.
```php
$this->core->bind('Models\MyService', new NewService(FileManager::getInstace(), 'someting else'));
```


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

`hasBinding`    | Check if desired alias or class name binded in Container.    
`removeBinding` | Removed existed binding (will destroy associated singleton). 
`getBindings`   | Get all bindings.                                            


