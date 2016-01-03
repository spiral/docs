# Controllers
Spiral framework provides simple separation layer between http dispatcher and controller classes - `CoreInterface` which by default binded to your App class.

> To read how to route http request to a specified container [go here](/http/routing.md).

# Controller classes

To better understand what controller is and how it's getting invoked by application let's take a look at two foundation interfaces first:

```php
namespace Spiral\Core\HMVC;
use Spiral\Core\Exceptions\ControllerException;

/**
 * General application enterpoint class.
 */
interface CoreInterface
{
    /**
     * Request specific action result from Core. Due in 99% every action will need parent
     * controller, we can request it too.
     *
     * @param string $controller Controller class.
     * @param string $action     Controller action, empty by default (controller will use default
     *                           action).
     * @param array  $parameters Action parameters (if any).
     * @return mixed
     * @throws ControllerException
     * @throws \Exception
     */
    public function callAction($controller, $action = '', array $parameters = []);
}
```

```php
namespace Spiral\Core\HMVC;
use Spiral\Core\Exceptions\ControllerException;

/**
 * Class being treated as controller.
 */
interface ControllerInterface
{
    /**
     * Execute specific controller action (method).
     *
     * @param string $action     Action name, without postfixes and prefixes.
     * @param array  $parameters Method parameters.
     * @return mixed
     * @throws ControllerException
     * @throws \Exception
     */
    public function callAction($action = '', array $parameters = []);
}
```

As you might notice there is two similar interfaces Core and Controller, Core interface is usually access point to your application (if you using proposed HMVC or MVC organization), this endpoint can be called from almost any part of your application and by default invoked by Http routes.

Once callAction method of `CoreInterface` invoked implementation must route request to specific controller `callAction` method. On a practice it means that can call any controller in your application or module using following code:

```php
protected function indexAction(CoreInterface $core)
{
    return $core->callAction(SomeController::class, 'action', [...]);
}
```

Since, by default, CoreInterface implemented by your application you can also do that:

```php
protected function indexAction()
{
    return $this->app->callAction(SomeController::class, 'action', [...]);
}
```

When no controller under given name can be found, or action is invalid `ControllerException` will be throwed out.s.

## Default Controller Implementation
Even if you can easily implement your contoroller using interface only (controllers resolved using `FactoryInterface`, so you can use contructor injections) it might be easier to use pre-created controller implementation `Spiral\Core\Controller`. This implementation provides support for method injections and simplified access to container bindings (every Controller is Service):

```php
protected function indexAction(StoreInterface $store)
{
    dump($store);
    return $this->app->callAction(SomeController::class, 'action', [...]);
}
```

As you might notice every controller action has specific postfix "Action", such string is required for default implementation so class
can decide if requested action are allowed to be executed.

Simpliest implementation of controller might look like:

```php
class HomeController extends Controller
{
    protected function indexAction()
    {
        return 'hello world';
    }
}
```

> When no action set controller will use it's default action, which is "index" (you can overwrite this value as values used for actions prefixes and postfixes, see Controller code).

## "HMVC"
Due every controller can be called without nesessary routing you can implement HMVC approach in your application like that:

```php
protected function hvmcAction(OtherController $controller)
{
    $other = $controller->callAction('action', [...]);
    
    //Or using core
    $this->app->callAction(OtherController::class, 'action', [...]);
}
```

> Spiral does not have specific implemetation of `View` but rather provides simplifier access to rendering engines, you can create missing models if you need them.

## Other Approaches
Spiral framework does not force you to any specific implementation of your application architecture, you can easily remove every controller and replace it with ADR, MMVM and other patters since the only thing which links http layers to controllers is routes and `CoreInterface`:

```php
//Calling different actions of HomeController
$route = new Route('home', '<action>.html', 'Controllers\HomeController::<action>');

//Calling closures
$route = new Route('test', 'test', function(){
    return 'hello world'
});
```

> You can read more about routing [here](/http/routing.md).

Since route target will be automatically resolved using factory and DI you can replace your controller layer with, for example, Actions:

```php
class Action extends Service
{
    //Since this class is invokable you 
    public function __invoke()
    {
        dump($this->request);
        return 'hello world';
    }
}
```

```php
$http->addRoute(new Route('test', 'test', \Actions\Action::class));
```

> You can implement method injections by creating additional method inside your action, for example perform, by default route will send you ServerRequestInterface and ResponseInterface into your __invoke method.
