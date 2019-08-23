# Controllers
Spiral framework provides simple separation layer between http routes and controller classes - `CoreInterface` which by default binded to your App class.

> To read how to route http request to a specified controller [go here](/old/httphttp/routing.md).

# Controller classes
To better understand what controller is and how it's getting invoked by application let's take a look at two foundation interfaces first:

```php
namespace Spiral\Core\HMVC;

use Spiral\Core\Exceptions\ControllerException;

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
     *
     * @param array  $scope      Scope in a form if [alias=>binding] to be set by container before
     *                           executing given action.
     *
     * @return mixed
     *
     * @throws ControllerException
     *
     * @throws \Exception
     */
    public function callAction(
        string $controller,
        string $action = null,
        array $parameters = [],
        array $scope = []
    );
}
```

```php
namespace Spiral\Core\HMVC;
use Spiral\Core\Exceptions\ControllerException;

interface ControllerInterface
{
    /**
     * Execute specific controller action (method).
     *
     * @param string $action     Action name, without postfixes and prefixes.
     * @param array  $parameters Method parameters.
     *
     * @return mixed
     *
     * @throws ControllerException
     *
     * @throws \Throwable
     */
    public function callAction(string $action = null, array $parameters = []);
}
```

By default spiral expect you to call your controller actions using `CoreInterface` as proxy:

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

When no controller under given name can be found, or action is invalid `ControllerException` will be thrown out.

## Default Controller Implementation
Implement `Spiral\Core\Controller` by your class to automatically enable shortcuts and methods injections in your action methods:

```php
protected function indexAction(StoreInterface $store)
{
    dump($store);
    
    return $this->app->callAction(SomeController::class, 'action', [...]);
}
```

As you might notice every controller action has specific postfix "Action", such string is required by default implementation so class can decide if requested action are allowed to be executed.

> `indexAction` is treated as default action for controller.

## HMVC Cores
You are free to define your own implementation of `CoreInterface` in order to implement custom functionality for accessing your controllers and processing controller response.
HttpDispatcher routes provide you ability to associate custom core with any of your routes:

```php
class SecuredCore extends Component implements CoreInterface
{
    use GuardedTrait;

    /**
     * @var SecureConfig
     */
    private $config = null;

    /**
     * @var CoreInterface
     */
    protected $app = null;

    /**
     * @param SecureConfig  $config
     * @param CoreInterface $app User application.
     */
    public function __construct(SecureConfig $config, CoreInterface $app)
    {
        $this->config = $config;
        $this->app = $app;
    }

    /**
     * @return SecureConfig
     */
    public function getConfig(): SecureConfig
    {
        return $this->config;
    }

    /**
     * {@inheritdoc}
     */
    public function callAction(
        string $controller,
        string $action = null,
        array $parameters = [],
        array $scope = []
    ) {
        if (!$this->config->hasController($controller)) {
            throw new ControllerException(
                "Undefined controller '{$controller}'",
                ControllerException::NOT_FOUND
            );
        }

        $actionPermission = "{$this->config->guardNamespace()}.{$controller}";

        if (!$this->getGuard()->allows($actionPermission, compact('action'))) {
            throw new ControllerException(
                "Unreachable controller '{$controller}'",
                ControllerException::FORBIDDEN
            );
        }

        //Delegate controller call to real application
        return $this->app->callAction(
            $this->config->controllerClass($controller),
            $action,
            $parameters,
            $scope + [Vault::class => $this]
        );
    }
}
```

Given example will force user authorization for a every controller action. To associate such core with our route:

```php
$route = new ControllersRoute(
    'default',                                  //Route name
    'secured/[<controller>[/<action>[/<id>]]]', //Pattern [] braces define optional segment
    'Controllers\Secured'                       //Namespace
);

$http->addRoute($route->withCore(SecuredCore::class));
```

> There is no real limitation on how deep you can go.

## Other Approaches
Be free to implement your own route/action architecture by implementing custom `CoreInterface` or `ControllerInterface` or skip them completely (ARD):

```php
class MyAction extends Service
{
    //By default routes treat thier endpoints as PSR-7 compatible;
    //controllers are an exception from this rule
    public function __invoke($request, $response)
    {
        dump($this->request);
        return 'hello world';
    }
}
```

```php
$http->addRoute(new Route('test', 'test', \Actions\MyAction::class));
```

> You can also make your Actions more friendly by utilizing shortcuts and method injections:

```php
abstract class Action extends Service
{
    //Child class must define perform method with allowed DI
    public function __invoke($request, $response)
    {
        $reflection = new \ReflectionMethod($this, 'perform');
        $reflection->setAccessible(true);
        
        //Can be received as constructor injection
        $resolver = $this->container->get(ResolverInterface::class);
        
        return $reflection->invokeArgs($this, $resolver->resolveArguments(
            $reflection,
            compact('request', 'response')
        ));
    }
}
```

```php
class MyAction extends Acion
{
    //By default routes treat thier endpoints as PSR-7 compatible;
    //controllers are an exception from this rule
    public function perform($request)
    {
        return $this->view->render('welcome');
    }
}
```