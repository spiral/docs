# Controllers
Spiral controller is an separate application part which can have multiple entry points (actions). 
Controllers can be called under any environment and return any type of response. Most commonly controllers
are used as target for http Router.

## Definition
The definition of a controller includes a simple class creation in any accessible location (PSR-4 compatible):
```php
<?php
namespace Controllers;
use Spiral\Core\Controller;

class HomeController extends Controller
{
    //Controller action  
}
```
The default location for new controllers is application/classes/Controllers under namespace `Controllers`. 
Every controller should extend `Spiral\Core\Controller` or any of it's children.

##Actions
Any real controller should contain at least one action and/or default controller action (recommended) (by default 
`index`). The definition of an action is the same as defining public class method:
```php
<?php
namespace Controllers;
use Spiral\Core\Controller;

class HomeController extends Controller
{
    public function index() 
    {
        return "This is controller response.";
    }
}
```

##Executing controller actions
Even if the controllers is executed as result of routing, you can call any controller action by 
using the `Core` method `callAction`. This method will accept the controller class name, action name (method)
 and the set of parameters will be passed to the controller method. Parameters will automatically be matched
  with appropriate argument name in method definition.
```php
public function action($name, $value = 'default') 
{
    dump($name);
    dump($value);
    return "This is controller response.";
}
```
To execute such action we will call `Core` directly:
```php
Core::getInstance()->callAction('Controllers\HomeController', 'action', ['name' => 'John']);
```
Or inside controller (HMVC):
```php
public function index() 
{
   return $this->core->callAction('Controllers\HomeController', 'action', ['name' => 'John']);
}
```
Executing this controller action without all the required parameters will return `ClientException`. Most
 of the time, the controllers are called via http `Router`. Parameters will be populated using route optional 
 segments.
```php
public function bootstrap()
{
    $this->http->route('show/<id>', 'Controllers\HomeController::index');
}
```
Action definition:
```php
public function index($id) 
{
    dump($id);
}
```

##Dependency Injection
Controllers fully support Spiral IoC `Container` in contructors:
```php
<?php
namespace Controllers;
use Spiral\Core\Controller;
use Spiral\Components\DBAL\DatabaseManager;

class HomeController extends Controller
{
    protected $dbal;

    public function __construct(DatabaseManager $dbal)
    {
        $this->dbal = $dbal;
    }
}
```
Or controller actions (Method injection):
```php
<?php
namespace Controllers;
use Spiral\Core\Controller;
use Spiral\Components\DBAL\DatabaseManager;

class HomeController extends Controller
{
    public function index(DatabaseManager $dbal)
    {
    }
}
```
> **Attention:** Action parameters can be combined with IoC. However, controller prefers 
parameter over injection.

```php
public function index($id, Database $defult)
{

}
```
##Controllers and HTTP
For more information on routing to controllers and generating responses, read HTTP section.
