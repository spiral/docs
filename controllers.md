# Controllers
Spiral controller is independend application part which can have multiple enterpoints (actions). 
Controllers can be called under any evrionment and return any type of response. Most commonly controllers
 used are target for Http Router.

## Definition
Definition of controller includes simple class creation in any reachable location (PSR-4 compatible):
```php
<?php
namespace Controllers;
use Spiral\Core\Controller;

class HomeController extends Controller
{
    //Controller action  
}
```
Default location for new controllers is application/classes/Controllers under namespace `Controllers`. 

##Actions
Any real controller should contain at least one action, and (recommendedly) default action (by default 
`index`). Definition of action is the same as defining public class method:
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
Even if controllers usually executed as result of routing, you can call any controller action by 
using `Core` method `callAction`. This method will accept controller class name, action name (method)
 and set of parameters will be passed to controller method. Parameters will be automatically matched
  with appropriate argument name in method definition.
```php
public function action($name, $value = 'default') 
{
    dump($name);
    dumP($value);
    return "This is controller response.";
}
```
To execute such action we are going to call `Core` directly:
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
Executing such controller action without all required parameters will cause `ClientException`. As most
 of the time controllers called via http `Router` parameters will be populated using route optional 
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
> **Attention:** Action parameters can be combined with IoC, however controller will always prefer 
parameter over injection.

```php
<?php
public function index($id, Database $defult)
{

}
```